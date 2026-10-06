# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 203 | 3.42 | 1900 | 3300 | 4200 | 6.9 | 0.0% |
| 50 | 201 | 3.39 | 13000 | 14000 | 15000 | 41.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.99x** (20% of linear) |
| P95 latency | **4.24x** |
| Effective concurrency at 50 users | 41.7 vs `--parallel 4` slots (occupancy/slot ratio 10.42) |

**Saturated.** Throughput delivered only 0.99x for 5x the offered load, and effective concurrency (41.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.99x while P95 moved 4.24x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading 

- **Điểm bão hòa và Bằng chứng**: Server bị bão hòa hoàn toàn ở mức tải dưới hoặc bằng 50 users (thực tế đã tiệm cận bão hòa ngay từ mốc 10 users). Con số chứng minh rõ nhất là: khi tải tăng gấp 5x (từ 10 lên 50 users), thông lượng thực tế (Throughput / RPS) hầu như đi ngang, chỉ đạt 0.99x (từ 3.42 req/s giảm nhẹ xuống 3.39 req/s). Trong khi đó, độ trễ P95 tăng vọt 4.24x (từ 3300 ms lên 14000 ms). Toàn bộ độ trễ tăng thêm này là do thời gian xếp hàng (queue time / waiting time), thể hiện qua effective concurrency đạt 41.7 vượt xa sức chứa 4 slots.
- **Lập luận Goodput @ SLO**: Nếu đặt mục tiêu SLO cho hệ thống là P95 latency ≤ 4000 ms:
  - Ở mức 10 users: P95 là 3300 ms, toàn bộ 3.42 RPS đều đạt chuẩn SLO (Goodput = 100%).
  - Ở mức 50 users: P95 vọt lên 14000 ms, vi phạm nghiêm trọng SLO; hầu hết request đều bị quá hạn nên Goodput hữu ích giảm về gần 0.
- **Knob cần thay đổi đầu tiên để nâng cao Goodput**:
  - Knob ưu tiên số 1 là **tăng `--parallel` (ví dụ từ 4 lên 8 slots)** nếu còn đủ VRAM, kết hợp với KV cache quantization (`--ctk q8_0 --ctv q8_0`). Việc tăng số slot giúp server phục vụ đồng thời nhiều người hơn, giải tỏa trực tiếp hàng đợi `requests_deferred` và kéo giảm queue time.
  - Knob ưu tiên số 2 là chuyển sang bản lượng tử hóa nhẹ hơn (**UD-Q2_K_XL**), giúp tốc độ decode tăng 1.30x (như đã đo ở Track 01), từ đó giải phóng mỗi slot nhanh hơn để đón request tiếp theo.

