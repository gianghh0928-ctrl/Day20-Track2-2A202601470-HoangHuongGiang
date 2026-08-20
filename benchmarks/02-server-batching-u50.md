# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.82 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 2188 |

Highest sampled value was **3.82 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation
Peak batch width: 3.82/4 slots
Effective concurrency: 8.9
Peak bathc width cho thấy cơ chế hoạt động hiệu quả. 
Con số này không bị mâu thuẫn với Effective Concurrency vì:
- 3,82 thể hiện Slot Utilisation thực tế bên trong inference engine.  => Cho biết engine đã bão hòa tính toán
- 8,9 thể hiện System Occupancy bao gồm cả 4 request đang xử lý và nhiều request đang chờ. => Chứng minh tình trạng nghẽn hàng đợi 
Hai số liệu mô tả 2 khía cạnh khác nhau.  
