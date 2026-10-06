# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Tran Thu Phuong
**MSSV:** 2A202602734
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home (host). Lab chạy trong **WSL2 Ubuntu** trên cùng laptop (kernel `6.18.33.2-microsoft-standard-WSL2`)
- **CPU:** Intel Core i5-11400H @ 2.70GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX-512, AVX2
- **RAM:** 7.8 GB trên host (1 thanh DDR4-3200, **single channel**). WSL2 được cấp 3.7 GB, `hardware.json` ghi 3.7 GB
- **Accelerator:** NVIDIA RTX 3050 Laptop 4 GB có trên máy nhưng **không dùng**: chạy CPU only, `ngl=0` (lý do ở setup story)
- **llama.cpp asset đã tải:** `llama-b10488-bin-ubuntu-x64.tar.gz` (CPU)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=`qwen35-0.8b)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (WSL2 Ubuntu trên Windows 11). Không dùng Colab/Kaggle.
Probe chọn Qwen3.5 0.8B vì RAM < 8 GB.

**Setup story** (≤ 80 chữ):

Trên Windows, Smart App Control chặn `llama-server.exe` vì binary không có chữ ký
(`WinError 4551`), và `lab.ps1` lỗi parse trên PowerShell 5.1 (file UTF-8 không có
BOM). Mình chuyển sang WSL2: lấy bản CPU `ubuntu-x64` (không lấy Vulkan vì WSL chỉ có
`llvmpipe`, tức Vulkan giả lập trên CPU), tạo venv bằng `get-pip.py` vì không có sudo,
và đặt `libgomp.so.1` (giải nén từ `apt-get download libgomp1`) cạnh binary. Các lệnh
`make` chạy trực tiếp bằng `.venv/bin/python <script>`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2125 | 175 / 231 | 32.9 / 35.0 | 2254 / 2366 / 2366 | 30.4 |
| UD-Q2_K_XL | 0.39 | 2054 | 237 / 252 | 28.0 / 29.2 | 2002 / 2079 / 2079 | 35.8 |

**Quan sát** (≤ 60 chữ):

2-bit decode nhanh hơn **1.18×** (TPOT 32.9 → 28.0 ms) và nhỏ hơn 0.11 GB, nhưng **TTFT
chậm hơn 35%** (175 → 237 ms) vì prefill compute-bound mà Q2 dequantize đắt hơn.
**Không đáng:** cùng câu hỏi, Q4 đúng (Canberra, 391), Q2 sai (Sydney, 381) và lặp vòng
(`01-quant-quality-check.md`). Đây là lần bench thứ hai; lần đầu bị nhiễu.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.80 | 11000 | 15000 | 15000 | 8.7 | 0 (0.0%) |
| 50 | 0.67 | 32000 | 55000 | 57000 | 20.3 | 0 (0.0%) |

- **Offered load tăng 5×, throughput thực tăng:** 0.83× (tức là giảm)
- **P95 tăng:** 3.67×
- **Effective concurrency ở 50 users:** 20.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.93 / 4 slots (`requests_deferred` lên tới 46)

**Saturation reading** (≤ 80 chữ):

Bão hoà **từ 10 users**: effective concurrency 8.7 > 4 slot, P50 11 s (chạy một mình
2.3 s). Lên 50 users: RPS 0.83×, P95 3.67×. Bằng chứng: 4 slot luôn bận (3.93/4),
`deferred = 46`, nên P95 thêm là **queue time**; `--parallel 1` cho P95 gần y hệt.
Knob đổi trước: **admission control** (~8 request in-flight), không phải `--parallel 8`:
batch 4 chỉ cho 1.4× throughput, mỗi slot chỉ còn 256 token ctx.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost (WSL2), không có k8s/Compose | stub |
| N17 Data pipeline | `TOY_DOCS` list in-memory, không có Airflow/batch job | stub |
| N18 Lakehouse | dict trong `pipeline.py`, không có Delta/Iceberg | stub |
| N19 Vector + features | keyword overlap (`retrieve()` STUB 2), không có vector index/Feast, không có embed server | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.1 ms (không có embed server, fallback keyword overlap)
- retrieve: 0.1 ms
- llm: 6702.8 ms
- **stage chiếm nhiều nhất:** llm (100% của total 6703.4 ms)

**Reflection** (≤ 60 chữ):

Đúng kỳ vọng: retrieval toy gần 0, LLM 0.8B trên CPU chiếm hết. Trong llm, decode chiếm
~77% thời gian server (query 1 sinh đủ 200 token, 12.8 s). Muốn giảm 2×: giảm
`max_tokens`/độ dài câu trả lời trước, sau đó tăng memory bandwidth (offload GPU). Còn
~3.7 s client-side ở query 1 mình chưa giải thích được.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ số thread decode từ `-t 12` (dùng hết logical thread) xuống `-t 6` (= số
core vật lý), đo bằng `make tune` (`llama-bench`, tg128, Qwen3.5-0.8B Q4_K_M)

```
before:  25.4 tok/s   (-t 12)
after:   31.3 tok/s   (-t 6)
speedup: 1.23×
```

(Cùng sweep: `-t 1` = 19.3, `-t 3` = 29.3, `-t 24` = 3.9 tok/s. Default của lab đã là
`-t 6`, nên so với default thì là 1.00×, và `make tune` xác nhận default đó đúng cho
máy mình.)

**Tại sao nó work:**

Decode sinh mỗi token bằng cách đọc lại gần như toàn bộ ~0.53 GB trọng số (Qwen3.5 dùng
tied embedding nên lm_head là cả bảng embedding), và mỗi weight chỉ dùng cho một phép
nhân. Vậy tốc độ decode ≈ memory bandwidth / số byte mỗi token. Ở 31.3 tok/s mình đang
kéo ~16.7 GB/s. Laptop của mình chỉ có **một thanh DDR4-3200, single channel**, đỉnh lý
thuyết 25.6 GB/s, nên đã dùng ~65% đỉnh. Đường cong cho thấy đúng điều đó: 1 core đã đạt
62% tốc độ tốt nhất, 3 core đạt 93%, 6 core chỉ thêm 7% nữa. Memory controller bão hoà
từ ~3 core, nên thêm core không còn byte nào để kéo thêm. Lên 12 thread, hai
hyper-thread cùng core tranh chung load port và L1/L2 vốn đã đứng chờ RAM, nên không có
bandwidth mới. Cái thêm vào chỉ là chi phí đồng bộ: ggml chia mỗi op cho N thread và đặt
barrier sau mỗi op, mỗi token có hàng trăm op nhỏ (batch = 1), nên nhiều thread hơn
nghĩa là nhiều thời gian chờ nhau hơn. Kết quả: -19%. Ở 24 thread (gấp đôi CPU logic)
thì sập 8×: thread spin-wait ở barrier, nên mỗi khi OS preempt một thread, 23 thread kia
phải chờ nó hết một time slice, và chuyện đó lặp lại ở mỗi op.

Bằng chứng mình tin nhất cho cơ chế này là **prefill đi theo hướng ngược lại** trên cùng
máy (`benchmarks/01-tuning-pp512.md`): pp512 tăng gần tuyến tính tới 6 core (3.3×),
`-t 12` còn **nhanh hơn** `-t 6` 15%, và `-t 24` chỉ chậm 2%. Prefill dùng mỗi weight
cho 512 token nên là compute-bound. Khi đó SMT lấp được chỗ trống của FMA unit, và mỗi op
đủ lớn để chi phí barrier/preempt không đáng kể. Cùng một knob, hai phase, hai kết luận
ngược nhau. Bench cũng khớp: Q2 (ít byte hơn) decode nhanh hơn nhưng prefill chậm hơn.
Với chat ngắn, decode chiếm ~90% E2E (64 × 33 ms ≈ 2.1 s so với TTFT 0.18 s), nên `-t 6`
là lựa chọn đúng. Điều deck không nói mà máy mình cho thấy: trần bandwidth ở đây là do
**cấu hình RAM single channel**. Lắp thêm một thanh RAM (dual channel) có lẽ là
"speedup" lớn nhất có thể, lớn hơn bất kỳ knob phần mềm nào.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B1 (build llama.cpp `b10488` với `-DGGML_NATIVE=ON` + `compare-builds`), B2
(`make sweep-batch` + `make sweep-ctx`), B3 (before/after từ `sweep-batch`), B4 (challenge
**C2**: KV cache `q8_0` so với `f16`), B5 (challenge **C9**: embedding serving). Vì WSL
không có compiler và không có sudo, mình cài gcc 16.2 + cmake bằng micromamba
(conda-forge) ở user-level.

**Numbers** (B3: `benchmarks/bonus-batch-size-sweep.md`, pp512, `-t 6`, CPU):

```
before:  214.4 tok/s   (-b 128 -ub 128)
after:   246.1 tok/s   (-b 256 -ub 256)
speedup: 1.15×
```

So với default của llama.cpp (`-b 2048 -ub 512`, 236.4 tok/s) thì chỉ còn 1.04×, ngang
mức nhiễu. Speedup thật chỉ nằm ở bước từ micro-batch quá nhỏ (128) lên 256. Ban đầu mình
giải thích là "chuyển từ memory-bound sang compute-bound", nhưng C9 bác bỏ điều đó: pp16
đã đạt 213 tok/s, tức CPU này compute-bound từ ~16 token. Giải thích hợp lý hơn là chi
phí cố định mỗi ubatch (số lần chạy graph và số barrier giảm một nửa), cộng với việc tái
sử dụng block trọng số đã dequantize tốt hơn. Mình chưa tách được hai nguyên nhân này.

**B1 — tự compile không nhanh hơn:** tg128 31.6 → 31.6 tok/s (**1.00×**), pp512 223.2 →
217.5 (**0.97×**, trong biên nhiễu). Prebuilt Linux không phải bản "baseline chung": nó
ship 15 biến thể `libggml-cpu-*.so` và tự nạp `libggml-cpu-icelake.so` (AVX-512 +
VNNI), gần như cùng tập lệnh với `-march=native` = tigerlake của mình. Thêm nữa, decode
bị chặn bởi bandwidth nên compiler không giúp được. Lần `compare-builds` đầu cho 1.20×,
nhưng đó là artifact: prebuilt chạy trước, lúc máy còn lạnh sau khi WSL khởi động lại
(21.1 tok/s, so với 31.3 của chính binary đó trong `make tune`). Đo xen kẽ A/B 5 vòng
cho 1.01× (`bonus-build-compare-ab.csv`).

**B5 / C9 — batching không giúp embedding trên CPU:** batch 1 → 64 text chỉ tăng 7.3 →
8.9 texts/s (1.22×), latency tăng tuyến tính ~110 ms/text, `--parallel 32` không đổi gì.
Hai nguyên nhân đo được: (1) prefill trên CPU đã compute-bound từ ~16 token (pp16 213
so với pp256 234 tok/s), nên static batch lớn không còn compute rảnh để lấp như trên GPU;
(2) chế độ embedding tính lm_head (vocab 248k) cho **mọi** token. Theo FLOPs mình dự đoán
chậm 1.51×; `llama-bench -embd 1` đo được 1.53×.

**Điều này nói lên gì mà deck chưa nói:**

Cả ba thí nghiệm bonus đều quay về một sự thật: Qwen3.5-0.8B là model **hybrid**
(metadata GGUF: `full_attention_interval = 4`). Chỉ 6/24 layer có attention + KV cache,
18 layer là SSM/Gated DeltaNet với state cố định. Hệ quả đo được:
(1) `sweep-ctx` gần như tuyến tính: 4096 token chỉ tốn 1.17× so với tuyến tính, khớp
ước lượng FLOPs ~1.12× (một model full attention cùng shape sẽ ~1.5×), nên "prefill
O(N²)" của deck gần như không thấy;
(2) C2: KV chỉ 12 KB/token, nên `q8_0` tiết kiệm 11 MB ở ctx 2048 (đo RSS khớp tính toán
tới MB) nhưng làm prefill chậm hơn 41% trên CPU. Lời khuyên "FP8 KV cache" của deck giả
định KV là thứ chiếm bộ nhớ nhiều nhất, điều không còn đúng với model hybrid;
(3) prefix cache chỉ tái dùng được tới checkpoint: câu hỏi mới trên cùng một context
2929 token vẫn phải prefill ~515 token, vì recurrent state không tua lại được như KV.
Kiến trúc model thay đổi cả các kết luận về serving, không chỉ chất lượng.

Sự thật thứ hai đến từ B1 và C9: **phần cứng quyết định chỗ nào đáng tối ưu.** Trên CPU
6 core với RAM single channel, decode bị chặn bởi bandwidth (nên compiler không giúp,
B1 1.00×), còn prefill đã chạm trần compute từ ~16 token (nên static batch lớn không
giúp embedding, C9 1.22×). Các lời khuyên "tự compile", "batch embedding thật lớn" trong
deck đúng với baseline yếu hoặc với GPU, nơi điểm giao giữa memory-bound và compute-bound
nằm ở hàng trăm token. Hai lời khuyên đó không tự động đúng trên laptop này.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Little's Law cho effective concurrency 20.3 ở 50 users, trong khi gauge của server cho
thấy đúng 50 request đang ở trong hệ thống (4 đang chạy + 46 chờ). Công thức chỉ đúng ở
steady state, mà run 60 s với latency trung bình 30 s thì chưa bao giờ đạt steady state.
Request chưa xong không được đếm, và cuối run chúng bị cancel.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/` (`01`–`05`; ảnh `03-serve-and-smoke.png` là một ảnh chụp màn hình có cả hai terminal serve + smoke cạnh nhau; thêm 2 ảnh optional `07-batching.png`, `08-pipeline.png`)
- [x] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Dùng **Claude Code (Claude Opus 5.5)** trong VS Code để: chẩn đoán lỗi setup (Smart App
Control chặn binary, `lab.ps1` trên PowerShell 5.1, thiếu `libgomp`) và dựng workaround
qua WSL2; chạy các lệnh của lab (`probe`, `bench`, `tune`, `serve`, `smoke`, `load-10/50`,
`metrics`, `load-report`, `pipeline`) và chụp screenshot từ cửa sổ terminal thật đang
chạy lệnh; chạy thêm thí nghiệm `--parallel 1`, so chất lượng Q4/Q2, các sweep bonus và
script đo C2 (RSS + eval 10 câu); cài toolchain bằng micromamba, build llama.cpp và đo
A/B cho B1; viết script đo C9 và các thí nghiệm kiểm chứng (`-embd`, 1 text dài so với
16 text); đọc metadata GGUF; soạn nháp phần nhận xét trong `benchmarks/*.md` và
REFLECTION. Mọi con số đều lấy từ output của các lần
chạy đó, không sửa tay. Mình đã đọc lại và chịu trách nhiệm về lập luận.
