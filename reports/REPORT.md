# Lab 25B — Report

**Họ tên / nhóm:** Trương Việt Anh — 2A202602444  
**Commit nền:** `18cc181d9577bd2e4c45ab64eb6e23da86c20030` 
**Điểm tự động (`make grade`):** **84 / 88** (toàn bộ phần public đạt; 4 điểm SLO hidden không có dữ liệu để chạy cục bộ)

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4 | Không |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0 | Có |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155 | Không |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0 | Có |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0 | Không |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0 | Có |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0 | Không |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0 | Có |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | N/A | Không |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | Có |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0 | Không |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0 | Có |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 s | Chặn request chậm trước deadline 30 s. Ở G4, p95 giảm từ 13.764 s xuống 5.562 s; đặt 30 s làm G6 availability giảm còn 0.9488. |
| `max_attempts_per_provider` | 2 | Cho một lần retry cục bộ nhưng chuyển vùng sớm. Kết quả G2 amplification chỉ 1.041 thay vì 9.904 của naive. |
| `backoff_base_s / cap_s` | 0.25 / 4.0 s | Full jitter tách các retry đồng loạt. G1 đạt availability 1.0 và retry ratio 0.002 trong đợt rate-limit. |
| `retry_budget_ratio` | 0.10 | Giới hạn retry theo first attempts. Khi breaker bị vô hiệu, budget giữ G2 amplification 1.380; tắt budget làm tăng lên 1.678. |
| `breaker.window_size / min_calls` | 20 / 10 | Cửa sổ đủ nhạy nhưng cần 10 mẫu để tránh mở bởi vài lỗi lẻ; G2 hồi phục 0 s và không tạo retry storm. |
| `breaker.failure_rate_threshold` | 0.50 | Mở khi ít nhất nửa cửa sổ thất bại; G6 failover sạch và availability 1.0. |
| `breaker.slow_call_s` | 5.0 s | Thấp hơn timeout 8 s để slow-call rate cảnh báo trước khi deadline bị tiêu hết; G4 p95 5.562 s. |
| `breaker.open_duration_s` | 10.0 s | Giảm tải đủ lâu nhưng thử hồi phục nhanh; G1/G2/G3/G6 đều recovery 0 s theo tiêu chí băng. |
| `breaker.half_open_max_probes` | 2 | Giới hạn tải thăm dò và yêu cầu 2 thành công trước khi đóng; tránh dội lưu lượng ngay khi provider vừa hồi. |

Quyết định khi nhận lỗi:

| Loại lỗi | Ghi breaker? | Retry cùng provider? | Failover? | Cooldown? |
|---|:---:|:---:|:---:|:---:|
| timeout / 5xx / overloaded | Có | Có, nếu còn budget + thời gian | Có | Không |
| rate limit / quota | Không | Chỉ sau `Retry-After` nếu còn deadline | Có | Có, theo `Retry-After` hoặc quota cooldown |
| auth / invalid request / content policy | Không | Không | Không | Không |
| model/output không tương thích | Không | Không | Có sang route tương thích | Không |

Mọi retry còn phải qua ba cổng chung: deadline, full-jitter backoff và token bucket retry budget. Ladder được lọc theo capability nên bước tool không bao giờ chọn model không hỗ trợ tool.

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối.** Khi cố tình vô hiệu breaker và tăng tối đa lên 5 attempts/provider, bật budget cho G2: availability 0.9987, amplification 1.380, retry ratio 0.058, recovery 0 s; tắt budget: 0.9661, 1.678, 0.494, recovery 20 s. Với G3, bật budget cho amplification/retry ratio 1.136/0.068, còn tắt budget là 1.162/0.159. Khi breaker hoạt động đúng, nó cắt phần lớn retry sớm hơn nên budget ít xuất hiện trên happy path, nhưng vẫn cần vì budget bảo vệ toàn cục trước cấu hình breaker sai, trạng thái breaker chưa đủ mẫu, hoặc lỗi phân tán trên nhiều provider.

2. **`attempt_timeout_s: 30`.** G4 vẫn có availability 1.0 và p95 5.608 s, nhưng G6 chỉ còn availability 0.9488 (p95 5.315 s): attempt treo có thể chiếm gần trọn deadline, không còn thời gian failover. Timeout đúng phải dựa vào phân phối latency tốt của provider, SLO p95 và phần deadline cần dành cho bước tool, backoff và fallback; không nên chỉ sao chép toàn bộ turn deadline.

3. **Bỏ lọc tool calling khỏi `ladder()`.** G6 giảm availability từ 1.0000 xuống 0.9019 và tạo **234 incompatible routes**; cost/good-turn cũng tăng từ 2.189 lên 2.272. Theo ngôn ngữ sản phẩm: 234 lượt khách được chuyển đến một “nhân viên” không có công cụ cần thiết, nên dù hệ thống đó đang online, yêu cầu vẫn không thể hoàn tất. Lọc capability là điều kiện đúng chức năng, không chỉ là tối ưu hiệu năng.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.000 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | 1.000 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Idempotency key làm side effect lặp lại trở thành cùng một thao tác, nhưng không cho executor biết bước nào đã hoàn tất; thiếu checkpoint thì lần resume vẫn gọi lại mọi bước và lãng phí LLM/API. Checkpoint cho biết điểm resume, nhưng một crash giữa payment thành công và ghi checkpoint vẫn có thể hoàn tiền lần hai; vì vậy payment/email vẫn cần idempotency key ổn định. Nếu ghi checkpoint **trước** payment, crash sau checkpoint nhưng trước lời gọi sẽ khiến lần resume bỏ qua payment, tạo lost refund. Thiết kế dùng key ổn định cho side effect và chỉ checkpoint sau khi side effect trả về thành công.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ burn-rate |
|---|---|---|---|---:|
| checkout_bug | page | page, 1 phút | page | 27 phút |
| provider_outage | page | page, 0 phút | page | 4 phút |
| deploy_blip | none | page sai | không alert | N/A |
| slow_leak | ticket | bỏ sót | ticket | 1,337 phút |
| degraded_evening | page | page, 214 phút | page | 291 phút |
| night_spike | none | page sai | không alert | N/A |
| small_bump | none | page sai | không alert | N/A |

Trong 25,901,755 request, budget cho phép 129,508.8 bad request và thực tế có 55,618: đã tiêu **42.95%**, còn **57.05%**, availability 99.7853%. Vì budget vẫn còn hơn một nửa nên `freeze_releases = false`. `small_bump` chỉ kéo dài 20 phút ở 12% lỗi: nó không đốt đủ budget ở cả hai cửa sổ để biểu thị nguy cơ SLO dài hạn, nên paging lúc 11 giờ trưa chỉ gây alert fatigue; nếu cần theo dõi thì dùng dashboard/ticket thay vì đánh thức on-call.

## 6. Postmortem — Băng G2 với client naive (blameless)

**Tóm tắt.** Trong G2, alpha mất 75% năng lực tại t=120 s; retry ba tầng không jitter, không budget và không breaker khuếch đại tải. Dù alpha hồi phục ở t=300 s, hàng đợi chứa request mà client đã bỏ và retry mới giữ tải cao hơn năng lực, tạo metastable loop. Hệ thống chỉ hồi phục sau 155 s, trong khi resilient client đạt availability 1.0, amplification 1.041 và recovery 0 s.

**Ảnh hưởng.** 1,285 / 2,362 lượt user thất bại (availability 45.60%) trong băng 600 s. Nếu coi mức mất availability này là full-outage tương đương, sự cố tiêu thụ khoảng `10 phút × 54.4% = 5.44 phút`; với SLO 99.5%, budget tháng 30 ngày là 216 phút, nên sự cố dùng khoảng **2.52% error budget tháng**. Đây là phép quy đổi xấp xỉ theo thời lượng/mức lỗi của băng.

![G2 overload storm](gameday/G2_overload_storm.png)

| t / cửa sổ | Sự kiện | Bằng chứng từ CSV |
|---:|---|---|
| 110–119 | Trạng thái ổn định trước sự cố | 9.1 attempts/s; success ratio 1.000 |
| 120–129 | Alpha mất 75% năng lực | 6.2 attempts/s; success ratio 1.000, hàng đợi bắt đầu tích tụ |
| 180–189 | Retry storm đã hình thành | 148.1 attempts/s; success ratio 0.000 |
| 290–299 | Ngay trước khi provider hồi | 146.1 attempts/s; success ratio 0.000 |
| 300–309 | Alpha hồi năng lực nhưng backlog còn đầy | 38.8 attempts/s; success ratio 0.000 |
| 360–369 | Vòng lặp metastable tiếp tục | 110.8 attempts/s; success ratio 0.000 |
| 445–454 | Bắt đầu thoát backlog | 13.8 attempts/s; success ratio 0.200 |
| 455–464 | Hệ thống hồi phục | 13.0 attempts/s; success ratio 0.857 |
| 590–599 | Trở lại ổn định | 7.1 attempts/s; success ratio 1.000 |

**Trigger vs root cause.** Trigger là alpha đột ngột mất 75% GPU/năng lực. Root cause mang tính hệ thống là chính sách retry ba tầng không jitter, retry budget hay circuit breaker: timeout sinh retry, retry tăng hàng đợi, hàng đợi làm thêm timeout rồi lại sinh retry. Sau t=300, provider phục hồi nhưng vẫn phải xử lý backlog của nhiều request mà client đã hết chờ; retry hiện tại tiếp tục giữ offered load trên năng lực, vì vậy vòng lặp tự duy trì thêm 155 s. Đây là thiếu guardrail và kiểm thử tải trong quy trình, không phải lỗi của một cá nhân.

**Điều gì giúp / điều gì làm tệ hơn.** Provider hồi năng lực và lưu lượng retry cuối cùng giảm giúp hệ thống thoát vòng lặp. Retry đồng bộ, nhiều tầng, deadline không được truyền xuyên suốt và tiếp tục xử lý request đã bị client bỏ làm tình hình tệ hơn. Client mới kết hợp full jitter, deadline, budget, breaker và failover nên G2 giảm amplification từ 9.904 xuống 1.041, cost/good-turn từ 4.627 xuống 2.119 và loại bỏ recovery lag.

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Đưa full-jitter + retry budget vào thư viện client chuẩn | prevent | Platform | CI G2: amplification ≤ 1.20 và retry ratio ≤ 0.10 trên 3 seed |
| 2 | Triển khai circuit breaker theo provider/route và bounded half-open probes | mitigate | Runtime | Chaos test mất 75% capacity: recovery ≤ 30 s, availability ≥ 0.99 |
| 3 | Dashboard và page theo amplification, queue depth và burn rate | detect | SRE | Inject G2: page trong 5 phút, 0 false page trên deploy_blip/night_spike |
| 4 | Truyền cancellation/deadline xuống provider để bỏ việc vô chủ | mitigate | Provider team | Sau client timeout, orphan work < 1% và queue trở về baseline ≤ 30 s |

## 7. Một điểm yếu còn lại

G2 đạt availability 1.0 nhưng p95 vẫn là 10.920 s, cao hơn nhiều băng khác, vì failover sang beta/region khác vẫn có latency và cost cao hơn. Nếu cả bốn provider cùng chậm, breaker có thể bảo vệ hệ thống nhưng không thể tạo thêm năng lực; user vẫn nhận lỗi nhanh hoặc trễ gần deadline. Bước tiếp theo là dùng timeout thích nghi theo latency quantile của từng route, hedging có kiểm soát chỉ cho request idempotent khi còn budget, và admission control/load shedding; sẽ kiểm chứng rằng G2 p95 giảm mà amplification vẫn ≤ 1.20 và cost/good-turn không tăng quá ngưỡng.

## 8. Cache (Part G)

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | Không |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | Có |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.90 | 0.80 tạo 157 wrong answers; 0.97 an toàn nhưng hit ratio chỉ 0.581 và cost 0.964. 0.90 giữ wrong=0, hit=0.652, cost=0.915. |
| `ttl_s` | 60 s | TTL 30 cho hit 0.592/cost 0.955; TTL 300 tăng hit 0.731 nhưng tạo 95 outdated answers. 60 s chỉ có 3 outdated. |
| `max_stale_s` | 600 s | Cho phục vụ 157 stale answer và FAQ availability 0.9645 khi outage; đặt 0 làm FAQ availability giảm còn 0.8733. |

1. **Ngưỡng.** Ở 0.80: hit ratio 0.666, wrong answers 157, cost 0.965; toàn bộ lỗi thuộc `other_question`, tức các câu gần về embedding nhưng khác ý định bị lấy chung câu trả lời. Ở 0.90: 0.652, 0, 0.915. Ở 0.97: 0.581, 0, 0.964. Ngưỡng 0.90 là điểm cân bằng: không trả nhầm nhưng vẫn giảm chi phí tốt nhất trong ba cấu hình đo được.

2. **TTL.** TTL 30/60/300 lần lượt cho hit ratio 0.592/0.652/0.731, cost 0.955/0.915/0.863, và outdated answers 2/3/95. Webhook không giúp ở tình huống nguồn thay đổi nhưng sự kiện invalidation bị thiếu hoặc đến chậm; entry cũ vẫn hợp lệ theo cache. TTL là giới hạn thời gian độc lập đảm bảo entry cuối cùng phải tái xác thực, nên chọn 60 s để tránh 95 câu cũ của TTL 300 mà vẫn có hit ratio tốt hơn TTL 30.

3. **Stale-if-error.** Với `max_stale_s: 0`, không có stale nào được phục vụ và FAQ availability giảm từ 0.9645 xuống 0.8733 (giảm 9.12 điểm phần trăm). Cửa sổ 600 s phục vụ 157 stale answers trong outage, nhưng vẫn bị giới hạn tuổi để tránh dùng dữ liệu vô thời hạn.

Câu hỏi cá nhân như “Đơn #48213 của tôi đang ở đâu?” không được cache vì dữ liệu nhạy cảm và thay đổi nhanh; cache theo user vẫn có rủi ro sai phân quyền, tài khoản dùng chung, xóa dữ liệu không đồng bộ và trả trạng thái cũ gây hậu quả nghiệp vụ. Trong outage, dashboard “xanh” ở tầng trải nghiệm nhờ stale cache không có nghĩa upstream khỏe. On-call cần alert theo **origin/provider error rate và burn rate riêng**, đồng thời theo `stale_served / requests`, tuổi stale cực đại và cache-miss availability; tỷ lệ stale tăng đột biến (157 trong băng) phải tạo tín hiệu degraded ngay cả khi FAQ availability vẫn 0.9645.
