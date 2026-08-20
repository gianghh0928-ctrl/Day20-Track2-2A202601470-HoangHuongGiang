# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 22 | 0.44 | 23000 | 30000 | 30000 | 9.2 | 0.0% |
| 50 | 18 | 0.36 | 25000 | 55000 | 55000 | 8.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.81x** (16% of linear) |
| P95 latency | **1.83x** |
| Effective concurrency at 50 users | 8.9 vs `--parallel 4` slots (occupancy/slot ratio 2.21) |

**Saturated.** Throughput delivered only 0.81x for 5x the offered load, and effective concurrency (8.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.81x while P95 moved 1.83x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 18 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading
Hệ thống đã bão hòa trước mức 50 users: 
- Khi tải tăng 5x: thông lượng không tăng mà giảm nhẹ. 
- Độ trễ P95 tăng lên: Chủ yếu là Queue time do 4 slots xử lý đã lấp đầy và có tới 46 request bị hoãn. 
Để nâng cao goodpu, tôi sẽ điều chỉnh là:
- Đặt thread = 2 để tốc độ decode nhanh nhất. 
- Cấu hình cơ chế từ chối tải hoặc giới hạn hàng đợi. 
