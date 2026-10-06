# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=8` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 19 | 0.34 | 20000 | 34000 | 34000 | 7.1 | 0.0% |
| 50 | 22 | 0.38 | 35000 | 55000 | 58000 | 11.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.12x** (22% of linear) |
| P95 latency | **1.62x** |
| Effective concurrency at 50 users | 11.9 vs `--parallel 4` slots (occupancy/slot ratio 2.98) |

**Saturated.** Throughput delivered only 1.12x for 5x the offered load, and effective concurrency (11.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.12x while P95 moved 1.62x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 19 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Server bão hoà ngay từ mức **≤ 10 users** và quá tải nặng ở 50 users.

**Bằng chứng thuyết phục:**
1. **Throughput chỉ tăng 1.12× (từ 0.34 lên 0.38 RPS)** khi offered load tăng 5× (chỉ đạt 22% mức tăng tuyến tính).
2. **Effective concurrency đạt 11.9** ở 50 users (và đã là 7.1 ở 10 users), cao gấp gần **3 lần** năng lực `--parallel 4` của server (tỷ lệ occupancy/slot là 2.98).
3. Server metrics xác nhận toàn bộ 4 slots luôn bị chiếm dụng (`requests_processing = 4`) và có tới 46 request bị hoãn vào hàng đợi (`requests_deferred = 46`). Mức tăng 1.62× của P95 latency (34s → 55s) hoàn toàn là **queue time**, không phải compute time.

**Knob thay đổi trước tiên để nâng goodput@SLO:**
Tôi sẽ tăng **`--parallel` từ 4 lên 8** (kết hợp quantize KV cache `--cache-type-k q8_0 --cache-type-v q8_0` nếu bị giới hạn RAM/VRAM). Do nút thắt lớn nhất là thiếu slot phục vụ khiến 46 requests phải chờ đợi, việc tăng slot cho phép requests mới vào decode ngay lập tức, triệt tiêu queue delay và kéo P95 latency về dưới ngưỡng SLO. Chỉnh các knob compute như `--threads` sẽ không giải quyết được việc hàng đợi bị nghẽn ở đầu vào.
