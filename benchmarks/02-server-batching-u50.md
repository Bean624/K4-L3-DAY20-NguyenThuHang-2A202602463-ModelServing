# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.80 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 44 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 25983 |

Highest sampled value was **3.80 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation 

- **Peak batch width quan sát được**: Giá trị đỉnh trung bình đạt `3.80 / 4 slots` (tương đương 95% công suất tối đa của `--parallel 4`), với `requests_processing = 4` và có thời điểm `requests_deferred` lên tới 44 requests trong hàng đợi.
- **So sánh với Effective concurrency (41.7)**: Hai con số này đo lường hai khía cạnh khác nhau nhưng hoàn toàn tương thích và bổ trợ cho nhau:
  - Chỉ số `3.80 / 4 slots` từ `/metrics` đo lường mức độ sử dụng phần cứng thực tế (Hardware Slot Utilisation) trong lõi llama-server tại từng bước decode, giá trị này bị chặn trên bởi cấu hình `--parallel 4`.
  - Chỉ số `41.7` tính theo Định luật Little ($L = \lambda \times W$) đo lường tổng tải hệ thống (System Occupancy), bao gồm cả 4 request đang được xử lý trong slot và khoảng ~37-38 request đang phải xếp hàng chờ trong hàng đợi (`requests_deferred`).
- **Độ tin cậy**: Cả hai số liệu đều đáng tin cậy ở góc độ riêng: con số `3.80` chứng minh continuous batching đã tận dụng tối đa năng lực GPU, trong khi con số `41.7` chứng minh tình trạng nghẽn hàng đợi nghiêm trọng khi tải tăng lên 50 users.
