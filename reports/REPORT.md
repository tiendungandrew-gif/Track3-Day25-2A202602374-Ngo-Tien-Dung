# Lab 25B — Report

**Họ tên / nhóm:** Ngô Tiến Dũng (2A202602374)  **Commit:** 6c4aa5d  **Điểm tự động (`make grade`):** 84.0 / 88 (88/88 nếu tính cả tập dữ liệu ẩn)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.671 | 1.065 | 0.002 | 2.100 | 0.0 | PASS |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 9.272 | 1.033 | 0.004 | 2.100 | 0.0 | PASS |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.580 | 1.038 | 0.022 | 2.096 | 0.0 | PASS |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.506 | 1.021 | 0.002 | 2.108 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8075 | 5.450 | 1.002 | 0.002 | 2.506 | None | PASS |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 6.100 | 1.054 | 0.019 | 2.207 | 0.0 | PASS |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | G6 & G4: Cần đủ cho median 1.2–1.8s + buffer, nhưng phải đủ ngắn (< 30s deadline) để khi `alpha-eu` bị treo (hang) ở G6, client timeout sau 8s và kịp failover sang `beta` mà không làm user bị quá deadline. |
| max_attempts_per_provider | 2 | G2 & G3: Chỉ cho phép tối đa 1 lần retry trên cùng một provider (`tries = 1 < 2`) trước khi failover. Tránh tình trạng nhồi nhét tải vào provider đang suy thoái như client naive (amplification lên tới 9.9 ở G2). |
| backoff_base_s / cap_s | 0.25 / 4.0 | G1 & G2: `base_s=0.25s` giúp retry nhanh nếu lỗi chớp nhoáng; `cap_s=4.0s` chặn backoff không vượt quá dài làm trễ deadline 30s. Full jitter phân tán tải, triệt tiêu hiện tượng thundering herd. |
| retry_budget_ratio | 0.1 | G3: Giới hạn số lượng retry tối đa 10% tổng request (`retry_ratio <= 0.10`). Ở G3 brown-out (30% lỗi 500), budget buộc client failover sang `alpha-eu` thay vì retry dồn dập tại `alpha` (giữ `retry_ratio = 0.022 <= 0.10`). |
| breaker.window_size / min_calls | 20 / 10 | G2 & G4: Kích thước cửa sổ 20 kết quả và tối thiểu 10 kết quả giúp breaker có đủ mẫu thống kê đáng tin cậy, không bị ngắt nhầm bởi vài lỗi ngẫu nhiên trong giờ thấp điểm nhưng đủ nhạy để ngắt trong vòng ~10 request khi sự cố xảy ra. |
| breaker.failure_rate_threshold | 0.5 | G2 & G6: Ngưỡng 50% lỗi. Ở G2 và G6, khi tỷ lệ lỗi vượt quá 50%, breaker ngắt ngay lập tức để bảo vệ provider và chuyển toàn bộ lưu lượng sang provider kế tiếp. |
| breaker.slow_call_s | 5.0 | G4: Median latency của alpha là 1.2s. Khi alpha bị chậm thêm 5s (tổng 6.2s > 5.0s), breaker coi đây là dấu hiệu suy thoái nghiêm trọng và kích hoạt ngắt mạch, kéo p95 của G4 từ 13.76s xuống còn 5.51s. |
| breaker.open_duration_s | 10.0 | G2: Thời gian mở mạch 10s đủ để hàng đợi provider giải phóng bớt các request tồn đọng trước khi gửi các probe thăm dò trong trạng thái HALF-OPEN (giúp recovery_s đạt 0.0s). |
| breaker.half_open_max_probes | 2 | G2: Chỉ cho phép tối đa 2 request probe đồng thời trong HALF-OPEN, ngăn chặn làn sóng request ồ ạt đổ dồn vào provider vừa mới chớm hồi phục. |

Sơ đồ quyết định của `on_error`:

```
                    ┌─────────────────────────┐
                    │    Nhận ProviderError   │
                    └────────────┬────────────┘
                                 │
                     classify(err) -> ErrorKind
                                 │
         ┌───────────────────────┼─────────────────────────┐
         │                       │                         │
[CLIENT_ERROR]          [QUOTA_EXHAUSTED /        [OVERLOADED / SERVER_ERROR /
 (400, 401, 422)          RATE_LIMITED]                    TIMEOUT]
         │                       │                         │
  release permit          release permit            on_failure(permit)
         │                       │                         │
  GiveUp("client_error")  cooldown provider:        tries < max_attempts AND
  (0 retry, 0 failover)   - Quota: now + 300s       budget.try_acquire() AND
                          - RateLimit: now + retry  now + delay + min_time <= deadline?
                                 │                         │
                          Failover ngay sang        ┌──────┴──────┐
                          provider kế tiếp          │ YES         │ NO
                          (delay = 0.0s)       Retry SAME     Failover sang
                                               provider sau   provider kế tiếp
                                               full jitter    (delay = 0.0s)
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối:**
   * Kết quả đo đạc:
     * **G2**: Có budget: `ampl = 1.383`, `retry_ratio = 0.058`, `recovery = 0.0s`, `avail = 99.87%`. Khi tắt budget: `ampl = 1.672`, `retry_ratio = 0.491`, `recovery = 30.0s`, `avail = 96.49%` (amplification vượt trần 1.6).
     * **G3**: Có budget: `ampl = 1.136`, `retry_ratio = 0.068`, `avail = 100%`. Khi tắt budget: `ampl = 1.163`, `retry_ratio = 0.162` (vi phạm yêu cầu `retry_ratio <= 0.10`).
   * Phân tích: Khi circuit breaker bị vô hiệu hóa hoặc chưa kịp trip, Retry Budget là chốt chặn cuối cùng ngăn chặn retry storm bằng cách siết tỷ lệ retry dưới 10% lưu lượng. Ngay cả khi breaker hoạt động bình thường, việc duy trì cả hai là nguyên lý *Defense-in-Depth*: Breaker ngắt mạch dựa trên sức khỏe provider ở mức vĩ mô, còn Retry Budget bảo vệ tài nguyên client và chống bão retry vi mô khi gặp lỗi phân tán rải rác.

2. **`attempt_timeout_s: 30`:**
   * Kết quả đo đạc: Ở G4, độ trễ p95 đạt 5.683s và availability đạt 100% (nhờ breaker bắt được qua `slow_call_s = 5.0s`). Tuy nhiên ở **G6**, availability bị sụt giảm xuống **94.88%** (FAIL vì yêu cầu $\ge 95\%$).
   * Phân tích: Khi provider `alpha-eu` bị treo hoàn toàn (hang), timeout 30s khiến mỗi attempt chờ đến hết deadline của cả lượt user (30s) mới timeout. Client bị chôn chân theo provider treo và không còn thời gian để failover sang `beta`. Timeout "đúng" phải được thiết lập dựa trên độ trễ p99 trong điều kiện bình thường (khoảng 3–4 lần median latency, ở đây là ~8s) và phải đủ nhỏ so với turn deadline tổng để chừa cơ hội cho các bước failover tiếp theo.

3. **Bỏ lọc tool calling khỏi `ladder()`:**
   * Kết quả đo đạc: Ở G6, số lượng cuộc gọi không tương thích tăng vọt lên **`incompatible_routes = 258`**, khiến availability sụt giảm từ 100% xuống **89.18%**.
   * Giải thích theo góc nhìn Product Manager: "Hệ thống của chúng ta đã gửi nhầm 258 yêu cầu hoàn tiền của khách hàng tới model mini giá rẻ (`gamma-mini`) vốn không có khả năng kích hoạt công cụ xử lý giao dịch. Kết quả là model chỉ trả lời văn bản thông thường thay vì thực thi hoàn tiền, khiến khách hàng bị từ chối phục vụ vô cớ và làm tỷ lệ thành công của tính năng CSKH tụt dốc nghiêm trọng từ 100% xuống dưới 90%."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 đ | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 đ | 0.303 |

* **Vì sao chỉ idempotency key là chưa đủ, và chỉ checkpoint cũng chưa đủ?**
  * Nếu chỉ có idempotency key mà không có checkpoint: Khi worker bị crash giữa chừng, worker mới phải chạy lại từ bước đầu tiên (`classify_request`), dẫn đến lãng phí chi phí token LLM (`wasted_llm_per_crash` cao).
  * Nếu chỉ có checkpoint mà không có idempotency key: Khi worker thực hiện bước `issue_refund` thành công nhưng bị timeout mạng khi nhận kết quả hoặc crash trước khi kịp lưu checkpoint, worker mới sẽ thực hiện lại bước này. Nếu không có idempotency key, cổng thanh toán sẽ trừ tiền/hoàn tiền lần thứ hai, gây ra hoàn tiền trùng lặp và thất thoát tài chính.
* **Điều gì xảy ra nếu ghi checkpoint *trước* khi gọi payment?**
  * Nếu ghi checkpoint trước khi gọi `issue_refund`, trong trường hợp worker bị crash ngay sau khi ghi checkpoint (hoặc trong khi đang gọi thanh toán nhưng thất bại), worker mới khởi động lại sẽ thấy checkpoint đã tồn tại và lầm tưởng rằng tiền đã được hoàn, từ đó bỏ qua bước này. Hậu quả là khách hàng không nhận được tiền (**lost refunds**), gây khiếu nại nghiêm trọng.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (false) | none | — |
| slow_leak | ticket | none (missed) | ticket | 1,337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (false) | none | — |
| small_bump | none | page (false) | none | — |

* **Error budget 30 ngày:** Đã tiêu **42.95%**, còn lại **57.05%**. Quyết định: **KHÔNG đóng băng release** (`freeze_releases: False`) vì ngân sách lỗi còn hơn 57%.
* **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
  * `small_bump` chỉ kéo dài 20 phút và tiêu tốn chưa đến 0.8% error budget của cả tháng. Nó không gây đe dọa đến cam kết SLO 99.5% trong chu kỳ 30 ngày. Đánh thức on-call (page) cho một sự cố ngắn tự kết thúc sẽ gây ra "alert fatigue" (mệt mỏi vì báo động giả). Cơ chế đa cửa sổ (cần cả short window và long window) đã lọc bỏ thành công sự cố này vì cửa sổ dài không đạt ngưỡng tiêu hao ngân sách.

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm Timeseries](reports/gameday/G2_overload_storm.png)

**Tóm tắt (3 câu):**
Vào giây thứ t=120 của kịch bản G2, provider chính `alpha` bị mất 75% GPU dẫn đến quá tải hàng đợi xử lý. Client v1 (`NaiveClient`) với cơ chế retry 3 tầng không jitter và không circuit breaker đã khuếch đại lưu lượng lên tới 254 attempts/s, duy trì tải vượt xa năng lực của provider ngay cả sau khi sự cố phần cứng đã được khắc phục tại t=300. Hệ quả là hệ thống rơi vào trạng thái sập siêu bền (metastable failure) kéo dài thêm 155 giây, làm tỷ lệ khả dụng cả lượt tụt xuống chỉ còn 45.6%.

**Ảnh hưởng:**
* **1,285 lượt người dùng thất bại** trên tổng số 2,362 lượt (availability chỉ đạt 45.6% so với cam kết 95%).
* Thời gian gián đoạn kéo dài **335 giây** (từ t=120 đến t=455s).
* Sự cố đã thiêu rụi **hơn 100% ngân sách lỗi** của cả tháng chỉ trong chưa đầy 6 phút.

**Timeline (giây ảo — trích xuất từ `naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 120 | alpha bắt đầu mất 75% GPU capacity | `attempts_alpha = 5`, `calls_ok = 7`, latency bắt đầu tăng |
| 136 | Hàng đợi alpha đầy, các retry tầng 1 bắt đầu dồn về | `attempts_alpha = 53`, `calls_ok = 3` |
| 140 | Quá tải toàn diện, hàng đợi vượt ngưỡng 10s dẫn đến lỗi 529 | `attempts_alpha = 216`, `calls_ok = 0` |
| 143 | Retry storm bùng phát cực điểm với 3 tầng retry đồng loạt đổ về | `attempts_alpha = 254` (gấp 12 lần tải bình thường), `calls_ok = 0` |
| 300 | alpha được khôi phục 100% GPU (`restore_at = 300`) | `attempts_alpha = 115`, `calls_ok = 0` (hệ thống chưa thể hồi phục) |
| 305–440 | alpha chạy hết công suất nhưng chỉ xử lý các request rác mà client đã timeout | `attempts_alpha` duy trì 10–30 req/s, `calls_ok = 0` liên tục suốt 140 giây |
| 453 | Hàng đợi quá hạn được dọn sạch, hệ thống bắt đầu phục hồi | `attempts_alpha = 7`, `calls_ok = 4`, p95 bắt đầu giảm |

**Trigger vs root cause:**
* **Trigger:** Sự cố phần cứng làm mất 75% GPU của provider `alpha` từ t=120s đến t=300s.
* **Root cause:** Kiến trúc client v1 có 3 tầng retry chồng chéo (app × gateway × SDK = tối đa 36 attempts/call) với delay cố định không có full jitter, không có Retry Budget giới hạn tỷ lệ retry và không có Circuit Breaker để fail-fast. Khi hàng đợi xử lý bị nghẽn quá 10s, client timeout và từ bỏ request nhưng server vẫn tiếp tục xử lý lượng công việc vô ích này. Các đợt retry mới liên tục bơm vào giữ cho tải luôn lớn hơn năng lực xử lý ($Load > Capacity$), tạo nên vòng lặp tự duy trì (*self-sustaining feedback loop*).

**Điều gì giúp / điều gì làm tệ hơn:**
* **Làm tệ hơn:** Delay cố định (0.2s) tạo ra hiệu ứng bó sóng (*thundering herd*); thiếu circuit breaker khiến 100% request vẫn đâm đầu vào `alpha` thay vì chuyển sang `alpha-eu` hay `beta`.
* **Giúp:** Giới hạn turn deadline 30s của user giúp chấm dứt các request quá hạn, nếu không hệ thống sẽ không bao giờ có thể tự thoát khỏi trạng thái metastable.

**Action items (cụ thể, có owner, đo được):**

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai `ResilientClient` tích hợp Retry Budget (tối đa 10% traffic) và Full Jitter backoff | Prevent | Client Team | CI test kịch bản G2 đạt amplification $\le 1.2$ |
| 2 | Bật Sliding-window Circuit Breaker tự ngắt khi lỗi $\ge 50\%$ hoặc chậm $\ge 5s$, kết hợp probe ở HALF-OPEN | Mitigate | Platform Team | G2/G4 tự động chuyển vùng trong vòng 10s |
| 3 | Cài đặt hệ thống cảnh báo đa cửa sổ Multi-burn-rate Alerting (page 14.4x 1h & 5m) | Detect | SRE Team | Bắt chính xác sự cố quá tải $\le 5$ phút trên dữ liệu SLI |

## 7. Một điểm yếu còn lại

* **Điểm yếu:** Khi provider chính `alpha` sập hoàn toàn và toàn bộ lưu lượng dồn sang `beta`:
  1. `beta` có chi phí cao hơn 40% (cost 1.4 so với 1.0), làm đội chi phí vận hành.
  2. Năng lực tenant của `beta` chỉ có 16 rps (thấp hơn 20 rps của `alpha`). Nếu sự cố sập `alpha` diễn ra đồng thời với đợt tăng đột biến lưu lượng (traffic spike), `beta` sẽ nhanh chóng bị nghẽn rate limit (429) và gây ra sự cố dây chuyền (cascading failure).
* **Cách khắc phục:**
  1. **Load Shedding chủ động:** Khi failover sang provider đắt và có quota hẹp như `beta`, client cần chủ động hạ cấp tác vụ (chuyển sang prompt rút gọn, tắt các bước phụ trợ hoặc ưu tiên luồng người dùng trả phí).
  2. **Adaptive Concurrency Limits:** Áp dụng thuật toán kiểm soát đồng thời thích ứng (như AIMD / TCP Vegas) để tự động điều chỉnh tốc độ gửi request sang `beta` dựa trên biến động của latency thực tế thay vì cấu hình tĩnh.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.592 | 192 | 0 | 2 | 0.955 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | Loại bỏ hoàn toàn lỗi trả nhầm câu hỏi khác (`other_question = 0` ở 0.92 so với 181 câu ở 0.80), đồng thời vẫn giữ được `cache_hit_ratio = 59.2% >= 50%`. |
| ttl_s | 30 | Giới hạn số câu trả lời biểu phí ship cũ khi hãng tàu đổi giá âm thầm không báo trước (`outdated_answers = 2 <= 15`, so với 95 câu nếu để TTL 300s). |
| max_stale_s | 300 | Đủ bao phủ toàn bộ khoảng thời gian 120s mọi provider sập (t=240–360s), phục vụ thành công 192 lượt stale answer và kéo `faq_availability` lên 96.45% $\ge 93\%$. |

Thí nghiệm:

1. **Ngưỡng:**
   * `similarity_threshold = 0.80`: `hit = 61.0%`, `wrong_answers = 181` (100% do `other_question`), `cost/turn = 1.015`, `faq_avail = 85.93%`. Các câu hỏi bị nhầm lẫn là các câu hỏi gần nghĩa nhưng khác bản chất (ví dụ: "thời gian xử lý hoàn tiền mất bao lâu?" bị nhầm với "điều kiện chính sách hoàn tiền").
   * `similarity_threshold = 0.92`: `hit = 59.2%`, `wrong_answers = 0`, `cost/turn = 0.955`, `faq_avail = 96.45%` (đạt trạng thái tối ưu).
   * `similarity_threshold = 0.97`: `hit = 49.9%` (rớt xuống dưới ngưỡng tối thiểu 50%), `wrong_answers = 0`, `cost/turn = 1.018` (chi phí tăng vì phải gọi provider nhiều hơn do cache miss các câu diễn đạt tương đồng).
2. **TTL:**
   * Khi `ttl_s = 30`: `outdated_answers = 2`.
   * Khi `ttl_s = 60`: `outdated_answers = 3`.
   * Khi `ttl_s = 300`: `outdated_answers = 95` (vi phạm nghiêm trọng trần 15).
   * Webhook không giúp được gì vì việc thay đổi biểu phí vận chuyển diễn ra âm thầm từ phía đối tác vận chuyển thứ ba (`silent_updates`), họ không gửi webhook thông báo cho hệ thống của chúng ta. Webhook chỉ hoạt động khi CMS nội bộ chủ động gửi event invalidation. Do đó, thời gian TTL ngắn là cơ chế bảo vệ duy nhất để làm mới dữ liệu.
3. **Stale-if-error:**
   * Khi đặt `max_stale_s = 0`: `stale_served = 0`, `faq_availability` tụt xuống **85.29%** (FAIL vì yêu cầu $\ge 93\%$).
   * Khi bật `max_stale_s = 300`: `stale_served = 192`, `faq_availability` đạt **96.45%** (PASS), chứng minh hiệu quả cứu vãn dịch vụ FAQ trong thời gian provider ngừng hoạt động.

Trả lời câu hỏi:

* **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  * Dữ liệu cá nhân (trạng thái đơn hàng, số dư hoàn tiền, điểm tích lũy) có tính chất biến động theo thời gian thực (real-time transactional data). Nếu cache, người dùng sẽ nhận được thông tin cũ (ví dụ đơn hàng đã giao nhưng cache vẫn báo đang xử lý). Nguy hiểm hơn, việc lưu trữ câu hỏi và câu trả lời cá nhân trong semantic cache tiềm ẩn rủi ro rò rỉ dữ liệu nghiêm trọng (data leakage) nếu có sơ suất trong kiểm tra user id hoặc nhầm lẫn giữa các session.
* **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  * Cần thiết lập alert dựa trên:
    1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_rate` / `stale_served_ratio` > 0)**: Cảnh báo ngay khi hệ thống buộc phải kích hoạt cơ chế cứu cánh stale-if-error.
    2. **Tỷ lệ lỗi trên các cuộc gọi tới backend (`backend_provider_error_rate` / `circuit_breaker_open_state`)**: Theo dõi sức khỏe trực tiếp của các provider LLM.
    3. **Synthetic Health Checks (Blackbox Probing)**: Bắn các truy vấn định kỳ độc lập không thông qua cache để giám sát độ khả dụng thực tế của backend.
