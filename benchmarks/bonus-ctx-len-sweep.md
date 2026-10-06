# Bonus - Context-length sweep (prefill cost)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=8` `ngl=99` · RAM 15.6 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 146.7 | 1744.9 | 1.00x |
| 1024 | 123.5 | 8288.8 | 1.19x |
| 2048 | 2207.1 | 927.9 | 0.07x |
| 4096 | 4576.8 | 894.9 | 0.03x |
| 8192 | 4361.8 | 1878.1 | 0.03x |

At 8192 tokens, prefill costs **1878 ms** --
0.03x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

- **Mối quan hệ giữa Context Length và Prefill TTFT**: Khi tăng độ dài ngữ cảnh từ 256 lên 8192 tokens, giai đoạn prefill đòi hỏi tính toán ma trận self-attention trên toàn bộ dãy prompt trước khi sinh token đầu tiên.
- **Tác động tới RAG Pipeline**: Trong các ứng dụng RAG, việc nhồi nhét quá nhiều context chunks (ví dụ 8k+ tokens) sẽ làm phình to thời gian TTFT lên hàng giây, trở thành nút thắt cổ chai chi phối toàn bộ độ trễ người dùng cảm nhận được. Do đó, một RAG pipeline hiệu quả cần giới hạn số lượng retrieved chunks (k=3 đến k=5, prompt dưới 2048 tokens) hoặc bắt buộc phải có Prompt Caching để tái sử dụng KV cache của các tài liệu tĩnh.
