# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.93 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5043 |

Highest sampled value was **3.93 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

**Peak batch width là 3.93 / 4 slot (98%), nên continuous batching hoạt động và cả 4 slot
luôn bận.** Gauge này là trung bình tích luỹ từ lúc khởi động server (gồm cả request
smoke-test 1-slot và load-10), nên trong riêng load-50 thì gần như chắc chắn là 4/4.
`requests_processing` = 4 và `requests_deferred` dao động 42–46 trong suốt 60 s.

**So với effective concurrency trong `02-server-results.md` (20.3):** hai số không mâu
thuẫn, chúng đo hai thứ khác nhau, và mình tin gauge của server hơn:

- Số request **đang thực sự ở trong server** = processing + deferred = 4 + 46 = **50**,
  tức đúng bằng số user locust (wait_time 0.2–1.5 s rất nhỏ so với latency ~30 s, nên gần
  như user nào cũng đang có request treo).
- Little's Law (L = λ·W = 0.67 × 30.4 s = 20.3) chỉ đúng ở **steady state**, mà run 60 s
  với latency trung bình 30 s (max 57 s) chưa đạt steady state. λ chỉ đếm request **đã
  xong**. ~50 request đang treo lúc locust dừng không được tính, và log server cho thấy
  chúng bị `cancel task` khi locust ngắt kết nối. Vì vậy Little's Law ở đây đánh giá thấp
  độ đông thực tế khoảng 2.5×.
- Hai mẫu cuối `processing = 0, deferred = 0` là lúc locust dừng và huỷ hết request,
  không phải server rảnh tự nhiên.
