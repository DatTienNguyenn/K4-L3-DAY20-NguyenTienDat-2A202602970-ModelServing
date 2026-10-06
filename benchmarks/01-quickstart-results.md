# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3172 | 432 / 654 | 43.6 / 46.4 | 2954 / 3559 / 3559 | 22.9 |
| UD-Q2_K_XL | 0.39 | 4215 | 612 / 808 | 51.6 / 57.0 | 3797 / 4222 / 4222 | 19.4 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.18x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

### 1. So sánh định lượng (Metrics & Latency)
- **Kích thước (Size):** Bản `UD-Q2_K_XL` (0.39 GB) nhỏ hơn bản `Q4_K_M` (0.50 GB) khoảng 0.11 GB (~22% reduction).
- **TTFT (Time to First Token - Prefill):** 
  - `Q4_K_M`: P50 = 432.4 ms, P95 = 654.2 ms.
  - `UD-Q2_K_XL`: P50 = 611.7 ms, P95 = 807.5 ms.
  - Prefill của bản 2-bit chậm hơn đáng kể (~1.41x ở P50).
- **TPOT (Time Per Output Token - Decode):**
  - `Q4_K_M`: P50 = 43.6 ms (22.9 tok/s).
  - `UD-Q2_K_XL`: P50 = 51.6 ms (19.4 tok/s).
  - Tốc độ decode của bản 2-bit chậm hơn **1.18x** so với bản 4-bit.
- **Thời gian nạp model (Load Time):** `Q4_K_M` nạp trong 3172 ms, trong khi `UD-Q2_K_XL` mất 4215 ms (lâu hơn do chi phí phân rã định dạng động).

### 2. Giải thích cơ chế (Bottleneck Mechanism)
- Việc giảm số bit (quantization sâu hơn) thường chỉ giúp tăng tốc decode khi hệ thống bị nghẽn ở **băng thông bộ nhớ (memory-bandwidth bound)** — tức khi thời gian nạp trọng số từ RAM chiếm phần lớn thời gian chu kỳ.
- Trên máy tính thử nghiệm (CPU Intel Core i7-1260P, chạy thuần CPU `ngl=0`, không offload GPU):
  - Model `Qwen3.5 0.8B` có kích thước rất nhỏ (~0.5 GB), hoàn toàn nằm gọn trong bộ nhớ đệm và RAM tốc độ cao, không gặp tình trạng nghẽn băng thông.
  - Ngược lại, máy tính rơi vào trạng thái **bị giới hạn bởi năng lực tính toán của CPU (compute-bound)**.
  - Định dạng lượng tử hóa sâu `UD-Q2_K_XL` đòi hỏi nhiều phép tính giải lượng tử hóa (dequantization / dynamic scale-offset unpacking) trên mỗi khối trọng số phức tạp hơn rất nhiều so với phép giải lượng tử hóa chuẩn của `Q4_K_M`.
  - Do CPU phải gánh thêm chi phí tính toán giải nén này mà không bù lại được lợi ích tiết kiệm I/O, tốc độ decode thực tế bị giảm từ 22.9 tok/s xuống 19.4 tok/s.

### 3. Đánh giá chất lượng sinh văn bản (Output Quality)
Khi thử nghiệm hỏi cùng một câu hỏi kỹ thuật (*"Explain briefly why LLM prefill is compute-bound while decode is memory-bandwidth bound"*):
- **Bản 4-bit (`Q4_K_M`):** Câu trả lời mạch lạc, đúng trọng tâm kỹ thuật, phân tích rõ ràng hai giai đoạn và cấu trúc ngữ pháp hoàn chỉnh.
- **Bản 2-bit (`UD-Q2_K_XL`):** Câu trả lời bắt đầu xuất hiện câu từ lủng củng ("The Prefill is primarily a computable process"), cấu trúc câu nông và suy luận logic bị suy giảm rõ rệt. Với mô hình siêu nhỏ (0.8B tham số), việc ép xuống 2-bit gây thất thoát thông tin trọng số quá lớn.

### 4. Kết luận: Có đáng dùng không?
**HOÀN TOÀN KHÔNG ĐÁNG DÙNG** trên hệ thống này:
- Bản 2-bit **chậm hơn 1.18x** thay vì nhanh hơn.
- Chất lượng câu trả lời bị suy giảm nghiêm trọng.
- Mức dung lượng tiết kiệm được (~110 MB) không có ý nghĩa thực tế khi máy đã có 8 GB RAM.
=> Khuyến nghị: Giữ nguyên bản **`Q4_K_M`** làm baseline serving.
