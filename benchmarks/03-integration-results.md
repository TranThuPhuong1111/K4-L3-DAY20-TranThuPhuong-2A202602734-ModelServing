# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 12809.9 | 12810.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.2 | 0.3 | 3099.9 | 3100.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 4198.6 | 4198.7 |

Mean per stage (ms): embed **0.1** · retrieve **0.1** ·
llm **6702.8** · total **6703.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it **ignores SLOs (Service Level Objects) at saturation**.

Here is the breakdown of why this makes Goodput superior:

*   **SLOs are ignored at saturation:** The context states that "Throughput at saturation ignores SLOs." Since Goodput calculates throughput based on requests per second (TPS) that meet targets, 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it removes the fragmentation that would otherwise waste most of the GPU's memory.

**When does splitting prefill and decode help?**

> Based on the provided context, splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**.

The context explicitly states that prefill is compute-bound, meaning it requires significant processing power, while decode is memory-bandwidth-bound, meaning it requires significant data transfer speed. Therefore, splitting them allows the system to utilize co


## Which N16-N19 pieces are real

| Day | Piece | Real hay stub |
|---|---|---|
| N16 Cloud/IaC | localhost (WSL2 trên laptop), không có k8s/Compose | **stub** |
| N17 Data pipeline | `TOY_DOCS` in-memory list, không có Airflow/batch job | **stub** |
| N18 Lakehouse | dict trong `pipeline.py`, không có Delta/Iceberg | **stub** |
| N19 Vector + features | keyword overlap (`retrieve()` STUB 2), không có vector index/Feast. Không chạy embed server nên `embed ≈ 0 ms` | **stub** |
| N20 Serving | `llama-server` b10488, Qwen3.5-0.8B Q4_K_M, CPU | **real** |

**Stage chiếm nhiều nhất là llm (100%: 6702.8 / 6703.4 ms trung bình).** Đúng với kỳ
vọng. Retrieval bằng keyword overlap trên 6 doc mất 0.1 ms, còn LLM 0.8B trên CPU sinh
43–200 token ở ~26–30 tok/s.

- Query 1 (12.8 s) lệch hẳn, chủ yếu vì model sinh đủ 200 token (chạm `max_tokens`).
  Prefill 151 token của nó cũng chậm (72 tok/s). Một phần lý do là slot được chọn theo
  LRU nên không dùng lại được gì, còn query 2–3 được chọn theo LCP similarity (0.26),
  giữ lại 11–20% prompt (system prompt giống từng byte) và prefill 123–209 tok/s. Nhưng
  11–20% reuse không đủ giải thích chênh lệch 3×, nên phần còn lại có thể là CPU vừa
  thoát trạng thái rảnh sau load test. Mình chưa kiểm chứng.
  Ngoài ra còn ~3.7 s client-side (12.8 s so với 9.1 s server tự báo) mà mình **chưa giải
  thích được**. Query 2–3 chỉ lệch ~0.5 s. Giả thuyết là overhead kết nối của lần gọi
  đầu, nhưng mình chưa kiểm chứng.
- **Muốn giảm 2× thì đánh vào decode trong stage llm**, vì decode chiếm ~77% thời gian
  server xử lý (decode 6969 / 1610 / 3118 ms so với prefill 2108 / 923 / 540 ms), tức
  ~58% của llm đo phía client.
  Rẻ nhất là giới hạn độ dài câu trả lời (`max_tokens` 200 → 100 cắt đôi decode của
  query 1). Muốn nhanh hơn mà không cắt câu trả lời thì phải tăng bandwidth: offload GPU.
  Tối ưu retrieval/embed không giúp gì vì chúng chỉ chiếm 0%.
