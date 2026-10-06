# Bonus - Batch-size sweep (chunked prefill)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=6` `ngl=0` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 214.4 | 87% |
| 256 | 256 | 246.1 | 100% |
| 512 | 256 | 239.1 | 97% |
| 512 | 512 | 234.3 | 95% |
| 1024 | 512 | 233.0 | 95% |
| 2048 | 512 | 236.4 | 96% |

Best: `-b 256 -ub 256` at 246.1 tok/s
(1.15x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

**Micro-batch 128 → 256 tăng prefill 1.15× (214.4 → 246.1 tok/s). Từ 256 trở lên thì
đi ngang (233–239 tok/s, chênh ≤ 5%, ngang mức nhiễu với `reps=2`).**

- **Cơ chế 128 → 256 (đã sửa sau khi làm C9):** ban đầu mình viết rằng ở `ub = 128`
  matmul vẫn còn bị chặn bởi memory bandwidth. **Điều đó sai.** Đo ở C9
  (`bonus-c9-embedding-serving.md`): `llama-bench` pp16 đã đạt 213 tok/s, nghĩa là CPU
  này compute-bound từ khoảng ~16 token/lượt. Ở 128 token/ubatch, mỗi token chỉ cần
  ~4 MB trọng số (~0.9 GB/s ở 214 tok/s), còn rất xa trần ~16.7 GB/s. Cách giải thích
  khớp hơn là chi phí cố định **mỗi ubatch**: prefill 512 token với `ub = 128` là 4 lần
  chạy graph, mỗi lần đi qua hàng trăm op, mỗi op có một barrier giữa 6 thread. `ub = 256`
  giảm số lần đó xuống một nửa, và kernel matmul lượng tử hoá cũng dùng lại mỗi block
  trọng số đã dequantize cho nhiều cột activation hơn. Phần chênh 15% này mình **chưa tách
  được** giữa hai nguyên nhân đó. Trên 256 thì cả hai đã được bù đủ, nên đường cong đi
  ngang.
- **Hơi giảm ở `ub = 512`** (246 → 234): giả thuyết là activation của 512 token × 1024
  dim × 4 B ≈ 2 MB, vượt L2 1.25 MB/core của i5-11400H, nên phải xuống L3. Mình chưa
  kiểm chứng; mức chênh này cũng gần ngưỡng nhiễu.
- **Production, mình sẽ chạy `-b 512 -ub 256`.** Đây vừa là điểm throughput tốt nhất,
  vừa nhỏ hơn default `-ub 512`, nên mỗi chunk prefill chiếm CPU ngắn hơn và các request
  đang decode trong batch bị chặn ít hơn. Để chắc nó không làm hại P95 của server đang
  tranh chấp, cần chạy lại `make load-50` (có tỉ lệ request `long-rag`) với `-ub 256` và
  `-ub 512`, rồi so **TPOT P95** của request đang decode (bị chen bởi prefill của request
  mới) và **TTFT P95** (prefill bị chia nhỏ nên tốn nhiều bước hơn). Throughput tổng ở đây
  không phải thứ quyết định, vì server đã bão hoà ở 4 slot.
