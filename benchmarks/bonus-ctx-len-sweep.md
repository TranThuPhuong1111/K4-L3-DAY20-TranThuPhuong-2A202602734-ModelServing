# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=6` `ngl=0` · RAM 3.7 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 237.4 | 1078.1 | 1.00x |
| 1024 | 233.1 | 4392.8 | 1.02x |
| 2048 | 217.9 | 9397.9 | 1.09x |
| 4096 | 202.9 | 20187.3 | 1.17x |

At 4096 tokens, prefill costs **20187 ms** --
1.17x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

**Gần như tuyến tính: 16× token (256 → 4096) chỉ tốn 1.17× so với tuyến tính
(237 → 203 tok/s). Đường cong bậc hai của deck gần như không thấy, và đó là do kiến trúc
model, không phải do range đo quá ngắn.**

- **Vì sao:** metadata GGUF của Qwen3.5-0.8B có `full_attention_interval = 4`: chỉ
  **6/24 layer** là full attention (O(N²)), 18 layer còn lại là SSM/Gated DeltaNet với
  state cố định (O(N)). Ước lượng FLOPs: với một layer full attention (8 head × 256 dim),
  phần attention của token ở vị trí N tốn ~4096·N MAC, trong khi projection + FFN tốn
  ~16.3 M MAC/token. Trung bình trên prompt dài N thì phần thêm là ≈ N/7960, tức +51% ở
  N = 4096 cho **riêng layer đó**. Chỉ 1/4 số layer như vậy nên tổng chỉ thêm ~+13%, tức
  dự đoán ~1.12× so với điểm 256. Đo được 1.17×, phần dư có thể do KV/attention ít
  cache-friendly hơn. Một model full attention cùng shape sẽ ở khoảng ~1.45–1.5×.
- **Prefill bắt đầu lấn decode ở ~500 token prompt:** một câu trả lời 64 token tốn ~2.1 s
  decode (TPOT 32.9 ms), và prefill 237 tok/s cũng tốn ~2.1 s ở ~500 token. RAG prompt 4096
  token tốn 20 s chỉ riêng TTFT trên CPU này.
- **RAG được bao nhiêu chunk:** ở đây cái giới hạn là tổng compute tuyến tính, không phải
  bậc hai. Với SLO TTFT ≤ 2 s cho một request chạy một mình, budget là ~470 token prompt,
  tức ~3–4 chunk 100 token cộng system prompt. Khi 4 slot cùng chạy thì budget còn nhỏ
  hơn. Thêm nữa, ctx mặc định 2048 / 4 slot = 512 token/slot, nên trần cứng cũng gần như
  vậy.
