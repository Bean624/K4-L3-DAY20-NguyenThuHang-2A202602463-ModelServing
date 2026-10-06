# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Thu Hằng

**MSSV:** 2A202602463

**Cohort:** K4-L3

**Ngày submit:** 2026-10-06


---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** 13th Gen Intel(R) Core(TM) i5-13420H
- **Cores:** 8 physical / 12 logical
- **CPU extensions:** AVX2
- **RAM:** 15.6 GB
- **Accelerator:** NVIDIA GeForce RTX 4050 Laptop GPU (6141 MiB), Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** gemma-4-E2B-it-UD-Q4_K_XL.gguf + gemma-4-E2B-it-UD-Q2_K_XL.gguf

**Chạy ở đâu:** Laptop cá nhân

**Setup story** (≤ 80 chữ): Chạy trên laptop Windows với PowerShell 5.1, script probe ban đầu gặp lỗi charmap khi in ký tự Unicode do mã hóa mặc định. Em đã khắc phục bằng cách thiết lập PYTHONIOENCODING=utf-8 và lưu file lab.ps1 dưới dạng UTF-8 có BOM. Sau đó tiến trình cài đặt runtime CUDA và tải hai bản weights GGUF diễn ra hoàn toàn suôn sẻ.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3644 | 451 / 702 | 12.5 / 13.1 | 1225 / 1523 / 1523 | 79.8 |
| UD-Q2_K_XL | 2.24 | 3185 | 189 / 691 | 9.6 / 10.9 | 792 / 1379 / 1379 | 103.7 |

**Quan sát** (≤ 60 chữ): Bản 2-bit decode nhanh gấp 1.30x (103.7 vs 79.8 tok/s) và nhẹ hơn 0.73 GB (~25%). Khi kiểm tra câu trả lời thực tế, bản 4-bit mạch lạc và chính xác hơn hẳn; bản 2-bit suy luận kém hơn, chỉ phù hợp cho tác vụ nhẹ cần phản hồi tức thì.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 3.42 | 1900 | 3300 | 4200 | 6.9 | 0.0% |
| 50 | 3.39 | 13000 | 14000 | 15000 | 41.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.99x
- **P95 tăng:** 4.24x
- **Effective concurrency ở 50 users:** 41.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.80 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa hoàn toàn ở dưới 50 users vì throughput đi ngang 0.99x khi tải tăng 5x. P95 tăng vọt 4.24x hoàn toàn là queue time vì effective concurrency đạt 41.7 vượt xa 4 slots. Để nâng goodput@SLO, em ưu tiên tăng `--parallel` lên 8 kết hợp lượng tử hóa KV cache để giảm nghẽn hàng đợi.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local Windows Laptop | stub |
| N17 Data pipeline | TOY_DOCS in-memory | stub |
| N18 Lakehouse | Dictionary storage | stub |
| N19 Vector + features | Keyword overlap fallback | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2680.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): LLM là nút thắt cổ chai chiếm 100% latency, đúng với kỳ vọng trong RAG. Để giảm latency 2x, cần tập trung vào LLM: dùng Prompt Caching cho context/system prompt và chuyển sang quantization 2-bit hoặc speculative decoding.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Chuyển đổi quantization từ UD-Q4_K_XL sang UD-Q2_K_XL

```
before: 79.8 tok/s 
after: 103.7 tok/s 
speedup: 1.30x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Trên GPU NVIDIA RTX 4050, giai đoạn decode của mô hình ngôn ngữ lớn bị giới hạn nghiêm trọng bởi băng thông bộ nhớ (memory bandwidth bound) chứ không phải năng lực tính toán (FLOPs). Trong mỗi bước sinh token, toàn bộ trọng số của mô hình phải được nạp lại từ VRAM vào các nhân tính toán của GPU.

Bản lượng tử hóa 2-bit (UD-Q2_K_XL) có kích thước 2.24 GB, nhẹ hơn 0.73 GB (~24.6% dung lượng) so với bản 4-bit (2.97 GB). Nhờ giảm mạnh lượng dữ liệu cần luân chuyển qua bus bộ nhớ VRAM trên mỗi token, tốc độ decode đã tăng vọt từ 79.8 tok/s lên 103.7 tok/s (đạt mức speedup thực tế 1.30x).

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

**Đã làm:** B2 sweep-gpu (GPU Layer Offload Sweep) & B5/Challenge C8 (Semantic Caching Offline)
**Numbers:**
```
before: 16.9 tok/s (-ngl 0, CPU-only) 
after: 79.1 tok/s (-ngl 99, Full GPU Offload) 
speedup: 4.67x
```

**Điều này nói lên gì mà deck chưa nói:**
1. **Khảo sát GPU Offload (B2/B3)**: Khác với CPU scaling (nơi throughput đi ngang do nghẽn memory bandwidth), GPU offload mang lại mức tăng tốc gần như tuyến tính theo số layer được chuyển giao sang GPU (từ 16.9 tok/s ở `-ngl 0` lên 79.1 tok/s ở `-ngl 99`, đạt **4.67x speedup**). Do VRAM 6GB của RTX 4050 đủ sức chứa toàn bộ model 2.97 GB, không hề có hiện tượng tràn VRAM hay nghẽn băng thông host-device PCIe ở mức full offload.
2. **Thử nghiệm Semantic Cache C8 (B4/B5)**: Semantic Cache nằm ở tầng cao nhất trước cả KV cache, giúp bắt các câu hỏi diễn đạt lại (paraphrase) có độ tương đồng ngữ nghĩa cao. Với tập prompt thử nghiệm, cache đạt hit rate **38% (3/8 requests)**, giúp tiết kiệm 100% tài nguyên tính toán (bỏ qua cả prefill lẫn decode) với độ trễ phục vụ = 0 ms. Tuy nhiên, trong môi trường multi-tenant, cần áp dụng cơ chế salting cache theo tenant để ngăn ngừa rò rỉ dữ liệu qua kênh timing attack (NDSS'25).


---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Em ngạc nhiên nhất khi thấy Continuous Batching có thể ép xung năng lực phục vụ lên tới 3.80 / 4 slots (95% công suất) dưới tải 50 users, và nhận ra độ trễ tăng vọt khi quá tải chủ yếu đến từ thời gian xếp hàng (queue time) chứ không phải do mô hình tính toán chậm đi.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md` đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Được hỗ trợ bởi trợ lý AI Antigravity trong việc giải thích khái niệm serving, xử lý lỗi encoding PowerShell trên Windows và rà soát đối chiếu số liệu báo cáo.

