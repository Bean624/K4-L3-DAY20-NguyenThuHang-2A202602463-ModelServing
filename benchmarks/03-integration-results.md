# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 2780.4 | 2780.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2525.5 | 2525.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2735.8 | 2735.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2680.6** · total **2680.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real 

- **Khai báo các thành phần N16 - N19**:
  - N16 Cloud/IaC: stub (chạy local trên Windows laptop)
  - N17 Data pipeline: stub (sử dụng danh sách văn bản mẫu TOY_DOCS)
  - N18 Lakehouse: stub (lưu trữ in-memory dictionary)
  - N19 Vector + features: stub (fallback keyword overlap matching do chưa bật server embedding riêng)
  - N20 Serving: **real** (kết nối trực tiếp tới `llama-server` thật đang chạy qua cổng HTTP :8080)
- **Đánh giá stage chiếm ưu thế**: Giai đoạn LLM chiếm tới 100% thời gian trễ (2680.6 ms trên tổng 2680.6 ms). Kết quả này hoàn toàn khớp với kỳ vọng thực tế trong kiến trúc RAG: việc tìm kiếm văn bản cục bộ diễn ra gần như tức thì (< 1 ms), trong khi việc nạp context vào prompt (prefill) và sinh câu trả lời (decode) của mô hình ngôn ngữ lớn là tác vụ tốn tài nguyên tính toán và băng thông nhất.
- **Giải pháp nếu cần giảm độ trễ 2x**:
  - Tấn công trực tiếp vào stage **LLM** vì nó chiếm trọn 100% latency.
  - Áp dụng các giải pháp:
    1. Chuyển sang mô hình lượng tử hóa sâu hơn (bản 2-bit UD-Q2_K_XL giúp tăng tốc decode 1.30x).
    2. Sử dụng Prompt Caching / Prefix Caching để lưu lại KV cache của system prompt và context đã nạp, giúp bỏ qua giai đoạn prefill khi người dùng hỏi các câu liên quan.
    3. Giới hạn `max_tokens` đầu ra hoặc sử dụng kỹ thuật Speculative Decoding.
