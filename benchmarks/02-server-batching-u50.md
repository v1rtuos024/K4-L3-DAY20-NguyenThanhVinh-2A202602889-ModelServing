# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` � `--parallel 4` � 14 samples over
60s at 2.0s intervals � raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.26 of 4 slots (82%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a � not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 406 |

Highest sampled value was **3.26 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak sampled batch width (`n_busy_slots_per_decode`) đạt **3.26 / 4 slots** (~82% slot utilization) và số request xử lý đồng thời (`requests_processing`) chạm đỉnh **4 slots** (bão hòa toàn bộ 4 slots của server).

Con số này **không khớp** với effective concurrency trong `02-server-results.md` (đạt **11.9** ở 50 users theo định luật Little: $L = \lambda \times W$). Hai con số này khác nhau vì:
- **Peak batch width (3.26 / 4 slots):** Đo lường mức độ song song thực tế tại tầng decode/compute của engine `llama.cpp` (bị giới hạn vật lý bởi `--parallel 4`).
- **Effective concurrency (11.9):** Đo lường tổng số request đang nằm trong hệ thống từ góc nhìn client, bao gồm cả request đang tính toán lẫn request đang chờ trong hàng đợi (`requests_deferred = 46`).

Tôi tin tưởng **cả hai con số cho từng mục đích**: tin `3.26 / 4` khi đánh giá năng lực thực thi và hiệu quả continuous batching của phần cứng, và tin `11.9` để chẩn đoán tình trạng quá tải/hàng đợi (chứng minh server đã bão hòa và độ trễ P95 tăng vọt là do queue time).
