# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 9389.4 | 9390.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5886.4 | 5886.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 5045.4 | 5045.5 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6773.7** · total **6774.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it specifically addresses the issue of **saturation**.

Here is the breakdown based on the text:

*   **Raw throughput** ignores SLOs (Service Level Objectives) and assumes the system is operating at full capacity ("throughput at saturation ignores SLOs").
*   **Goodput** counts only requests per second that met 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

Specifically, it addresses the issue where the KV cache is stored in non-contiguous pages. This layout causes wasted memory because the data is scattered across different physical memory blocks rather than being tightly packed into a single contiguous block. By storing the cache in non-contiguous pages, PagedAttention 

**When does splitting prefill and decode help?**

> Based on the context provided, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

The context explicitly states that prefill is compute-bound (requiring significant processing power) and that decoding is memory-bandwidth-bound (requiring significant data transfer). By splitting these operations into separate pools, the system allows the engine to skip


## Which N16-N19 pieces are real

### 1. Phân loại các thành phần N16–N19 (Real vs Stub)
Theo đúng kiến trúc đang thực thi trong script `pipeline.py`:
- **N16 (Cloud/IaC):** **`stub`** — Toàn bộ pipeline chạy cục bộ (local) trên laptop, không triển khai tài nguyên hạ tầng đám mây (AWS/GCP/Terraform).
- **N17 (Data pipeline):** **`stub`** — Dữ liệu context được định nghĩa sẵn trong mảng bộ nhớ `TOY_DOCS`, không có luồng thu thập hay ETL pipeline tự động.
- **N18 (Lakehouse):** **`stub`** — Dữ liệu lưu trong cấu trúc Python in-memory list/dict, không lưu trữ trên Lakehouse thật (như Delta Lake hay DuckDB/Iceberg).
- **N19 (Vector index & features):** **`stub`** — Quá trình retrieval sử dụng đối sánh từ khóa đơn giản (`keyword overlap`) trên RAM, thời gian embed = 0.0 ms và retrieve = 0.1 ms, không dùng model embedding hay cơ sở dữ liệu vector thật (Milvus/Qdrant/FAISS).
*(Lưu ý: Chỉ có **N20 Serving** là **`real`**, kết nối trực tiếp qua HTTP tới `llama-server` đang chạy mô hình `Qwen3.5 0.8B`).*

### 2. Đánh giá Stage chiếm nhiều thời gian nhất (Dominant Stage)
- **Số liệu đo đạc:**
  - `embed`: **0.0 ms**
  - `retrieve`: **0.1 ms**
  - `llm`: **6,773.7 ms** (chiếm **100%** tổng thời gian pipeline 6,774.2 ms).
- **Có đúng như kỳ vọng không?**
  - **Hoàn toàn đúng như kỳ vọng.** Các stage embed và retrieve là code stub chạy tức thì trong CPU cache trên một tập dữ liệu đồ chơi siêu nhỏ (6 documents), chỉ mất 0.1 ms. Ngược lại, stage `llm` phải nạp prompt, xử lý prefill context và thực hiện vòng lặp sinh token tuần tự (autoregressive decode) trên CPU (~20-40 tok/s). Kể cả khi triển khai Vector DB thật ở N19 (với độ trễ ~10–50 ms), stage `llm` với vài giây decode vẫn luôn là nút thắt áp đảo trong toàn bộ RAG pipeline.

### 3. Chiến lược giảm 2x độ trễ (Halving Pipeline Latency)
- **Stage cần tấn công:** Bắt buộc phải tấn công vào **stage `llm`** (theo Định luật Amdahl — Amdahl's Law). 
  - Do `llm` chiếm trọn 100% thời gian (6.77s / 6.77s), việc tối ưu embed hay retrieve về 0 ms cũng chỉ tiết kiệm được tối đa 0.1 ms (0.001%). Chỉ có can thiệp vào `llm` mới có khả năng giảm độ trễ pipeline xuống một nửa (~3.4 giây).
- **Các biện pháp cụ thể để đạt mục tiêu giảm 2x:**
  1. **Giới hạn số token đầu ra (Output Length Budgeting / Early Stopping):** Decode latency phụ thuộc tuyến tính vào số output tokens ($Latency \approx TTFT + n_{out} \times TPOT$). Bằng cách tinh chỉnh system prompt yêu cầu trả lời ngắn gọn, súc tích (ví dụ giới hạn trong 30-50 tokens thay vì để mô hình sinh dài dòng), ta có thể giảm ngay 50% thời gian decode.
  2. **Tối ưu cấu hình thực thi CPU (`threads=4`):** Áp dụng kết quả từ tuning sweep (`01-tuning-tg128.md`), thiết lập chạy 4 luồng trên 4 nhân P-cores vật lý để tăng tốc độ sinh token lên 40.6 tok/s thay vì để mặc định.
  3. **Offload GPU acceleration (`ngl`):** Nếu có GPU rời (hoặc CUDA/Metal), đẩy một phần hoặc toàn bộ model weights lên GPU để đẩy throughput decode từ ~23 tok/s lên >60-80 tok/s.
  4. **Prompt Caching / Prefix Caching (RadixAttention):** Tận dụng bộ đệm KV cache cho system prompt và các context tài liệu dùng chung, giúp giai đoạn prefill (TTFT) giảm gần như về 0 ms cho các lượt truy vấn tiếp theo.
