# Bonus C2 - KV cache quantization (f16 vs q8_0)

Host: laptop i5-11400H, WSL2 Ubuntu (3.7 GB), CPU only (`ngl=0`, `-t 6`) · llama.cpp `b10488` ·
`Qwen3.5-0.8B-Q4_K_M` · `--parallel 4` · raw data: `bonus-c2-kv-cache-quant.json`

**Thay đổi:** `serve.py -- --cache-type-k q8_0 --cache-type-v q8_0` so với mặc định (`f16`),
ở `LAB_N_CTX=2048` (default của lab) và `LAB_N_CTX=32768`.

**Cách đo:** RSS của process `llama-server` (`/proc/<pid>/status`, VmRSS) sau khi load,
và sau eval. Eval có 10 câu hỏi chấm tự động: một "kho" 120 dòng
`Record j: the <color> box weighs <w> kg and is stored in room <r>.` (~2930 token), mỗi câu
hỏi số kg của một record, `temperature=0`, so khớp đúng con số. Cùng 10 câu, cùng thứ tự
cho cả hai cache type (chỉ chạy ở ctx 32768, vì ở 2048 mỗi slot chỉ có 512 token).

## Số liệu

| ctx | KV cache | RSS sau load (MB) | RSS sau eval (MB) | Đúng (/10) | Prefill câu 1 (2929 tok) | Prefill câu 2–10 (~515 tok, median) | TPOT median |
|--:|:--|--:|--:|--:|--:|--:|--:|
| 2048 | f16 | 880.6 | — | — | — | — | — |
| 2048 | q8_0 | 869.6 | — | — | — | — | — |
| 32768 | f16 | 1240.7 | 1351.6 | 8 | 13541 ms | 2601 ms | 35.7 ms |
| 32768 | q8_0 | 1069.1 | 1181.1 | 9 | 16631 ms | 3675 ms | 36.5 ms |

**Đối chiếu với tính toán.** Metadata GGUF: 24 block, `full_attention_interval = 4`, nên
chỉ **6 layer có KV cache**, mỗi layer 2 KV head × 256 dim. KV f16 = 6 × 2 (K,V) × 2 × 256
× 2 B = **12 KB/token**:

| | Tính toán | Đo được (RSS) |
|:--|--:|--:|
| f16: ctx 2048 → 32768 | 384 − 24 = **360 MiB** | 1240.7 − 880.6 = **360 MB** |
| f16 → q8_0 ở ctx 32768 (q8_0 ≈ 1.06 B/giá trị) | 384 − 204 = **180 MiB** | 1240.7 − 1069.1 = **172 MB** |
| f16 → q8_0 ở ctx 2048 | 24 − 12.75 = **11 MiB** | 880.6 − 869.6 = **11 MB** |

## Phân tích

- **Ở ctx mặc định, q8_0 gần như vô nghĩa: tiết kiệm 11 MB (~1% RSS).** Phần lớn RSS
  là trọng số (~0.53 GB) và compute buffer, không phải KV. Ở ctx 32768 thì tiết kiệm
  172 MB (14% RSS), có ích nếu RAM thật sự chật.
- **Cái giá là prefill chậm hơn ~41% trên CPU** (median 2601 → 3675 ms cho ~515 token).
  Mỗi K/V ghi vào cache phải quantize, và attention phải dequantize khi đọc. Decode thì
  gần như không đổi (TPOT median 35.7 → 36.5 ms), vì mỗi token decode đọc ~0.53 GB trọng số
  nhưng chỉ ~36 MB KV (~3000 token × 12 KB/token), nên giảm nửa số byte KV không
  thay đổi gì đáng kể.
- **Chất lượng: không thấy suy giảm** (8/10 so với 9/10). Câu sai chung của cả hai
  (record 15) là lỗi của model. Câu khác nhau (record 116: f16 trả lời 61, q8_0 trả lời
  đúng 72) với 10 mẫu chỉ là nhiễu, không có nghĩa q8_0 "tốt hơn".
- **Điều deck chưa nói:** lời khuyên "FP8 KV cache" giả định KV là thứ chiếm bộ nhớ
  nhiều nhất, tức là đúng với model full attention. Qwen3.5 là model **hybrid**: 18/24
  layer dùng recurrent state (SSM/Gated DeltaNet) có kích thước cố định, chỉ 6/24 layer
  có KV, nên KV nhỏ hơn 4× so với một model full-attention cùng shape. Với model này, KV
  quantization có đòn bẩy nhỏ hơn 4×, và trên CPU còn phải trả bằng prefill chậm hơn.
  Kết luận cho máy mình: **giữ f16**, chỉ cân nhắc q8_0 khi chạy context rất dài và
  thiếu RAM.
- **Phát hiện phụ về prefix cache:** cùng một "kho" 2929 token, câu 2–10 vẫn phải
  prefill ~515 token chứ không chỉ ~20 token câu hỏi mới. Recurrent state không "tua
  lại" tới một vị trí bất kỳ như KV được, nên llama.cpp chỉ quay về được checkpoint gần
  nhất. Prefix caching trên model hybrid vì thế chỉ là một phần, khác với hình dung
  "cache hit thì bỏ qua prefill" trong deck (RadixAttention).
