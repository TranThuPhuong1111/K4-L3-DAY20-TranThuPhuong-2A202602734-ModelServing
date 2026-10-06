# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 47 | 0.80 | 11000 | 15000 | 15000 | 8.7 | 0.0% |
| 50 | 39 | 0.67 | 32000 | 55000 | 57000 | 20.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.83x** (17% of linear) |
| P95 latency | **3.67x** |
| Effective concurrency at 50 users | 20.3 vs `--parallel 4` slots (occupancy/slot ratio 5.07) |

**Saturated.** Throughput delivered only 0.83x for 5x the offered load, and effective concurrency (20.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.83x while P95 moved 3.67x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server đã bão hoà ngay từ 10 users, không phải đâu đó giữa 10 và 50.**

- **Bằng chứng ở 10 users:** effective concurrency 8.7 > 4 slot, tức trung bình ~4.7
  request đang chờ slot, nên khoảng một nửa latency là thời gian xếp hàng. P50 là 11 s
  cho câu trả lời 48–96 token, trong khi một request chạy một mình chỉ mất ~2.3 s
  (`make bench`, E2E P50 2254 ms ở 64 token). Nửa còn lại cũng chậm hơn chạy một mình vì
  mỗi bước decode phải chia cho 4 sequence (~105 ms/bước thay vì ~33 ms).
- **Từ 10 → 50 users:** offered load 5×, throughput **0.83×** (0.80 → 0.67 RPS), P95
  **3.67×** (15 s → 55 s). Con số thuyết phục mình nhất là `requests_deferred = 46` với 4
  slot luôn bận (3.93/4): thêm 40 user thì gần như toàn bộ thành hàng đợi.
- **RPS còn giảm** (không chỉ plateau), và mình chưa tách được hết nguyên nhân. Ở 50
  users, server đo được ~37 tok/s (`02-parallel-1-vs-4.md`). Ở 10 users mình không chạy
  `make metrics`, nên chỉ ước lượng thô từ counter `tokens_predicted_total` (~44–47
  tok/s, sai số lớn vì ranh giới cửa sổ không chính xác). Nếu ước lượng đúng thì server
  cũng chậm đi ~15%, khớp với RPS -17%. Hai giả thuyết: (1) công việc của ~50 request dở
  dang bị `cancel task` lúc locust dừng (log server), nên đã tốn CPU mà không thành
  request hoàn thành nào; (2) locust, `make metrics` và các app Windows tranh CPU với
  server, trong khi server đã dùng hết 6 core vật lý, và `make tune` cho thấy decode rất
  nhạy với tranh chấp (-19% chỉ vì thêm SMT thread).
- **Phần P95 tăng thêm là queue time, không phải compute.** Khi batch đã đầy 4/4 (cả ở
  10 lẫn 50 users số request in-flight đều > 4), mỗi bước decode tốn ~105 ms bất kể có
  bao nhiêu request chờ phía sau. Thứ tăng theo số user là số request đứng trước bạn
  trong hàng: deferred 46 ở 50 users, so với ~4.7 ở 10 users.
- **Goodput @ SLO:** chọn SLO là **P95 E2E ≤ 15 s** cho câu trả lời ≤ 96 token. Ở 10
  users toàn bộ 47 request đạt (P100 = 15.4 s, sát ngưỡng), goodput ≈ 0.80 RPS. Ở 50
  users P50 đã là 32 s, nên **chưa tới một nửa** request đạt SLO, goodput < 0.33 RPS, tức
  mất hơn 60% so với 10 users dù tải cao hơn 5×.
- **Knob mình đổi trước: giới hạn số request đồng thời (admission control).** Ví dụ chỉ
  nhận ~8 request in-flight (2× số slot) và trả 429 cho phần còn lại. Lý do: dữ liệu cho
  thấy request thêm vào không tăng throughput mà chỉ tăng chờ, nên chặn sớm giữ P95 của
  request được nhận quanh mức 10 users (~15 s). **Không** tăng `--parallel` trước, vì
  (1) batch 4 đã chỉ cho 1.4× so với batch 1 (bước decode 38 → 105 ms), nên 8 slot sẽ làm
  TPOT của mọi người tệ đi mà throughput tăng rất ít; (2) `ctx 2048 / 8 = 256` token mỗi
  slot, nhỏ hơn request long-rag (log server: 324–359 token/slot), nên sẽ bị truncate.
  Muốn tăng capacity thật thì phải tăng bandwidth: offload sang RTX 3050 (đang không dùng)
  hoặc RAM dual-channel.
