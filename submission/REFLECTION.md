# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** _Nguyễn Việt Trung_
**Cohort:** _A20-K1_
**Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** macOS Darwin 25.0.0 (arm64)
- **CPU:** Apple M1 Pro
- **Cores:** 10 physical / 10 logical
- **CPU extensions:** ARM NEON (Apple Silicon)
- **RAM:** 16.0 GB
- **Accelerator:** Apple Metal (Apple Silicon unified memory)
- **llama.cpp backend đã chọn:** Metal (`-DGGML_METAL=on`)
- **Recommended model tier:** Llama-3.2-3B-Instruct (Q4_K_M)

**Setup story:** Môi trường macOS arm64 với Python 3.11 từ Homebrew. Vấn đề duy nhất: virtualenv ban đầu được tạo với Python 3.9, dẫn đến `llama_cpp` không import được khi chạy `python`. Fix bằng cách rebuild venv với Python 3.11 và cài lại `llama-cpp-python` với `CMAKE_ARGS="-DGGML_METAL=on"`. Dùng `llama-server` từ Homebrew thay vì build từ source vì đã có Metal backend sẵn.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

Settings: `n_threads=10`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=99`.

| Model                             | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) |  E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
| --------------------------------- | --------: | ----------------: | ----------------: | --------------------: | ------------------: |
| Llama-3.2-3B-Instruct-Q4_K_M.gguf |    12,017 |          68 / 175 |       18.6 / 18.8 | 1,244 / 1,343 / 1,354 |                53.7 |
| Llama-3.2-3B-Instruct-IQ3_M.gguf  |     1,109 |          70 / 105 |       18.2 / 18.3 | 1,217 / 1,258 / 1,276 |                54.9 |

**Quan sát:** TPOT gần như bằng nhau (18.6 vs 18.2 ms) giữa Q4_K_M và IQ3_M — dấu hiệu điển hình của memory-bandwidth bottleneck trên Metal: decode speed bị giới hạn bởi băng thông unified memory của M1 Pro, không phải bởi kích thước weight. Sự khác biệt đáng kể nhất là load time (12,017 ms vs 1,109 ms — chênh 11×) và TTFT P95 (175 ms vs 105 ms). IQ3_M tiết kiệm 0.4 GB RAM và load nhanh hơn nhiều, với đánh đổi chính là chất lượng output, không phải latency.

---

## 3. Track 02 — llama-server load test

Server config: `llama-server -ngl 99 --parallel 4 --cont-batching --metrics --ctx-size 4096`

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
| ----------: | --------: | ------------: | -----------: | -----------: | -------: |
|          10 |      1.08 |         7,500 |       11,000 |       13,000 |        0 |
|          50 |      0.95 |        21,000 |       39,000 |       40,000 |        0 |

**KV-cache observation:** Metric `llamacpp:kv_cache_usage_ratio` không có trong Homebrew build (v9020). Dùng proxy metric `llamacpp:n_busy_slots_per_decode = 3.74` dưới 10-user load — tức **93.5% slot utilization** (4 slots được cấu hình). Khi tăng lên 50 users, median latency tăng từ 7.5s lên 21s vì toàn bộ 4 slots bão hòa và request phải queue. RPS gần như không tăng (1.08 → 0.95) — bottleneck là GPU compute, không phải network hay concurrency overhead.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub — localhost only, không kết nối cluster
- **N17 (Data pipeline):** stub — in-memory list thay Airflow DAG
- **N18 (Lakehouse):** stub — `TOY_DOCS` Python list thay Delta Lake / Iceberg
- **N19 (Vector + Feature Store):** stub — keyword overlap scoring thay embedding index (Qdrant/Feast)

**Nơi tốn nhiều ms nhất** trong pipeline:

- embed/retrieve: ~0.0 ms (toy keyword search trong memory, không có I/O)
- llama-server: 1,003 – 3,429 ms (toàn bộ thời gian nằm ở đây)

**Reflection:** Bottleneck hoàn toàn nằm ở LLM inference, đúng kỳ vọng. Retrieval 0ms vì dùng toy in-memory search — trong production với vector DB thực (Qdrant, Pinecone) retrieval sẽ tốn 10–100ms nhưng vẫn nhỏ hơn nhiều so với LLM call. Query đầu tiên (goodput vs throughput) trả lời sai vì keyword overlap giữa query và TOY_DOCS thấp — retrieval trả về wrong contexts, minh họa rõ lý do cần embedding-based retrieval trong production.

---

## 5. Bonus — The single change that mattered most

**Change:** Thread count sweep với Metal backend (`-ngl 99`) — test `-t 1, 2, 5, 10, 20` trên Llama-3.2-3B-Instruct Q4_K_M.

**Before vs after** (từ `benchmarks/bonus-thread-sweep.md`):

```
t=  1  →  54.6 tok/s
t=  2  →  54.7 tok/s
t=  5  →  50.9 tok/s
t= 10  →  53.9 tok/s
t= 20  →  54.7 tok/s
```

**Tại sao kết quả flat — và đây là insight quan trọng:**

Curve hoàn toàn flat (~54 tok/s bất kể 1 hay 20 threads) vì với `-ngl 99`, **toàn bộ model được offload lên Metal GPU**. CPU threads chỉ xử lý orchestration overhead — memory copy, tokenization, sampling — trong khi 99% compute (matrix multiply trên attention layers) chạy trên GPU cores của M1 Pro. Băng thông bộ nhớ GPU (unified memory ~200 GB/s trên M1 Pro) là ceiling thực sự, không phải số lượng CPU threads.

Điều này khác với CPU-only setup nơi thread count sweep thường cho bell curve rõ ràng: tăng đến physical core count thì peak, rồi drop khi hyperthreads tranh giành memory bandwidth. Trên Metal, tuning knob quan trọng là số GPU layers (`-ngl`), không phải `-t`. Kết quả này khớp với deck §3: **hardware-matched kernels** (Metal trên Apple Silicon) loại bỏ hoàn toàn CPU-side bottleneck, và "production tuning" trên laptop Apple Silicon nên tập trung vào quantization và context size thay vì thread count.

---

## 6. Điều ngạc nhiên nhất

Thread sweep hoàn toàn flat khi dùng Metal — ban đầu tôi kỳ vọng sẽ thấy bell curve như deck mô tả, nhưng hóa ra CPU threads irrelevant khi GPU đã handle toàn bộ compute. Điều này làm rõ rằng "tuning" phụ thuộc rất nhiều vào hardware path — knob quan trọng trên CPU-only machine (thread count) hoàn toàn vô nghĩa trên GPU-accelerated machine.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` đã commit
- [x] `benchmarks/bonus-thread-sweep.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
