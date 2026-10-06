# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 16.9 | 1.00x | 21% |
| 8 | 26.6 | 1.57x | 34% |
| 16 | 34.1 | 2.02x | 43% |
| 24 | 49.7 | 2.93x | 63% |
| 32 | 62.2 | 3.67x | 79% |
| 99 | 79.1 | 4.67x | 100% |

Best: `-ngl 99` at 79.1 tok/s
-- 4.67x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding 

- **Kết quả Full offload**: Trên máy tính có GPU NVIDIA RTX 4050 Laptop (6GB VRAM), Full offload (`-ngl 99`) mang lại hiệu năng cao nhất đạt **79.1 tok/s**, nhanh gấp **4.67x** so với chỉ dùng CPU (`-ngl 0` đạt 16.9 tok/s).
- **Phân tích đường cong hiệu năng**: Đường cong throughput tăng tuyến tính và đơn điệu theo số layer được offload lên GPU (từ 16.9 -> 26.6 -> 34.1 -> 49.7 -> 62.2 -> 79.1 tok/s). Không xuất hiện hiện tượng sụt giảm (peak below full offload) do toàn bộ mô hình Gemma 4 E2B UD-Q4_K_XL (~2.97 GB) cùng KV cache hoàn toàn nằm gọn trong 6GB VRAM của RTX 4050.
- **Cơ chế**: Ở các mức partial offload (ví dụ `-ngl 8` đến `-ngl 32`), hệ thống phải liên tục luân chuyển tensor activation giữa CPU (RAM máy chủ) và GPU (VRAM) qua bus PCIe, tạo ra chi phí trễ truyền thông liên thiết bị (host-device transfer overhead). Khi đạt full offload (`-ngl 99`), toàn bộ chu trình tính toán và bộ nhớ đều được giữ cục bộ trên GPU VRAM băng thông cao (~192 GB/s), loại bỏ hoàn toàn nút thắt cổ chai PCIe và tối ưu hóa throughput.
