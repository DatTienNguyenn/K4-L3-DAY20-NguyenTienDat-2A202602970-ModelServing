# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Tiến Đạt
**MSSV:** 2A202602970
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux (Ubuntu 24.04 trên WSL2 Linux 6.6.87.2-microsoft-standard-WSL2 x86_64)
- **CPU:** 12th Gen Intel(R) Core(TM) i7-1260P
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 7.6 GB
- **Accelerator:** NVIDIA GeForce MX570, 2048 MiB (CUDA backend / CPU runtime)
- **llama.cpp asset đã tải:** llama.cpp b10488
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Máy tính có 7.6 GB RAM (< 8 GB tối thiểu của Gemma 4 E2B), do đó tôi đã chọn model nhẹ hơn bằng cách xuất `LAB_MODEL=qwen35-0.8b` trước khi chạy `make setup`. Nhờ vậy đã tải thành công runtime prebuilt và bộ trọng số Qwen3.5 0.8B (~0.9 GB) nhanh chóng, không bị lỗi tràn bộ nhớ (OOM).

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3172 | 432 / 654 | 43.6 / 46.4 | 2954 / 3559 / 3559 | 22.9 |
| UD-Q2_K_XL | 0.39 | 4215 | 612 / 808 | 51.6 / 57.0 | 3797 / 4222 / 4222 | 19.4 |

**Quan sát** (≤ 60 chữ): Bản 2-bit chậm hơn 1.18x (19.4 vs 22.9 tok/s), TTFT và TPOT đều tăng, không đáng dùng. Hỏi cùng một câu, bản 4-bit trả lời mạch lạc, đúng trọng tâm; bản 2-bit câu từ lủng củng, suy luận nông do thất thoát thông tin.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.60 | 14000 | 23000 | 24000 | 8.2 | 0.0% |
| 50 | 0.57 | 21000 | 52000 | 57000 | 14.8 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.94×
- **P95 tăng:** 2.26×
- **Effective concurrency ở 50 users:** 14.8 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 4.00 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa ngay từ 10 users: khi tăng tải 5x, RPS đi ngang/giảm nhẹ (0.94x) trong khi P95 phình 2.26x lên 52s. Độ trễ tăng thêm 100% là queue time vì 4 slots đều bận (requests_deferred=46, effective concurrency=14.8 vs 4 slots). Để nâng goodput@SLO 25s, tôi đổi threads=4 trước để tăng tốc độ decode, giải phóng slot nhanh hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local WSL2 machine | stub |
| N17 Data pipeline | In-memory TOY_DOCS list | stub |
| N18 Lakehouse | Python dict/list in RAM | stub |
| N19 Vector + features | Keyword overlap search | stub |
| N20 Serving | llama-server on :8080 | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 6773.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total 6774.2 ms)

**Reflection** (≤ 60 chữ): Bottleneck nằm hoàn toàn ở LLM decode (100% thời gian), đúng như kỳ vọng vì retrieval là stub in-memory còn LLM chạy mô hình ngôn ngữ trên CPU. Muốn giảm 2x latency, phải tấn công vào LLM: giới hạn max_tokens đầu ra ngắn gọn và chạy với threads=4.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Hạ số luồng từ 16 (toàn bộ logical cores) xuống 4 (chỉ dùng các P-cores vật lý)

```
before:  16.0 tok/s
after:   40.6 tok/s
speedup: 2.54×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

CPU Intel Core i7-1260P là kiến trúc lai Alder Lake gồm 4 nhân P-cores (hiệu năng cao, IPC lớn, xung boost cao) và 8 nhân E-cores (tiết kiệm điện). Khi chạy 16 luồng (-t 16), các worker threads bị phân bổ vào cả các luồng Hyper-Threading và E-cores, gây ra hiện tượng tranh chấp bộ đệm (cache thrashing), nghẽn bus bộ nhớ và đặc biệt là straggler effect (các P-cores phải chờ E-core chậm nhất trong barrier synchronization của ma trận).

Khi hạ xuống 4 luồng (-t 4), hệ điều hành gán đúng 4 threads vào 4 nhân P-cores vật lý, tận dụng tối đa AVX2 và cache riêng mà không gặp xung đột luồng hay phải chờ các nhân E-cores chậm hơn. Điều này giúp tốc độ sinh token tăng vọt từ 16.0 lên 40.6 tok/s (tăng tốc 2.54 lần).

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

_(Công cụ nào, dùng vào việc gì. Ghi "Không dùng" nếu không dùng.)_
