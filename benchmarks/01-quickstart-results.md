# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2125 | 175 / 231 | 32.9 / 35.0 | 2254 / 2366 / 2366 | 30.4 |
| UD-Q2_K_XL | 0.39 | 2054 | 237 / 252 | 28.0 / 29.2 | 2002 / 2079 / 2079 | 35.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.18x faster** than `Q4_K_M` here, for 0.11 GB less on disk.

## Your observation

**2-bit nhanh hơn rất ít, và không đáng dùng trên máy này.**

- **Decode:** TPOT P50 32.9 → 28.0 ms, tức 30.4 → 35.8 tok/s (**1.18×**). File nhỏ hơn
  0.11 GB (0.50 → 0.39 GB, 0.78× số byte). Speedup (1.18×) nhỏ hơn tỉ lệ giảm byte
  (1.27×) vì decode không chỉ là đọc trọng số: unpack Q2_K tốn nhiều lệnh hơn cho mỗi
  weight, bản "UD" giữ một số tensor ở precision cao hơn, và còn chi phí cố định mỗi
  token (sampling, HTTP streaming).
- **Prefill thì ngược lại:** TTFT P50 175 → **237 ms (chậm hơn 35%)**. Prefill
  compute-bound, nên số byte ít hơn không giúp gì, còn chi phí dequantize cao hơn của Q2
  hiện ra trực tiếp. Đây là bằng chứng rõ nhất trong bảng rằng prefill và decode bị chặn
  bởi hai tài nguyên khác nhau.
- **Chất lượng** (cùng câu hỏi, `temperature=0`, xem `01-quant-quality-check.md`):
  Q4 trả lời đúng "Canberra" và "17 × 23 = 391". Q2 trả lời "Sydney", "381", và rơi vào
  vòng lặp lặp lại ở 2/3 câu hỏi.
- **Kết luận:** đổi 1.18× decode (và TTFT tệ hơn) lấy câu trả lời sai là không đáng. Q2
  chỉ hợp lý khi RAM thật sự không chứa nổi Q4, mà với model 0.5 GB thì máy này
  (3.7 GB cho WSL) không gặp tình huống đó.
- **Ghi chú đo:** đây là lần chạy `make bench` thứ hai. Lần đầu (ngay sau khi tải model,
  page cache còn lạnh, nhiều app chạy nền) bị nhiễu nặng: TPOT P95 của Q4 là 75 ms và một
  request Q2 có TTFT 1169 ms. Mình chạy lại và commit lần thứ hai (là lần có screenshot
  `02-bench.png`). Chỉ có 10 mẫu mỗi quant, nên P95 = P99 = giá trị lớn nhất.
