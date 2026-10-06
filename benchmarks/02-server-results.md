# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 35 | 0.60 | 14000 | 23000 | 24000 | 8.2 | 0.0% |
| 50 | 33 | 0.57 | 21000 | 52000 | 57000 | 14.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.94x** (19% of linear) |
| P95 latency | **2.26x** |
| Effective concurrency at 50 users | 14.8 vs `--parallel 4` slots (occupancy/slot ratio 3.69) |

**Saturated.** Throughput delivered only 0.94x for 5x the offered load, and effective concurrency (14.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.94x while P95 moved 2.26x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

### 1. Vị trí bão hoà của Server (Saturation Point)
- **Hệ thống đã bão hoà ngay từ mức tải 10 users (hoặc thấp hơn):**
  - Cấu hình server chỉ có **`--parallel 4`** slots xử lý đồng thời.
  - Ngay ở mức **10 users**, Effective Concurrency theo Định luật Little ($L = \lambda \times W$) đã đạt **8.2** — tức là trung bình luôn có 8.2 requests tồn tại trong hệ thống, vượt gấp **2.05 lần** năng lực phục vụ vật lý của 4 slots.
  - Khi tăng tải lên **50 users** (offered load tăng 5x), Effective Concurrency đạt **14.8** (gấp **3.69 lần** số slot), trong khi số request xếp hàng chờ trong hàng đợi lên đến **`requests_deferred = 46`**.

### 2. Các con số chứng minh hiện tượng bão hoà (Evidence)
Những con số định lượng thuyết phục nhất bao gồm:
1. **Thông lượng (Throughput/RPS) đi ngang và giảm nhẹ (Plateau):**
   - 10 users: **0.60 req/s** (35 reqs / 60s).
   - 50 users: **0.57 req/s** (33 reqs / 60s), chỉ đạt **0.94x** thông lượng ban đầu (19% so với kỳ vọng tăng tuyến tính). Việc tăng tải không đem lại thêm bất kỳ throughput nào.
2. **Độ trễ P95 bùng nổ (Latency Explosion):**
   - P95 tăng vọt từ **23.000 ms (23s) lên 52.000 ms (52s) — tăng 2.26x**.
   - P50 tăng từ 14.000 ms lên 21.000 ms.
   - Khoảng chênh lệch khổng lồ giữa P50 và P95 chứng minh rằng 100% tải tăng thêm sau điểm bão hòa đã biến thành **thời gian chờ trong hàng đợi (Queue Time)** thay vì thời gian tính toán hữu ích.

### 3. Lập luận về Goodput và SLO
- Giả định đặt mục tiêu **SLO: P95 latency ≤ 25.000 ms (25 giây)**:
  - **Ở 10 users:** P95 đạt 23.000 ms (< 25s), thỏa mãn SLO. Do đó, toàn bộ throughput 0.60 RPS đều là thông lượng hữu ích (**Goodput = 0.60 req/s, đạt 100%**).
  - **Ở 50 users:** P95 phình to lên 52.000 ms (vượt quá gấp đôi ngưỡng SLO). Đại đa số request hoàn thành quá muộn so với cam kết chất lượng dịch vụ => **Goodput sụp đổ gần như về 0 req/s** mặc dù phần cứng vẫn tiêu thụ 100% tài nguyên CPU.

### 4. Knob can thiệp đầu tiên để nâng Goodput và lý do
- **Knob can thiệp đầu tiên:** Giảm số luồng tính toán từ **`threads=8` xuống `threads=4` (`LAB_N_THREADS=4`)**.
- **Giải thích lý do chọn knob này thay vì knob khác:**
  - **Dựa trên kết quả đo thread tuning (`01-tuning-tg128.md`):** Do đặc thù CPU Intel Gen 12th i7-1260P có 4 nhân P-cores và 8 nhân E-cores, việc chạy 8 threads làm phát sinh hiệu ứng nghẽn rào cản (*straggler effect*) trên các nhân E-cores chậm hơn. Giảm về **4 threads** giúp tăng tốc độ sinh token từ **38.8 lên 40.6 tok/s (tăng 5% throughput thuần)**.
  - **Cơ chế giảm hàng đợi:** Khi mỗi request được decode nhanh hơn, thời gian chiếm dụng slot (service time $W_s$) giảm xuống, giúp 4 slots giải phóng nhanh hơn, trực tiếp giải tỏa hàng đợi `requests_deferred` và kéo độ trễ P95 giảm xuống dưới ngưỡng SLO 25s.
  - **Tại sao không tăng `--parallel` (ví dụ lên 8)?** Nếu tăng slot `--parallel` trong khi tài nguyên CPU/Memory Bandwidth không đổi, việc decode đồng thời 8 sequences sẽ làm tăng mạnh độ trễ trên từng token (TPOT tăng cao), khiến thời gian xử lý mỗi request cá nhân kéo dài hơn, đẩy P95 lên cao hơn nữa.
  - **Tại sao không giảm quantization xuống 2-bit?** Như kết quả baseline đã chỉ ra, bản 2-bit trên CPU này bị overhead giải mã làm chậm hơn 1.18x và suy giảm chất lượng, không giúp ích cho SLO.
=> Do đó, tối ưu luồng tính toán (`threads=4`) kết hợp thiết lập cơ chế giới hạn tiếp nhận (admission control / rate limiting ở mức ~8-10 concurrency) là giải pháp tối ưu nhất để bảo vệ Goodput.
