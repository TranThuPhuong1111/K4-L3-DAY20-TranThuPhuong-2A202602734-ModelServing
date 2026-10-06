# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `pp512`

| threads (-t) | pp512 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 68.9 | 26% |
| 3 | 159.3 | 61% |
| 6 | 229.0 | 87% |
| 12 | 262.6 | 100% |
| 24 | 256.1 | 98% |

**Best**: `-t 12` at 262.6 tok/s
**Slowest tested**: `-t 1` at 68.9 tok/s (3.81x spread)
**Against the physical-core default** (`-t 6`, 229.0 tok/s): 1.15x

Use this in your run:

```bash
LAB_N_THREADS=12 make bench
```

## Your explanation

**Prefill có hình dạng khác hẳn decode: tăng gần tuyến tính tới 6 core, SMT còn cho thêm
+15%, và oversubscription (24 thread) gần như không gây hại (-2%).**

- 1 → 3 → 6 thread: 68.9 → 159.3 → 229.0 tok/s (3.3× với 6 core). Prefill xử lý 512
  token trong một lần forward, nên mỗi weight đọc từ RAM được dùng cho 512 token. Đây là
  matmul thật sự (compute-bound), nên thêm core là thêm FLOPs.
- 12 thread (SMT): 262.6 tok/s (+15% so với 6). Khi bottleneck là execution unit
  (AVX-512/AVX2 FMA), hyper-thread thứ hai lấp được những chu kỳ core bị stall, nên SMT
  có ích ở đây, khác với decode, nơi cả hai thread cùng chờ RAM.
- 24 thread: 256.1 tok/s (-2%). Mỗi op của prefill lớn gấp ~512 lần op của decode, nên
  chi phí barrier và preemption bị chia đều trên nhiều việc hơn rất nhiều. Cùng
  oversubscription đó làm decode chậm 8× nhưng prefill gần như không thấy.
- **Hệ quả:** thread count tối ưu phụ thuộc vào phase. Với chat ngắn (prompt ~40 token,
  output 64 token), decode chiếm ~90% E2E (TPOT 33 ms × 64 ≈ 2.1 s, so với TTFT 0.18 s),
  nên mình giữ `-t 6`. Với RAG prompt dài, `-t 12` có thể đáng cân nhắc cho TTFT, nhưng
  llama-server dùng chung `-t` cho cả hai phase, trừ khi tách bằng `--threads-batch`.
