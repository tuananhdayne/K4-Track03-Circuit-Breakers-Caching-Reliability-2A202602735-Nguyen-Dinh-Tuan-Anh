# Lab 25B — Report

Họ tên / nhóm: Nguyễn Đình Tuấn Anh (MSV: 2A202602735)  Commit: a3f9ce7078c14d183ed1e95d311a2df59bc86855  Điểm tự động (make grade): 84.0 / 88

> Mọi nhận định phải có số. Dán số từ reports/gameday/*.json, reports/workflows.json, reports/slo.json.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa vào G6 và G4. Khi node chính sập, timeout 8s giúp client dứt điểm sớm để failover trước deadline 30s của user. Nếu để 30s như thí nghiệm 2 thì availability ở G6 tụt xuống 0.9488 do 109 request bị quá hạn deadline. |
| max_attempts_per_provider | 2 | Dựa vào G2 và G3. Chỉ thử lại đúng 1 lần (tổng 2 lần) trên cùng một node khi gặp lỗi tạm thời (500, 529). Nếu không được thì đổi node ngay để tránh dồn tải làm nghẽn thêm node đang yếu. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Dựa vào G1 và G2. Dùng full jitter với base 0.25s và trần 4.0s để rải đều các lần retry, tránh việc nhiều client cùng gọi lại đúng một thời điểm gây thundering herd khi node vừa hồi phục. |
| retry_budget_ratio | 0.1 | Dựa vào G1 và G2. Giới hạn số lượt retry tối đa bằng 10% số request gửi đi, chặn đứng nguy cơ bùng nổ retry storm ngay cả khi breaker không hoạt động. |
| breaker.window_size / min_calls | 20 / 10 | Đo trên 20 kết quả gần nhất và chỉ xét ngắt mạch khi có từ 10 kết quả trở lên để số liệu đủ tin cậy, tránh bị ngắt nhầm vào lúc tải thấp. |
| breaker.failure_rate_threshold | 0.5 | Dựa vào G3. Khi tỷ lệ lỗi vượt quá 50% trong cửa sổ trượt, breaker mở ra ngay để chuyển sang node khác. |
| breaker.slow_call_s | 5.0 | Dựa vào G4. Bình thường alpha chạy tầm 1.2s, nếu cuộc gọi kéo dài từ 5s trở lên là dấu hiệu node đang nghẽn nặng, cần xử lý sớm trước khi dính timeout. |
| breaker.slow_rate_threshold | 0.6 | Dựa vào G4. Khi có từ 60% cuộc gọi bị chậm, breaker ngắt mạch giúp kéo p95 ở G4 từ 13.764s xuống còn 5.562s. |
| breaker.open_duration_s | 10.0 | Dựa vào G2 và G6. Mở mạch trong 10 giây để node có đủ thời gian dọn sạch hàng đợi tồn đọng. |
| breaker.half_open_max_probes | 2 | Dựa vào G2. Khi ở trạng thái half-open chỉ cho đúng 2 request vào thăm dò để kiểm tra sức khỏe, không thả ồ ạt request vào node vừa gượng dậy. |

Sơ đồ quyết định của on_error:

```
                    +------------------------------------+
                    |       Nhận ProviderError           |
                    +------------------------------------+
                                      |
                           Phân loại classify(err)
                                      |
         +-----------------+----------+----------+-----------------+
         |                 |                     |                 |
   CLIENT_ERROR     QUOTA_EXHAUSTED         RATE_LIMITED     OVERLOADED /
   (400, 401, 422)  (429 no quota)        (429 + retry-after) SERVER_ERROR (5xx) /
         |                 |                     |           TIMEOUT (0, 408)
         v                 v                     v                 |
  release breaker   release breaker       release breaker          v
  GiveUp            cooldown provider     cooldown provider  on_failure breaker
  ("client_error")  300s                  (now + retry_after)      |
  (Không retry,     Failover ngay         Failover ngay      tries < max_attempts (2)
   không failover)                                           & budget cho phép
                                                             & kịp deadline?
                                                                   |
                                                      +------------+------------+
                                                      | Có                      | Không
                                                      v                         v
                                                Retry node cũ             Failover sang
                                                với Full Jitter           node tiếp theo
                                                (theo retry-after)        trong ladder
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. Retry budget là lớp phòng thủ cuối:
Khi giả lập breaker bị hỏng (min_calls đặt 1 tỉ) và cho thử lại tối đa 5 lần: nếu còn bật retry budget, G2 giữ được amplification 1.383, retry_ratio 0.058, recovery 0s và availability 0.9987; G3 có amplification 1.136, retry_ratio 0.068. Ngược lại, khi tắt budget thì G2 bị dồn tải với amplification vọt lên 1.672, retry_ratio tăng gấp 8 lần lên 0.491, mất tới 30s mới hồi phục và availability giảm còn 0.9649; G3 retry_ratio cũng tăng lên 0.162. Kể cả khi breaker chạy tốt thì budget vẫn rất cần thiết vì breaker chỉ ngắt cục bộ theo từng node, còn budget quản lý tổng lượng retry toàn hệ thống, giúp chặn đứng rủi ro bão retry khi nhiều node cùng gặp sự cố.

2. attempt_timeout_s = 30:
Khi chạy G6 với timeout 30s thay vì 8s, availability tụt từ 1.0000 xuống 0.9488, xuất hiện 109 lượt bỏ cuộc do deadline và 13 lượt không có node khả dụng. Tuy nhiên p95 lại giảm từ 5.961s xuống 5.306s do hiệu ứng survivorship bias: các request gặp node chết bị ngâm quá lâu nên đứt gánh giữa đường và bị loại khỏi thống kê, còn p95 chỉ tính trên những request may mắn đi thẳng vào node sống. Ở G4 thì availability vẫn giữ 1.0000 và p95 gần như giữ nguyên (5.562s so với 5.683s) vì alpha chỉ chậm 5s chứ không chết hẳn, và cơ chế phát hiện slow call của breaker đã ngắt mạch sớm để chuyển node trước khi chạm trần timeout. Vì vậy timeout không nên đặt theo deadline 30s của user mà nên dựa trên p99 lúc chạy bình thường cộng thêm một khoảng đệm an toàn (khoảng 6-8s) để nếu một node treo thì vẫn còn đủ thời gian failover.

3. Bỏ lọc tool calling khỏi ladder():
Khi bỏ lọc tool calling ở G6, availability của hệ thống rơi từ 1.0000 xuống 0.8977 và có đúng 244 lượt định tuyến không tương thích (incompatible_routes). Đứng ở góc độ làm sản phẩm, điều này có nghĩa là có 244 khách hàng gửi yêu cầu hoàn tiền nhưng lại bị đẩy sang model gamma-mini không hỗ trợ gọi tool ngân hàng. Kết quả là giao diện báo lỗi hoặc treo giao dịch, khách hàng không nhận được tiền hoàn, khiến tỷ lệ khiếu nại tăng vọt và làm giảm độ tin cậy của dịch vụ.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1496700 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Trả lời:
- Vì sao chỉ idempotency key là chưa đủ? Idempotency key chỉ giúp bên cổng thanh toán không trừ tiền 2 lần cho cùng một lệnh, nhưng nếu worker crash mà không có checkpoint thì khi chạy lại, worker sẽ phải gọi lại từ đầu toàn bộ các bước LLM phía trước (như classify_request), vừa tốn chi phí gọi model vô ích vừa dễ làm chậm toàn bộ phiên xử lý.
- Vì sao chỉ checkpoint cũng chưa đủ? Nếu mạng bị rớt sau khi cổng thanh toán đã trừ tiền thành công nhưng worker chưa kịp nhận kết quả về, worker sẽ thử gọi lại bước thanh toán. Nếu không có idempotency key cố định đi kèm, cổng thanh toán sẽ hiểu đây là giao dịch mới và tiếp tục trừ tiền lần nữa.
- Điều gì xảy ra nếu ghi checkpoint trước khi gọi payment? Nếu worker ghi checkpoint xong mà bị crash ngay trước lúc lệnh thanh toán thực sự được gửi đi, worker mới khi resume sẽ thấy checkpoint đã có và tưởng rằng tiền đã hoàn xong nên bỏ qua bước thanh toán. Khi đó khách hàng sẽ bị mất tiền hoàn và hệ thống ghi nhận sai lệch.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | paged (phút 3481) | paged (fast_burn_2pct_1h) | 27 min |
| provider_outage | page | paged (phút 8760) | paged (fast_burn_2pct_1h) | 4 min |
| deploy_blip | none | paged (phút 15240) | none | Không bị báo giả |
| slow_leak | ticket | not detected | ticket (budget_leak_10pct_3d) | 1337 min |
| degraded_evening | page | paged (phút 30214) | paged (slow_burn_5pct_6h) | 291 min |
| night_spike | none | paged (phút 34740) | none | Không bị báo giả |
| small_bump | none | paged (phút 39542) | none | Không bị báo giả |

Error budget 30 ngày: tổng 25,901,755 requests, số lỗi cho phép là 129,508.8, thực tế lỗi 55,618. Độ khả dụng đạt 99.7853% so với mục tiêu 99.5%. Ngân sách lỗi đã tiêu 42.95%, còn dư 57.05%. Vì ngân sách còn hơn một nửa nên không cần đóng băng release. Sự cố small_bump (20 phút với 12% lỗi) không nên page vì nó chỉ diễn ra trong thời gian ngắn rồi tự hết, lượng lỗi tích lũy rất nhỏ và không làm thâm hụt đáng kể ngân sách tháng, nếu page sẽ làm phiền kỹ sư trực một cách không cần thiết.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](/home/tuananh/vinuni/1310/K4-Track03-Circuit-Breakers-Caching-Reliability-2A202602735-Nguyen-Dinh-Tuan-Anh/reports/gameday/G2_overload_storm.png)

Tóm tắt (3 câu):
Tại thời điểm t=120s, node alpha bị tụt 75% năng lực GPU trong 3 phút. Client bản v1 do retry 3 tầng dồn dập, không có jitter và không có circuit breaker nên đã tự khuếch đại lưu lượng lên gấp 9.9 lần làm nghẽn hoàn toàn hàng đợi. Đến khi alpha hồi phục ở t=300s, hệ thống vẫn tiếp tục tê liệt thêm 155s nữa do rơi vào trạng thái metastable failure.

Ảnh hưởng:
Có 1,285 lượt người dùng bị lỗi (availability chỉ còn 45.60%), gián đoạn kéo dài 335 giây từ t=120s đến t=455s. Tổng chi phí bị đẩy lên 4,983 đơn vị và làm mất hơn 54% error budget của cả tháng.

Timeline (giây ảo — trích từ naive_G2_overload_storm_timeseries.csv):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha mất 75% GPU, hàng đợi bắt đầu ứ đọng | attempts_alpha = 5 req/s, calls_ok = 7 req/s |
| 136 | Bắt đầu bùng nổ retry khi các request đầu tiên quá hạn | attempts_alpha tăng vọt lên 53 req/s, calls_ok tụt về 3 |
| 143 | Cơn bão retry đạt đỉnh, hàng đợi server bị quá tải hoàn toàn | attempts_alpha đạt đỉnh 254 req/s, calls_ok sập về 0 |
| 300 | alpha phục hồi 100% năng lực phần cứng | attempts_alpha vẫn ở mức 115 req/s, calls_ok = 0 |
| 370 | Hàng đợi alpha vẫn ngập tràn request retry cũ và mới | attempts_alpha tiếp tục dao động quanh 173 req/s, calls_ok = 0 |
| 453 | Hàng đợi bắt đầu xả bớt các request cũ, hệ thống bắt đầu có phản hồi | attempts_alpha hạ xuống 7 req/s, calls_ok phục hồi đạt 4 req/s |
| 455 | Hệ thống hồi phục hoàn toàn trạng thái bình thường | calls_ok ổn định trở lại >= 8 req/s (recovery_s = 155s sau t=300) |

Trigger vs root cause:
- Trigger: Node alpha bị giảm 75% năng lực xử lý trong khoảng thời gian từ t=120s đến t=300s.
- Root cause: Thiết kế client thiếu cơ chế bảo vệ độ tin cậy: lồng 3 tầng retry độc lập (tối đa 36 lần thử), delay cố định, không có circuit breaker để cắt tải, không có retry budget và không có cơ chế chuyển vùng dự phòng.
- Vì sao provider đã hồi phục mà hệ thống vẫn sập (Metastable failure): Khi máy chủ bị chậm, client timeout nhưng máy chủ vẫn tiếp tục tốn tài nguyên xử lý request đó. Cùng lúc đó client lại bắn thêm request retry mới vào hàng đợi. Máy chủ bị ngập bởi các request đã quá hạn, khiến request mới đến đều bị nghẽn quá 10s và trả về 529. Vòng lặp này tự duy trì quá tải ngay cả khi phần cứng đã hồi phục xong, và chỉ kết thúc khi client hết lượt thử hoặc tải đầu vào giảm bớt.

Điều gì giúp / điều gì làm tệ hơn:
- Làm tệ hơn: Thử lại liên tục với delay cố định, không nghe theo header retry-after, tiếp tục gửi request vào node đang nghẽn thay vì chuyển vùng.
- Giúp khắc phục: Dùng ResilientClient có circuit breaker ngắt mạch khi lỗi hoặc chậm, dùng full jitter backoff và failover sang alpha-eu hoặc beta.

Action items:

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Đổi toàn bộ sang ResilientClient có SlidingWindowBreaker và FullJitterBackoff | Prevent | Platform Team | Chạy kiểm tra băng G2 đạt amplification <= 1.2 và availability >= 95% |
| 2 | Áp dụng trần Retry Budget tối đa 10% lưu lượng | Mitigate | SRE Team | Thử tắt breaker thì amplification vẫn duy trì <= 1.4 |
| 3 | Cài đặt cảnh báo Multi-window Burn-rate cho latency và lỗi 5xx | Detect | Monitoring Team | Cảnh báo page kích hoạt trong vòng dưới 5 phút khi có sự cố sập node |

## 7. Một điểm yếu còn lại

Tình huống client xử lý chưa tối ưu:
Khi phải chuyển vùng hàng loạt sang node beta (như ở G6 hoặc G2), chi phí vận hành sẽ tăng cao vì beta có giá đắt hơn (hệ số 1.4x so với 1.0x của alpha), đồng thời độ trễ của beta cũng chậm hơn (median 1.8s so với 1.2s của alpha). Nếu gặp sự cố mạng diện rộng làm cả 4 node cùng bị chậm một lúc, circuit breaker ở tất cả các node đều có nguy cơ bị ngắt mạch, dẫn đến việc client hết lựa chọn và phải bỏ cuộc hàng loạt dù hệ thống phía sau chưa chết hẳn.

Cách khắc phục:
1. Bổ sung cơ chế fallback có tính đến chi phí và chế độ giảm tải nhẹ (graceful degradation): khi phải chuyển sang model đắt như beta thì tự động giảm bớt max_tokens hoặc rút gọn ngữ cảnh để tiết kiệm tiền.
2. Thay vì dùng ngưỡng thời gian cố định 5.0s, áp dụng thuật toán điều tiết đồng thời thích ứng (như TCP Vegas hoặc CoDel) để tự động điều chỉnh tốc độ gửi request theo độ trễ thực tế của nhà cung cấp, tránh việc ngắt mạch đồng loạt khi toàn mạng bị trễ.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | PASS |

| Tham số cache: | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Ở ngưỡng 0.80, cache bị trả nhầm 157 câu hỏi (other_question). Tăng lên 0.92 giúp xóa sạch câu trả lời sai (wrong_answers = 0) trong khi tỷ lệ trúng cache vẫn đạt 65.2% (vượt mốc yêu cầu 50%). |
| ttl_s | 60 | Giới hạn độ tươi của dữ liệu. Khi bảng giá ship đổi ngầm ở t=480 mà không có webhook, TTL 60s giúp khống chế số câu trả lời cũ chỉ còn 3 (outdated_answers = 3, đạt chuẩn dưới 15). Nếu để 300s thì con số này sẽ vọt lên 95. |
| max_stale_s | 300 | Cho phép dùng lại câu trả lời cũ tối đa 300s khi tất cả các node cùng sập (sự cố kéo dài 120s từ t=240 đến t=360). Nhờ cơ chế stale-if-error đã cứu được 157 câu hỏi, nâng faq_availability từ 84.24% lên 96.45% (vượt mốc 93%). |

Thí nghiệm:

1. Ngưỡng similarity:
- Ở ngưỡng 0.80: hit ratio là 66.6%, nhưng bị trả nhầm tới 157 câu hỏi do gộp nhầm các câu hỏi khác nhau như chính sách hoàn tiền và thời gian hoàn tiền, làm faq_availability chỉ đạt 87.33%.
- Ở ngưỡng 0.92: hit ratio đạt 65.2%, không còn câu nào bị trả nhầm (wrong_answers = 0), chi phí mỗi lượt là 0.915 và faq_availability đạt 96.45%.
- Ở ngưỡng 0.97: do siết quá chặt nên hit ratio giảm còn 58.1%, bỏ sót nhiều câu hỏi tương tự nhau và chi phí tăng lên 0.964.

2. TTL:
- Với TTL = 30s: outdated_answers = 2, hit ratio đạt 59.2%.
- Với TTL = 60s: outdated_answers = 3, hit ratio tối ưu ở mức 65.2%.
- Với TTL = 300s: outdated_answers tăng vọt lên 95 (vượt ngưỡng 15). Webhook chỉ báo khi chính sách hoàn tiền thay đổi công khai; còn các cập nhật ngầm không báo trước (như bảng phí ship ở t=480) thì webhook không có tác dụng, lúc này TTL ngắn là lớp phòng vệ duy nhất để đẩy dữ liệu cũ ra khỏi cache.

3. Stale-if-error:
- Khi tắt tính năng này (max_stale_s = 0): số lượt phục vụ câu trả lời cũ stale_served bằng 0, kéo theo faq_availability giảm từ 96.45% xuống còn 87.33% (dưới mức yêu cầu 93%). Trong 2 phút mọi provider sập hoàn toàn (t=240 đến t=360), người dùng hỏi FAQ đều bị từ chối phục vụ.

Trả lời:

- Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?
Thứ nhất là tính cập nhật: trạng thái đơn hàng thay đổi liên tục theo thời gian thực nên lấy từ cache sẽ dễ trả về thông tin cũ đã lỗi thời. Thứ hai là bảo mật: việc lưu thông tin cá nhân vào cache tiềm ẩn nguy cơ lộ dữ liệu giữa các phiên hoặc các user khác nhau nếu việc chia scope bị lỗi. Thứ ba là hiệu quả bộ nhớ: mỗi đơn hàng chỉ thuộc về một khách hàng và ít khi hỏi lặp lại nhiều lần, việc lưu vào cache sẽ chiếm chỗ vô ích mà không đem lại tỷ lệ hit cao.

- Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?
Không thể chỉ nhìn vào availability tổng của user vì lúc này cache đang đỡ tải. Thay vào đó on-call cần thiết lập cảnh báo dựa trên tỷ lệ lỗi 5xx/timeout thực tế từ các provider upstream (hoặc trạng thái ngắt mạch của circuit breaker), và theo dõi tỷ lệ phục vụ câu trả lời cũ (stale_served rate). Khi thấy lượng stale_served tăng vọt đột biến, đội trực sẽ biết ngay là hạ tầng provider phía sau đang gặp sự cố và hệ thống đang phải chạy bằng dữ liệu đệm dự phòng.
