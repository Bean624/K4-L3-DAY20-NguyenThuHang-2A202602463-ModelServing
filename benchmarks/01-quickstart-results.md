# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3644 | 451 / 702 | 12.5 / 13.1 | 1225 / 1523 / 1523 | 79.8 |
| UD-Q2_K_XL | 2.24 | 3185 | 189 / 691 | 9.6 / 10.9 | 792 / 1379 / 1379 | 103.7 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.30x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

- **Dung lượng**: Bản 2-bit (UD-Q2_K_XL) nhẹ hơn 0.73 GB so với bản 4-bit (2.24 GB vs 2.97 GB, giảm khoảng 24.6% dung lượng).
- **Tốc độ**: Bản 2-bit decode nhanh gấp 1.30x so với bản 4-bit (103.7 tok/s vs 79.8 tok/s; TPOT P50 giảm từ 12.5 ms xuống 9.6 ms). Thời gian phản hồi token đầu tiên (TTFT P50) cũng nhanh hơn rõ rệt (189 ms vs 451 ms). Nguyên nhân do mô hình được offload hoàn toàn lên GPU NVIDIA RTX 4050 (`ngl=99`), quá trình decode bị giới hạn bởi memory bandwidth nên bản 2-bit nhẹ hơn giúp tiết kiệm băng thông truyền tải trên bus VRAM.
- **Đánh giá đánh đổi**: Bản 2-bit mang lại cải thiện tốc độ đáng kể (~30%) và độ trễ thấp hơn, phù hợp cho các tác vụ cần phản hồi nhanh hoặc thiết bị hạn chế tài nguyên. Tuy nhiên, việc lượng tử hóa xuống 2-bit làm giảm đáng kể độ chính xác và khả năng suy luận logic so với bản 4-bit. Do đó, trên máy có đủ VRAM (RTX 4050 6GB), bản 4-bit vẫn là lựa chọn cân bằng tối ưu giữa chất lượng và tốc độ.
