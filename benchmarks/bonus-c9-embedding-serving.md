# Bonus C9 (B5) - Embedding serving regime

Host: laptop i5-11400H, WSL2 Ubuntu, CPU only (`ngl=0`, `-t 6`) · llama.cpp `b10488` prebuilt ·
`Qwen3.5-0.8B-Q4_K_M` chạy ở chế độ embedding (`serve.py --embedding` = `--embedding --pooling mean`),
vector 1024 chiều · raw data: `bonus-c9-embedding-serving.json`

**Giới hạn cần nói trước:** đây là **chat model** dùng làm embedder (mean-pool hidden state
cuối), không phải embedding model chuyên dụng (Qwen3-Embedding, BGE-M3, EmbeddingGemma).
Số liệu dưới đây nói về **regime serving**, không nói về chất lượng retrieval.

## 1. Output của `make embed-demo`

```text
==> Embedding backend: llama-server /v1/embeddings @ http://localhost:8081/v1
    dim = 1024   corpus = 8 docs

Query: Does embedding serving use a KV cache and a decode loop like chat serving?

Top matches (cosine similarity):
  1. 0.897  Embedding serving is prefill-bound: one forward pass, no KV cache, no decode loop.
  2. 0.850  RadixAttention reuses a shared prompt prefix across requests via a radix tree.
  3. 0.823  Speculative decoding drafts several tokens and verifies them in one forward pass.

==> Throughput sweep (texts/sec as batch grows — prefill-bound):
  batch  1:   133.6 ms  ->     7.5 texts/s
  batch  2:   194.2 ms  ->    10.3 texts/s
  batch  4:   387.9 ms  ->    10.3 texts/s
  batch  8:   842.4 ms  ->     9.5 texts/s
  batch 16:  1703.1 ms  ->     9.4 texts/s
```

Top-1 đúng (0.897), nhưng hai câu **không liên quan** đứng ngay sau với 0.850 và 0.823.
Khoảng cách giữa doc đúng và doc sai rất hẹp. Hidden state của decoder mean-pool lại
có xu hướng dồn về một hướng chung (anisotropy), nên mọi câu đều "giống nhau" ở mức ~0.8.
Embedding model chuyên dụng được train contrastive để đẩy các câu không liên quan ra xa.

## 2. Latency theo batch size (prefill-only), đo kỹ hơn script gốc

Script gốc chỉ đo 1 lần mỗi batch, không warm-up. Ở đây mỗi điểm có 1 warm-up + 5 lần đo
(median), batch là các câu trong `CORPUS` lặp lại (~15 token/câu), batch tới 64. Hai cấu
hình: mặc định (`make serve-embed`: n_slots = 4, n_ctx_slot = 2048, kv_unified = 'true')
và `--parallel 32` (n_slots = 32, n_ctx_slot = 256, kv_unified = 'false').

| Batch (texts) | Prompt tokens | Default: median (ms) | texts/s | tok/s | `--parallel 32`: median (ms) | texts/s | tok/s |
|--:|--:|--:|--:|--:|--:|--:|--:|
| 1 | 15 | 137.1 | 7.3 | 109.4 | 126.7 | 7.9 | 118.4 |
| 2 | 28 | 234.3 | 8.5 | 119.5 | 235.5 | 8.5 | 118.9 |
| 4 | 58 | 454.0 | 8.8 | 127.8 | 443.3 | 9.0 | 130.8 |
| 8 | 121 | 943.4 | 8.5 | 128.3 | 859.6 | 9.3 | 140.8 |
| 16 | 242 | 1934.8 | 8.3 | 125.1 | 1784.0 | 9.0 | 135.7 |
| 32 | 484 | 3588.7 | 8.9 | 134.9 | 3799.8 | 8.4 | 127.4 |
| 64 | 968 | 7162.7 | 8.9 | 135.1 | 7343.6 | 8.7 | 131.8 |

**Kết quả trái với deck:** batch 1 → 64 (64× số text) chỉ tăng throughput
7.3 → 8.9 texts/s (**1.22×**),
còn latency tăng gần đúng tuyến tính, ~110 ms mỗi text. Tăng số slot lên 32 không thay đổi gì.

## 3. Vì sao batching không giúp: hai thí nghiệm kiểm chứng

| Thí nghiệm | Prompt tokens | Median (ms) | tok/s |
|:--|--:|--:|--:|
| 16 text khác nhau | 242 | 1818.7 | 133.1 |
| 16 bản sao **cùng một** text | 240 | 1844.1 | 130.1 |
| **1** text = nối 16 câu | 242 | 1763.2 | 137.2 |
| 1 text ngắn | 15 | 134.6 | 111.4 |
| `llama-bench` prefill pp16 (không HTTP) | 16 | — | 213.1 |
| `llama-bench` prefill pp256 (không HTTP) | 256 | — | 233.7 |
| `llama-bench -p 256 -embd 0` (2 lần) | 256 | — | 230.5 / 237.5 |
| `llama-bench -p 256 -embd 1` (2 lần) | 256 | — | 148.0 / 157.5 |

(Hai dòng `-embd` chạy xen kẽ 0,1,0,1 với `-r 3`; giá trị gốc ở khoá `llama_bench_pp256_embd_toggle` trong JSON.)

- **Không phải lỗi gộp batch giữa các sequence.** Ban đầu mình đoán model hybrid
  (recurrent state) buộc llama.cpp xử lý từng sequence một. Nhưng **một** text 242 token
  cũng chỉ đạt 137 tok/s, ngang 16 text riêng lẻ, nên giả thuyết đó sai.
- **Lý do 1: trên CPU, prefill đã compute-bound từ ~16 token.** `llama-bench` pp16 = 213
  tok/s, pp256 = 234 tok/s: 16× nhiều token hơn mỗi lượt đọc trọng số chỉ được thêm 1.1×.
  Với 16 token/lượt, mỗi token chỉ cần ~0.53 GB / 16 ≈ 33 MB trọng số, tức ~7 GB/s ở
  213 tok/s, thấp hơn nhiều so với ~16.7 GB/s mà decode kéo được. Nghĩa là FMA unit đã là
  trần. "Large static batch" có lợi trên GPU vì ở đó điểm giao giữa memory-bound và
  compute-bound nằm ở hàng trăm token. Trên CPU 6 core, điểm giao chỉ khoảng ~10 token,
  và một câu đã dài hơn mức đó.
- **Lý do 2: chế độ embedding tốn ~1.5× compute mỗi token.** Mình tính MAC/token từ shape
  tensor trong GGUF: 18 layer SSM × ~21.5 M + 6 layer attention × ~18.3 M ≈ **497 M**,
  còn lm_head (vocab 248,320 × 1024) là **254 M**. Prefill bình thường chỉ tính lm_head
  cho token cuối, còn khi bật embeddings thì llama.cpp xuất output cho **mọi** token, nên
  lm_head chạy trên toàn bộ prompt: (497 + 254) / 497 ≈ **1.51×**. `llama-bench -embd 1`
  đo được chậm **1.53×** (231–237 → 148–157 tok/s), khớp gần như hoàn toàn. Phần còn lại
  (157 → 137 tok/s qua server) là HTTP, tokenize và pooling. Một embedding model thật
  không có lm_head, nên tránh được toàn bộ chi phí này. Dùng lại chat GGUF làm embedder vì
  vậy không chỉ kém về chất lượng mà còn mất ~1/3 throughput.

## 4. Vì sao chat và embedding cần chiến lược batching ngược nhau

| | Chat (đo ở track 02) | Embedding (đo ở đây) |
|:--|:--|:--|
| Phase | decode, 1 token mỗi bước, đọc lại toàn bộ trọng số | prefill-only, 1 forward pass, không KV, không decode loop |
| Bị chặn bởi | memory bandwidth (16.7 GB/s single channel) | compute (FMA), từ ~16 token |
| Batching cho gì trên máy này | `--parallel 1 → 4`: 26.0 → 36.6 tok/s (**1.41×**) | batch 1 → 64: 7.3 → 8.9 texts/s (**1.22×**) |
| Batch lớn làm gì với latency | mỗi bước decode 38 → 105 ms (TPOT của mọi người tăng) | latency ∝ batch: 137 ms → 7.2 s |
| Chiến lược đúng | continuous batching: request vào/ra từng bước, giữ TPOT thấp | static batch lớn trên GPU (sort theo độ dài); trên CPU chỉ batch đủ để bù overhead HTTP |
| Tín hiệu autoscale | queue depth (`requests_deferred`), P95 TTFT/TPOT | backlog token cần embed / tok/s, thời gian xử lý mỗi batch |

**Hệ quả khi đặt cả hai sau một autoscaler:** cả hai đều cho "CPU 100%", nhưng ý nghĩa
khác nhau. Pod chat 100% CPU có thể chỉ là thread spin-wait chờ RAM (bandwidth-bound),
thêm request vẫn được batch rẻ. Pod embedding 100% CPU thì thật sự hết compute. Một
autoscaler theo CPU% sẽ scale sai cả hai. Tệ hơn, nếu chạy chung máy, một batch embedding
64 câu (~7 s prefill compute-bound) chiếm đúng các core mà bước decode cần, làm TPOT của
người dùng chat tăng vọt. Đây là cùng lý do deck tách prefill khỏi decode
(disaggregation): tách thành hai pool, mỗi pool có tín hiệu scale riêng (chat theo queue
và P95 TPOT, embedding theo backlog token).
