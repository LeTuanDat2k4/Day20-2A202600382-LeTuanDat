# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Lê Tuấn Đạt
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 11
- **CPU:** Intel i7-12650H
- **Cores:** 10 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.6 GB
- **Accelerator:** NVIDIA RTX 3070 Laptop GPU 8GB
- **llama.cpp backend đã chọn:** CUDA
- **Recommended model tier:** Qwen2.5-1.5B

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Phải cài đặt gói `llama-cpp-python[server]` riêng cho môi trường ảo `venv` và sửa lỗi parsing dấu ngoặc kép trên PowerShell script `start-server.ps1` để đọc được `active.json`.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| qwen2.5-1.5b-instruct-q4_k_m | 966 | 384 / 591 | 93.2 / 107.1 | 6259 / 7324 / 7581 | 10.7 |
| qwen2.5-1.5b-instruct-q2_k   | 421 | 387 / 413 | 89.2 / 91.1  | 5968 / 6086 / 6121 | 11.2 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Q4_K_M chậm hơn một chút so với Q2_K (10.7 vs 11.2 tok/s) nhưng độ trễ P95/P99 dao động nhiều hơn. Tuy nhiên với 16GB RAM, chất lượng văn bản tốt hơn của Q4 hoàn toàn xứng đáng để đánh đổi.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 1.2 | 1200 | 8500 | 9200 | 0 |
| 50 | 0.9 | 3500 | 18000 | 22000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _Không đo được_, nghĩa là …

Do script server chạy qua wrapper Python trên hệ điều hành Windows nên không hỗ trợ endpoint Prometheus `/metrics`. Tuy nhiên dựa vào cấu hình n_ctx=2048, VRAM của RTX 3070 khá thoải mái để chứa KV cache.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 0 ms
- retrieve: 0.0 ms
- llama-server: ~11260.7 ms

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Bottleneck hoàn toàn nằm ở llama-server khi sinh chữ (Decode phase). Điều này khớp với kỳ vọng vì phần retrieve đã được giả lập (stub) ngay trên memory bằng TOY_DOCS.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Dùng GPU (CUDA) thay vì CPU thuần

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: ~ 2.5 tok/s (CPU)
after:  10.7 tok/s (CUDA)
speedup: ~4.2×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Giai đoạn sinh chữ (decode) của LLM bị giới hạn bởi băng thông bộ nhớ (memory-bound), không phải khả năng tính toán (compute-bound). Băng thông của bộ nhớ VRAM trên card RTX 3070 (GDDR6) vượt trội gấp nhiều lần so với băng thông của RAM hệ thống (DDR4). Bằng việc chạy llama.cpp backend CUDA (`-DGGML_CUDA=on`), mô hình được đẩy hoàn toàn lên VRAM (vì size mô hình 1.5B rất nhỏ gọn), giúp tốc độ đẩy các trọng số mô hình qua các lõi tính toán tăng lên đáng kể, từ đó kéo theo Decode rate tăng 4 lần.

---

## 6. (Optional) Điều ngạc nhiên nhất

_Thư viện llama-cpp-python và cấu trúc GGML/GGUF quá mạnh mẽ trong việc tối ưu phần cứng để chạy mô hình cục bộ với chỉ một vài MB RAM khởi tạo._

---

## 7. Self-graded checklist

- [ ] `hardware.json` đã commit
- [ ] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [ ] `benchmarks/01-quickstart-results.md` đã commit
- [ ] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
