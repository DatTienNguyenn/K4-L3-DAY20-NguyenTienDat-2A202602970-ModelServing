# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 29 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5808 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

### 1. Bằng chứng về Continuous Batching (Peak Batch Width)
- **Đỉnh quan sát được:** `n_busy_slots_per_decode` đạt **4.00 trên tổng số 4 slots (100% capacity)**. Đồng thời chỉ số `requests_processing` cũng đạt mức tối đa là **4**.
- **Ý nghĩa:** Điều này là bằng chứng rõ ràng cho thấy cơ chế **Continuous Batching (iteration-level batching)** của `llama-server` đang hoạt động hoàn hảo. Scheduler đã gộp đồng thời 4 requests khác nhau vào cùng một bước tính toán decode ma trận thay vì xử lý tuần tự (FIFO) từng request.

### 2. So sánh giữa Peak Batch Width (4.00) và Effective Concurrency (14.8)
- Trong `02-server-results.md`, theo Định luật Little ($L = \lambda \times W$), giá trị **Effective Concurrency là 14.8** (tỉ lệ occupancy/slot đạt **3.69x** so với 4 slots).
- Hai con số này **không khớp nhau về mặt giá trị (14.8 vs 4.00)**, nhưng **hoàn toàn nhất quán về mặt bản chất hệ thống**:
  - **`n_busy_slots_per_decode` (4.00):** Đo lường **tài nguyên thực thi vật lý (Active Compute Slots)** bên trong server. Server chỉ có cấu hình `--parallel 4`, nên số request được decode đồng thời tại một thời điểm không bao giờ vượt quá 4.
  - **`Effective Concurrency` (14.8):** Đo lường **tổng số request đang tồn đọng trong toàn hệ thống (In-Flight System Occupancy = In-Processing + In-Queue)** từ góc nhìn của client (Locust).

### 3. Con số nào đáng tin cậy hơn và tại sao?
- **Cả hai con số đều đáng tin cậy nhưng đại diện cho hai tầng khác nhau:**
  - **Để biết năng lực tính toán và mức độ bận rộn của server:** Tin cậy **`n_busy_slots_per_decode`** (= 4.00). Nó chứng minh toàn bộ 4 slots compute đều đã bão hòa 100% công suất.
  - **Để phân tích hàng đợi và trải nghiệm người dùng:** Tin cậy **`Effective Concurrency`** (= 14.8) kết hợp với chỉ số **`requests_deferred` (= 46)** từ metrics server.
- **Kết luận:** Khoảng chênh lệch giữa 14.8 và 4.00 chính là số lượng request đang bị **nghẽn trong hàng đợi (Queue Backlog)**. Khi 4 slot đã đầy, 46 request đến sau phải chờ ở trạng thái `requests_deferred`. Đây chính là nguyên nhân giải thích vì sao độ trễ P95 bị đội lên tới **52.000 ms (52 giây)** — phần lớn thời gian người dùng phải chờ là **queue time** chứ không phải thời gian tính toán thực tế.
