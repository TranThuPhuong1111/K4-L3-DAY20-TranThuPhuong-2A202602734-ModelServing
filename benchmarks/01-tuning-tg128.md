# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 19.3 | 62% |
| 3 | 29.3 | 93% |
| 6 | 31.3 | 100% |
| 12 | 25.4 | 81% |
| 24 | 3.9 | 12% |

**Best**: `-t 6` at 31.3 tok/s
**Slowest tested**: `-t 24` at 3.9 tok/s (8.05x spread)
**Against the physical-core default** (`-t 6`, 31.3 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

**Knee ở 3 thread, đỉnh ở 6 (= số core vật lý), trên 6 thì giảm, và ở 24 thì sụp hẳn.**

- **1 → 3 thread: 19.3 → 29.3 tok/s, nhưng 3 → 6 chỉ thêm +7% (31.3).** Decode đọc gần
  như toàn bộ ~0.53 GB trọng số cho mỗi token (Qwen3.5 dùng tied embedding nên lm_head
  cũng là cả bảng embedding). 31.3 tok/s × 0.53 GB ≈ **16.7 GB/s**. Máy mình chỉ có
  **1 thanh DDR4-3200 (single channel, `Controller0-ChannelA-DIMM0`)**, đỉnh lý thuyết
  3200 MT/s × 8 B = **25.6 GB/s**, nên mình đang dùng ~65% đỉnh, gần mức thực tế đạt được
  với streaming read. Chỉ 1 core đã kéo được ~10 GB/s (62% tốc độ tốt nhất), 3 core gần
  như đã bão hoà memory controller. Thêm core thì vẫn chờ cùng một channel RAM.
- **12 thread (SMT): 25.4 tok/s, giảm 19%.** Hai hyper-thread trên cùng core dùng chung
  L1/L2 và load port, mà load port đã kẹt chờ RAM, nên thread thêm không mang thêm
  bandwidth. Cái mất thêm là đồng bộ: ggml chia mỗi op (matmul, norm, ...) cho N thread
  rồi barrier sau mỗi op, mỗi token có hàng trăm op, và op nhỏ (batch 1) nên chi phí
  barrier với 12 thread vượt phần compute tiết kiệm được.
- **24 thread (gấp đôi số logical core): 3.9 tok/s, chậm 8×.** Lúc này là oversubscription
  thật: 24 thread trên 12 CPU logic. Thread ggml spin-wait ở barrier, nên khi OS
  preempt một thread, 23 thread còn lại phải quay vòng chờ nó ở **mỗi** op. Với hàng
  trăm barrier mỗi token, mỗi lần bị preempt cộng thêm một time slice của scheduler.
- **Đối chiếu với prefill** (`01-tuning-pp512.md`): cùng máy, prefill tiếp tục tăng tới
  12 thread và ở 24 thread gần như không giảm. Hai đường cong khác nhau là vì decode
  memory-bound còn prefill compute-bound, đúng như deck nói.
- **Kết luận cho máy này:** `-t 6` (đúng default của lab) là tốt nhất cho decode. So với
  lựa chọn "dùng hết 12 logical thread" thì nhanh hơn **1.23×** (25.4 → 31.3 tok/s).
