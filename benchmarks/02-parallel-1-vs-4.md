# 02 - Thí nghiệm thêm: `--parallel 1` vs `--parallel 4` ở 50 users

Cùng máy, cùng model (`Qwen3.5-0.8B-Q4_K_M`, `-t 6`, `ngl=0`, `ctx=2048`), cùng lệnh
locust như `make load-50` (50 users, ramp 25/s, 60 s). Chỉ đổi `LAB_PARALLEL`.

- `--parallel 4`: `locust-50_stats.csv` + `02-server-metrics-u50.csv` (lần chạy `make load-50` / `make metrics`).
- `--parallel 1`: `locust-50-p1_stats.csv`, cộng `/metrics` lấy bằng `curl` ngay trước và
  ngay sau lần chạy (server vừa khởi động lại nên counter bắt đầu từ 0).

| `--parallel` | Reqs xong | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Server decode tok/s | Thời gian mỗi lần `llama_decode` |
|--:|--:|--:|--:|--:|--:|--:|--:|
| 1 | 27 | 0.46 | 32000 | 56000 | 58000 | 26.0 (1586 tok / 60.9 s) | 38 ms (1597 lần / 60.9 s) |
| 4 | 39 | 0.67 | 32000 | 55000 | 57000 | 36.6 (2025 tok / 55.3 s) | 105 ms (525 lần / 55.3 s) |

Tok/s của `--parallel 4` tính trên cửa sổ lấy mẫu của `make metrics` (mẫu đầu tiên và mẫu
bận cuối cùng trong `02-server-metrics-u50.csv`), nên hai cửa sổ lệch nhau ~5 s. Cả hai
đều là tốc độ (rate) nên vẫn so được.

## Nhận xét

- **Batching tăng throughput, không giảm latency.** RPS 0.46 → 0.67 (1.46×), decode
  throughput phía server 26.0 → 36.6 tok/s (1.41×), nhưng P95 vẫn ở 55–56 s. Ở 50 users
  cả hai cấu hình đều đã quá bão hoà, nên P95 chủ yếu là thời gian xếp hàng. Thêm slot
  thì hàng đợi được xử lý nhanh hơn, nhưng mỗi user vẫn phải chờ sau ~46 người khác.
- **Vì sao 4 slot chỉ cho 1.4×.** Một bước decode với ~4 sequence tốn ~105 ms, so với
  ~38 ms khi chỉ có 1 sequence: chi phí gấp ~2.8× để sinh ~3.9× token. Trọng số chỉ đọc
  một lần mỗi bước và dùng chung cho cả batch (đó là phần tiết kiệm bandwidth), nhưng trên
  CPU này phần việc riêng của từng sequence (thêm cột trong matmul, attention trên KV
  riêng, cập nhật state của các layer recurrent trong Qwen3.5) không miễn phí, nên bước
  decode đắt lên khi batch lớn hơn. Trên GPU dư compute, chi phí một bước gần như không đổi
  theo batch size; trên laptop 6 core thì có đổi.
