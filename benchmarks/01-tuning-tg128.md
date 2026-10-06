# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 92.8 | 99% |
| 4 | 93.7 | 100% |
| 8 | 89.6 | 96% |
| 12 | 90.5 | 97% |
| 24 | 90.4 | 96% |

**Best**: `-t 4` at 93.7 tok/s
**Slowest tested**: `-t 8` at 89.6 tok/s (1.05x spread)
**Against the physical-core default** (`-t 8`, 89.6 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation 

- **Hiện tượng quan sát được**: Tốc độ decode (tg128) gần như đi ngang trên toàn bộ dải thread từ 1 đến 24 threads (dao động trong khoảng hẹp từ 89.6 đến 93.7 tok/s, chênh lệch chỉ ~4.5%). Điểm tối ưu nhất đạt được tại `-t 4` (93.7 tok/s), trong khi số thread mặc định theo physical core (`-t 8`) lại là điểm chậm nhất được đo (89.6 tok/s).
- **Cơ chế và nguyên nhân**:
  1. Mô hình được offload hoàn toàn lên GPU NVIDIA RTX 4050 (`ngl=99`). Quá trình tính toán ma trận và đọc weights khi decode diễn ra trực tiếp trên GPU và VRAM băng thông cao, vì vậy throughput bị giới hạn bởi memory bandwidth của GPU chứ không phải CPU compute. Các thread CPU chủ yếu làm nhiệm vụ điều phối và phát lệnh (kernel dispatch) sang CUDA stream.
  2. CPU Intel Core i5-13420H có cấu trúc lai gồm 4 nhân P-core (Performance) và 4 nhân E-core (Efficient). Tại mức `-t 4`, các thread được phân bổ trọn vẹn trên các nhân P-core hiệu năng cao, giảm thiểu chi phí chuyển đổi ngữ cảnh (context switching) và tối ưu hóa việc gửi lệnh sang GPU.
  3. Khi tăng lên 8, 12 hoặc 24 threads (oversubscription), các thread thừa phải tranh chấp tài nguyên trên các nhân E-core và luồng ảo (hyper-threads), sinh ra chi phí điều phối và đồng bộ luồng (thread synchronization overhead), khiến throughput giảm nhẹ xuống 89.6 tok/s.
