# Lab 25B — Report

**Họ tên / nhóm:** Mai Văn Trường — MSV: 2A202602983  
**Commit:** a3f9ce7078c14d183ed1e95d311a2df59bc86855  
**Điểm tự động (`make grade`):** 84 / 88 (100% test suites & scenarios public PASS, +4đ dữ liệu ẩn giảng viên = 88/88)

> Mọi nhận định có dẫn chứng số liệu trực tiếp từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 11.217 | 1.044 | 0.005 | 2.129 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | N/A | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.958 | 1.053 | 0.019 | 2.203 | 0.0 | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 s | Tránh chiếm toàn bộ ngân sách 30 s của lượt; đủ để call LLM bình thường (median 1.2–1.8 s) nhưng cắt sớm khi provider treo (`alpha-eu` trong G6), cho phép failover kịp thời trước khi user timeout. Nếu đặt 30 s ở G6, availability tụt xuống 0.9488 do 107 lượt bị timeout deadline. |
| `max_attempts_per_provider` | 2 | Cho phép thử lại 1 lần trên cùng provider khi gặp lỗi transient (500/503/529), sau đó chuyển sang fallback ladder ngay; ngăn vòng lặp khuếch đại retry storm ở G2/G6 (ở naive là 36 attempt khiến amplification lên tới 9.9–17.0). |
| `backoff_base_s` / `cap_s` | 0.25 s / 4.0 s | Exponential backoff kèm full jitter giúp phân tán tải retry, triệt tiêu hiện tượng thundering herd đồng pha; trần 4.0 s giữ thời gian chờ vừa phải trong deadline 30 s. |
| `retry_budget_ratio` | 0.1 | Khống chế tổng số retry không vượt quá 10% lượng request mới; giữ `retry_ratio` ở G3 đạt 0.035 (chuẩn <= 0.10) và chặn bùng nổ retry khi xảy ra lỗi diện rộng. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Đủ mẫu thống kê để tính tỷ lệ lỗi chuẩn xác, tránh mở breaker tức thời vì vài lỗi ngẫu nhiên trong giờ thấp điểm (chỉ đánh giá khi có tối thiểu 10 kết quả gần nhất). |
| `breaker.failure_rate_threshold` | 0.5 | Ngắt mạch khi 50% cuộc gọi thất bại; ngắt nhanh dòng request đổ vào provider sập (như G2 khi alpha mất 75% GPU, G6 khi alpha 503) để chuyển hướng sang provider lành lặn. |
| `breaker.slow_call_s` | 5.0 s | Đánh giá cuộc gọi kéo dài >= 5.0 s (gấp hơn 4 lần service median 1.2 s) là dấu hiệu suy thoái nghiêm trọng, bắt bệnh suy thoái hàng đợi trước khi lỗi timeout tích tụ. |
| `breaker.open_duration_s` | 10.0 s | Thời gian provider mở mạch để phục hồi hàng đợi và làm mát tài nguyên; đủ để hạ tải quá tải mà không bỏ quên provider quá lâu. |
| `breaker.half_open_max_probes` | 2 | Chỉ cho phép tối đa 2 request bay đồng thời khi thăm dò trạng thái phục hồi ở HALF-OPEN, ngăn hiện tượng dồn tải làm sập lại provider đang gượng dậy. |

### Sơ đồ quyết định của `on_error`

```
                          [ Nhận ProviderError ]
                                     |
                             classify(err)
                                     |
       +-----------------------------+-----------------------------+
       |                             |                             |
[ CLIENT_ERROR ]             [ RATE_LIMITED ]             [ OVERLOADED /
  (400, 401, 422)             (429 + retry_after)           SERVER_ERROR /
       |                             |                      TIMEOUT ]
  Breaker release               Breaker release                    |
       |                             |                     Breaker on_failure
  GiveUp("client_error")     cooldown = now + retry_after          |
  (Không retry,              Failover sang bậc kế         tries < max_attempts
   không failover)           trong ladder ngay             AND budget cho phép
                                                           AND fits deadline?
                                                              /         \
                                                            (Yes)       (No)
                                                             /             \
                                                    Retry provider cũ    Failover sang
                                                    với Jitter Backoff   bậc kế trong
                                                                         ladder ngay
```

Bảng tổng hợp:

| ErrorKind | Breaker | Cooldown | Retry cùng provider? | Failover? |
|---|---|---|:---:|:---:|
| `CLIENT_ERROR` | `release` (không tính lỗi sức khỏe) | Không | **Không bao giờ** (`GiveUp`) | **Không bao giờ** |
| `QUOTA_EXHAUSTED` | `release` | `now + 300 s` (dài) | Không | **Có** (chuyển ngay) |
| `RATE_LIMITED` | `release` | `now + retry_after` | Không | **Có** (chuyển ngay) |
| `OVERLOADED` / `SERVER_ERROR` / `TIMEOUT` | `on_failure` | Không | **Có** (nếu `tries < 2`, còn budget, còn hạn) | **Có** (nếu hết lượt retry hoặc breaker OPEN) |

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

### 1. Retry budget là lớp phòng thủ cuối
- **Số liệu thí nghiệm:**
  - **G2 (overload storm):**
    - Khi có retry budget: `availability = 0.9992`, `amplification = 1.380`, `retry_ratio = 0.058`, `recovery_s = 0.0 s`, `p95 = 19.239 s`.
    - Khi tắt retry budget: `availability = 0.9704`, `amplification = 1.673`, `retry_ratio = 0.494` (tăng gấp 8.5 lần!), `recovery_s = 8.0 s`, `p95 = 21.886 s`.
  - **G3 (brownout):**
    - Khi có retry budget: `availability = 1.0000`, `retry_ratio = 0.068` (đạt chuẩn <= 0.10).
    - Khi tắt retry budget: `availability = 1.0000`, `retry_ratio = 0.159` (vượt ngưỡng cho phép, fail check).
- **Cơ chế & giải thích:** Khi circuit breaker bị vô hiệu hóa hoặc cấu hình sai ngưỡng (min_calls quá cao), retry budget hoạt động như một van tiết lưu token bucket độc lập chặn đứng bão retry, không cho phép lượng retry vượt quá tỷ lệ 10% request đầu vào. Khi breaker hoạt động chuẩn, retry budget vẫn tối quan trọng vì đây là kiến trúc phòng thủ chiều sâu (defense-in-depth): Breaker ngắt mạch theo sức khỏe từng provider riêng lẻ, trong khi Retry Budget giới hạn lượng tải dội ngược trên quy mô toàn hệ thống, bảo vệ downstream khi nhiều provider cùng suy thoái nhẹ đồng thời.

### 2. `attempt_timeout_s: 30`
- **Số liệu thí nghiệm:**
  - **G6 (provider outage):**
    - `attempt_timeout_s = 8.0 s`: `availability = 1.0000`, `p95 = 5.958 s`, `give_up = {}`.
    - `attempt_timeout_s = 30.0 s`: `availability = 0.9488` (tụt dốc!), `give_up = {'deadline': 107, 'no_provider_available': 15}`, nhưng `p95 = 5.411 s` (giảm ảo!).
  - **G4 (slow region):**
    - `attempt_timeout_s = 8.0 s`: `availability = 1.0000`, `p95 = 5.684 s`.
    - `attempt_timeout_s = 30.0 s`: `availability = 1.0000`, `p95 = 5.608 s`.
- **Cơ chế & giải thích:** Trong G6, provider `alpha-eu` bị treo hoàn toàn; nếu đặt timeout 30 s thì request bị giam cầm suốt 30 s cho đến khi cạn sạch `turn_deadline` của user, dẫn đến 107 request bị hủy (`give_up: deadline`). Do `latency_p95_s` chỉ tính trên các request *thành công*, các request thất bại bị loại khỏi mẫu tính, tạo ra hiện tượng **thiên lệch sống sót (survivorship bias)** khiến p95 trông có vẻ "nhanh hơn" dù hệ thống thực tế sụt giảm nghiêm trọng. G4 không đổi vì provider chỉ chậm thêm 5 s (tổng ~6.2 s) chứ không chết, nên cả timeout 8 s và 30 s đều kịp xử lý. Timeout đúng phải dựa trên thời gian cần thiết để thực hiện ít nhất 2–3 bước failover trong tổng turn deadline và khoảng 3–5 lần latency median lúc bình thường.

### 3. Bỏ lọc tool calling khỏi `ladder()`
- **Số liệu thí nghiệm:** Khi bỏ điều kiện `supports_tools` trên băng G6, số lượng `incompatible_routes` tăng vọt lên **235**, và `availability` tụt từ 1.0000 xuống **0.9015**.
- **Giải thích dưới góc nhìn Product Manager:** "Việc định tuyến tác vụ hoàn tiền sang mô hình không hỗ trợ tool calling (`gamma-mini`) tương đương với việc chuyển máy khách hàng cho một nhân viên tổng đài không có quyền truy cập cổng thanh toán. Hậu quả là 235 giao dịch của khách hàng bị gián đoạn và báo lỗi hệ thống, tỷ lệ phục vụ thành công mất gần 10%, gây tổn hại nghiêm trọng đến lòng tin của người dùng và làm tiêu tốn chi phí vận hành cho những cuộc gọi hoàn toàn vô nghĩa."

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,496,700 cents | 1.123 |
| yours | 1.0 | 66 | **0** | **0** | **0** | **0 cents** | **0.303** |

### Trả lời câu hỏi kiến trúc:
1. **Vì sao chỉ idempotency key là chưa đủ?** Idempotency key bảo đảm bên thứ ba (cổng thanh toán, email) không thực hiện lặp lại tác vụ đã hoàn thành. Tuy nhiên, nếu không có checkpoint, khi worker bị crash ở bước sau (ví dụ crash ở `send_email`), worker mới khởi động lại sẽ phải chạy lại từ bước 1 (`classify_request`, gọi LLM, truy vấn đơn hàng), gây lãng phí tài nguyên và chi phí LLM (`wasted_llm_per_crash` ở naive lên tới 1.123 so với 0.303 của bản có checkpoint).
2. **Vì sao chỉ checkpoint là chưa đủ?** Nếu chỉ có checkpoint mà không có idempotency key, khi worker gặp lỗi mạng sau khi cổng thanh toán đã trừ tiền (`ServiceTimeout` sau commit) hoặc worker bị crash ngay sau khi tiền đã chuyển nhưng trước khi kịp ghi checkpoint, lượt retry hoặc worker phục hồi sẽ gọi lại API thanh toán với một request mới, dẫn đến khách hàng được hoàn tiền 2 lần (như naive đã hoàn thừa $14,967).
3. **Điều gì xảy ra nếu ghi checkpoint TRƯỚC KHI gọi payment?** Nếu worker ghi checkpoint trước khi gọi payment rồi đột ngột crash trước hoặc trong khi gọi API thanh toán, worker khởi động lại thấy checkpoint đã tồn tại nên ngộ nhận bước hoàn tiền đã xong và bỏ qua. Hậu quả là xuất hiện **lost refunds** (khách hàng không nhận được tiền nhưng hệ thống đánh dấu hoàn tất), vi phạm nghiêm trọng tính toàn vẹn dữ liệu tài chính.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (delay 1m) | **page** | 27 min |
| provider_outage | page | page (delay 0m) | **page** | 4 min |
| deploy_blip | none | page (FALSE ALERT) | **none** | N/A (không báo động ảo) |
| slow_leak | ticket | none (BỎ SÓT) | **ticket** | 1337 min (~22.3 h) |
| degraded_evening | page | page (delay 214m) | **page** | 291 min |
| night_spike | none | page (FALSE ALERT) | **none** | N/A (không báo động ảo) |
| small_bump | none | page (FALSE ALERT) | **none** | N/A (không báo động ảo) |

### Báo cáo Error Budget 30 ngày:
- **Tổng số request:** 25,901,755
- **Số lỗi tối đa cho phép (allowed_bad):** 129,508.8
- **Số lỗi thực tế (bad):** 55,618
- **Availability đạt được:** 99.7853% (vượt mục tiêu SLO 99.5000%)
- **Error budget tiêu thụ:** **42.95%**
- **Error budget còn lại:** **57.05%**
- **Quyết định đóng băng release:** **KHÔNG** (`freeze_releases = False`), do ngân sách lỗi vẫn còn hơn 57%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?** Sự cố kéo dài 20 phút với 12% lỗi chỉ tiêu tốn một phần rất nhỏ (< 0.5%) tổng ngân sách lỗi 30 ngày và tự kết thúc nhanh chóng. Việc gửi page đánh thức kỹ sư on-call tạo ra hội chứng mệt mỏi vì cảnh báo (alert fatigue). Cơ chế đa cửa sổ (multi-window) của Google SRE đòi hỏi cả cửa sổ dài (xác nhận mức độ nghiêm trọng) và cửa sổ ngắn (xác nhận lỗi vẫn đang diễn ra), nhờ đó triệt tiêu hoàn toàn false page trên các đợt biến động ngắn này.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Tài liệu đối chiếu: `reports/gameday/G2_overload_storm.png` và `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

### Tóm tắt sự cố (Executive Summary)
Vào giây ảo t=120, deployment chính `alpha` gặp sự cố phần cứng làm mất 75% năng lực tính toán GPU trong vòng 180 giây. Do client phiên bản v1 (`NaiveClient`) sử dụng 3 tầng retry xếp chồng không có jitter, không có retry budget và không có circuit breaker, lượng request retry tăng vọt gấp hơn 25 lần (đạt đỉnh 254 attempts/s), làm nghẽn hoàn toàn hàng đợi FIFO của provider. Sau khi sự cố GPU đã được khắc phục hoàn toàn tại t=300, hệ thống rơi vào trạng thái lỗi siêu bền (metastable failure) và tiếp tục tê liệt thêm 155 giây, khiến tỷ lệ availability của toàn bộ phiên giảm xuống còn 45.60%.

### Ảnh hưởng (Impact)
- **Người dùng:** 1,285 trên tổng số 2,362 lượt người dùng thất bại hoàn toàn (tỷ lệ thành công chỉ đạt 45.60%).
- **Thời gian gián đoạn:** Kéo dài 335 giây (từ t=120 đến t=455), trong đó có 155 giây gián đoạn xảy ra khi hạ tầng provider đã hoạt động bình thường trở lại.
- **Error budget:** Tiêu tốn hơn **108.8%** tổng error budget cho phép của cả tháng nếu áp dụng cho lưu lượng tương đương.
- **Chi phí phát sinh:** Chi phí trung bình mỗi lượt thành công tăng từ 2.13 lên 4.63 đơn vị chi phí do lãng phí tính toán cho các request đã bị hủy.

### Timeline sự cố (Giây ảo từ CSV)

| t (s) | Sự kiện | Bằng chứng định lượng (Số liệu thực tế từ CSV) |
|---:|---|---|
| 0–119 | Trạng thái bình thường | Lưu lượng ổn định 7–15 attempts/s; `calls_ok` đạt 100% (7–15 calls/s). |
| 120 | Provider `alpha` mất 75% GPU | Năng lực xử lý tụt dốc, `calls_ok` giảm ngay lập tức về 2 calls/s. |
| 122 | Bắt đầu mất khả năng phục vụ | `calls_ok = 0`, hàng đợi server bắt đầu ứ đọng request. |
| 136 | Bão retry bùng nổ (Thundering Herd) | Lượng attempt dồn vào `alpha` vọt lên 53 attempts/s, gấp 5 lần bình thường. |
| 143 | Đỉnh điểm bão retry | Lượng attempt đạt đỉnh **254 attempts/s**; 0 cuộc gọi thành công (`calls_ok = 0`). |
| 144–299 | Tê liệt hoàn toàn | Lượng attempts duy trì liên tục ở mức 100–200 attempts/s; 0 request thành công. |
| 300 | `alpha` hồi phục 100% GPU | Năng lực GPU khôi phục hoàn toàn, nhưng `calls_ok` vẫn bằng **0**; attempts vẫn ở mức 115 attempts/s. |
| 301–452 | Trạng thái lỗi siêu bền (Metastable Failure) | Provider đã khỏe nhưng hàng đợi FIFO vẫn kẹt cứng bởi các request quá hạn; client liên tục retry dồn tải giữ hệ thống ở trạng thái chết. |
| 453 | Bắt đầu thoát khỏi bão tải | Hàng đợi cũ được giải tỏa, xuất hiện các lượt thành công đầu tiên (`calls_ok = 4`). |
| 455 | Hệ thống hồi phục hoàn toàn | Tỷ lệ thành công đạt chuẩn trở lại (`calls_ok = 8–16 calls/s`); thời gian hồi phục trễ `recovery_s = 155.0 s`. |

### Trigger vs Root Cause
- **Trigger:** Sự cố sụt giảm đột ngột 75% năng lực GPU của provider `alpha` tại t=120.
- **Root Cause:** Kiến trúc client `NaiveClient` có lỗi nghiêm trọng:
  1. **Nhân bội retry nhiều tầng:** 3 tầng retry chồng chéo (App 4x, Gateway 3x, SDK 3x) tạo ra tới 36 attempts cho mỗi logical call.
  2. **Thiếu Full Jitter:** Các lần retry sử dụng fixed delay, khiến toàn bộ client dồn request vào đúng các mốc thời gian cố định (thundering herd).
  3. **Không có Circuit Breaker:** Không ngắt kết nối khi provider suy thoái, tiếp tục bơm tải vào hàng đợi đang tắc nghẽn.
  4. **Không có Retry Budget:** Không khống chế tỷ lệ retry trên tổng lưu lượng.
- **Vòng lặp tự duy trì (Metastable Failure Loop):** Provider sử dụng hàng đợi FIFO. Khi xử lý chậm, client phía ngoài timeout sau 8 s và gửi request retry mới. Tuy nhiên, server vẫn tiếp tục xử lý các request cũ đã bị client bỏ rơi (wasted work). Lượng request retry mới liên tục đổ về với tốc độ > năng lực xử lý, khiến hàng đợi không bao giờ vơi, duy trì trạng thái quá tải vô tận ngay cả khi GPU đã phục hồi ở t=300.

### Điều gì giúp / Điều gì làm tệ hơn
- **Làm tệ hơn:** Cơ chế timeout ngắn phía client nhưng server không hủy request (không có cancellation propagation); retry mù quáng cùng một provider mà không có fallback ladder.
- **Giúp ích:** Lượng traffic người dùng dừng lại ở cuối kịch bản giúp server có cơ hội xử lý cạn hàng đợi cũ và thoát khỏi vòng lặp.

### Action Items

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|:---:|:---:|---|
| 1 | Thay thế toàn bộ `NaiveClient` bằng `ResilientClient` có Circuit Breaker, Exponential Backoff với Full Jitter và Fallback Ladder | Prevent | Core Agent Team | CI chạy kịch bản G2 đạt `amplification <= 1.2` và `recovery_s = 0.0 s`. |
| 2 | Áp dụng Token-Bucket Retry Budget ở tầng Client Gateway (tỷ lệ tối đa 10% request) | Mitigate | Gateway Team | Unit test `test_budget_caps_retries` đạt chuẩn, `retry_ratio <= 0.10` dưới lỗi 100%. |
| 3 | Triển khai hệ thống giám sát Multi-window Burn-rate Alerting cho availability và circuit breaker state | Detect | SRE Team | Phát hiện sự cố quá tải trong vòng <= 5 phút, 0 false alarm trong 30 ngày. |

---

## 7. Một điểm yếu còn lại

- **Điểm yếu còn lại:** Khi xảy ra sự cố diện rộng khiến toàn bộ các provider trong cùng vùng hoặc cùng hỗ trợ tool đều bị suy thoái (ví dụ cả `alpha` và `beta` đều chạm ngưỡng rate-limit hoặc cùng chậm đồng thời), client của chúng ta sẽ dồn tải fallback sang `beta`. Mô hình `beta` có chi phí gọi cao hơn 40% (`cost_per_call = 1.4` so với 1.0 của `alpha`) và rate limit tenant thấp hơn (16 rps so với 20 rps của `alpha`). Việc dồn tải ồ ạt sang `beta` có thể nhanh chóng làm nghẽn `beta`, gây tăng đột biến chi phí (`cost_per_turn`) và đẩy p95 lên cao.
- **Cách khắc phục:** 
  1. Triển khai cơ chế **Client-side Concurrency Limiting (Adaptive Concurrency Limits / CoDel)** nhằm tự động điều chỉnh số lượng request đồng thời gửi tới từng provider dựa trên xu hướng RTT thực tế, ngăn ngừa việc làm quá tải provider dự phòng.
  2. Áp dụng kỹ thuật **Hedging Requests có kiểm soát** (chỉ hedge sau khi P95 latency của attempt 1 vượt ngưỡng cho phép) thay vì failover tuần tự, giúp tối ưu p95 mà không làm bùng nổ chi phí.
  3. Bổ sung tính năng **Graceful Degradation**: Tự động chuyển đổi tạm thời các câu hỏi phức tạp sang chế độ tổng hợp nhẹ hoặc sử dụng mô hình rẻ hơn (`gamma-mini`) đối với các lượt không bắt buộc dùng tool khi chi phí vận hành chạm ngưỡng ngân sách khẩn cấp.

---

## 8. Cache (Part G)

*Số liệu từ `reports/cache/*.json` (`make cache`):*

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9124** | **0.9645** | **0.652** | **157** | **0** | **3** | **0.915** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.92 | Tại 0.80 có tới 157 câu trả lời sai (`wrong_answers = 157` do `other_question`). Đặt 0.92 giúp triệt tiêu hoàn toàn câu trả lời sai (`wrong_answers = 0`) trong khi vẫn giữ `cache_hit_ratio` đạt 65.2% (vượt xa chuẩn >= 50%). Nếu đặt 0.97, hit ratio tụt xuống 58.1% làm tăng chi phí gọi model. |
| `ttl_s` | 60.0 s | Khi có thay đổi bảng giá vận chuyển không báo trước ở t=480, TTL 60 s khống chế số câu trả lời cũ chỉ ở mức **3** (đạt chuẩn <= 15). Nếu để TTL 300 s như starter, số câu cũ vọt lên 95 (vi phạm nghiêm trọng). |
| `max_stale_s` | 300.0 s | Đủ dài để bao trọn sự cố toàn bộ provider sập 120 s (t=240–360), phục vụ an toàn 157 lượt stale cho câu hỏi FAQ, nâng `faq_availability` từ 87.33% lên 96.45% (chuẩn >= 93%). |

### Kết quả các thí nghiệm Cache:

1. **Ngưỡng similarity:**
   - `similarity_threshold = 0.80`: `cache_hit_ratio = 0.666`, `wrong_answers = 157` (`other_question: 157`), `cost = 0.965`, `faq_avail = 0.8733`. Các câu hỏi có từ khóa gần giống nhau nhưng ngữ nghĩa khác nhau (ví dụ: hỏi về "thời gian hoàn tiền" bị nhầm sang "chính sách điều kiện hoàn tiền") bị nhận diện sai.
   - `similarity_threshold = 0.92` (tối ưu): `cache_hit_ratio = 0.652`, `wrong_answers = 0`, `cost = 0.915`, `faq_avail = 0.9645`.
   - `similarity_threshold = 0.97`: `cache_hit_ratio = 0.581`, `wrong_answers = 0`, `cost = 0.964`, `faq_avail = 0.9640`. Ngưỡng quá cao làm giảm tỷ lệ trúng cache, tăng số lần phải gọi model.

2. **TTL:**
   - `ttl_s = 30.0 s`: `outdated_answers = 2`, `cache_hit_ratio = 0.592`, `cost = 0.955`.
   - `ttl_s = 60.0 s`: `outdated_answers = 3` (đạt yêu cầu <= 15), `cache_hit_ratio = 0.652`, `cost = 0.915`.
   - `ttl_s = 300.0 s`: `outdated_answers = 95` (fail nặng), `cache_hit_ratio = 0.731`, `cost = 0.863`.
   - *Giải thích:* Webhook chỉ kích hoạt khi hệ thống quản trị chủ động gửi thông báo cập nhật (như sự kiện ở t=150). Khi bên thứ ba âm thầm thay đổi bảng giá cước ở t=480 mà không có thông báo, không có webhook nào được kích hoạt để gọi `invalidate()`. Khi đó, TTL là cơ chế duy nhất giúp cache tự làm mới dữ liệu.

3. **Stale-if-error:**
   - `max_stale_s = 0.0 s`: `faq_availability = 0.8733` (fail check >= 0.93), `stale_served = 0`, `availability = 0.8565`.
   - `max_stale_s = 300.0 s`: `faq_availability = 0.9645` (PASS), `stale_served = 157`, `availability = 0.9124`. Khi mọi provider cùng sập, việc phục vụ câu trả lời cũ cho câu hỏi FAQ chung cứu vãn trải nghiệm người dùng và giúp hệ thống đạt chỉ tiêu.

### Trả lời câu hỏi:
- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  1. *Nguy cơ rò rỉ bảo mật (Data Leakage):* Không gian vector embedding không hoàn toàn phân biệt được user ID; hai câu hỏi của hai user khác nhau nếu vô tình vượt ngưỡng cosine sẽ làm lộ dữ liệu đơn hàng nhạy cảm của người này cho người khác.
  2. *Dữ liệu biến động theo thời gian thực:* Trạng thái đơn hàng, số tiền hoàn, điểm thưởng thay đổi liên tục. Nếu cache (dù theo user), khách hàng sẽ nhận được thông tin cũ sai lệch (ví dụ đơn đã giao nhưng cache vẫn báo đang đóng gói), dẫn đến khiếu nại dịch vụ.
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  1. **Tỷ lệ phục vụ stale (`stale_served_ratio` hoặc `stale_served_count`):** Cảnh báo ngay khi số lượng response `stale=True` tăng đột biến.
  2. **Upstream Provider Error Rate & Timeout Rate:** Đo lường tỷ lệ lỗi thực tế từ các lệnh gọi LLM trực tiếp lên provider (429, 503, 529, client timeout) thay vì chỉ đo availability ở tầng gateway khách hàng.
  3. **Circuit Breaker State:** Cảnh báo page ngay khi Circuit Breaker của primary provider chuyển trạng thái sang `OPEN`.
