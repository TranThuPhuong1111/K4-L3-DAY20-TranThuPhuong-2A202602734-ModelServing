# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `11th Gen Intel(R) Core(TM) i5-11400H @ 2.70GHz`
Vector extensions detected: AVX-512, AVX2
llama.cpp `b10488` both sides · `threads=6` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 31.6 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 31.6 | 1.00x |

On this machine, **they are within 3% -- no meaningful difference**.

before: 31.6 tok/s (prebuilt release)
after:  31.6 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.00x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.
A gap this small usually means the prebuilt binary already dispatches to the right kernels at runtime (releases ship one libggml-cpu-*.so per microarchitecture and pick via CPUID), or that this workload is bandwidth-bound rather than instruction-bound. Both are real findings -- say which one you think it is.


## Your explanation

**Không có khác biệt: 1.00× (decode) và 0.97× (prefill, `bonus-build-compare-pp512.md`).
Tự compile không giúp gì trên máy này, và có hai lý do độc lập cùng dẫn tới kết quả đó.**

**Cách build** (WSL không có compiler và không có sudo): toolchain conda-forge
(`gcc 16.2.0`, `cmake 4.4.4`) cài bằng micromamba ở user-level; đúng tag `b10488`;
`-DGGML_NATIVE=ON -DCMAKE_BUILD_TYPE=Release` như `make build-llama`, thêm
`-DLLAMA_BUILD_TESTS=OFF -DLLAMA_OPENSSL=OFF -DLLAMA_BUILD_UI=OFF` (tests, HTTPS, Web UI,
không đụng tới đường tính toán) và `-Wl,-rpath` tới `libgomp` của toolchain.

1. **Prebuilt không phải bản "baseline chung".** Release Linux ship 15 biến thể
   `libggml-cpu-*.so` và chọn theo CPUID lúc chạy. Log `llama-bench` của prebuilt:
   `loaded CPU backend from libggml-cpu-icelake.so`. CPU của mình là Tiger Lake
   (`gcc -march=native` → `tigerlake`). So với Ice Lake, Tiger Lake chủ yếu thêm
   `AVX512_VP2INTERSECT` và `MOVDIRI/MOVDIR64B`. Theo mình biết thì các kernel matmul
   của ggml không dùng những lệnh này. Các lệnh quan trọng
   (AVX-512 F/BW/VL, VNNI cho dot product int8, VBMI) **cả hai bản đều có**, nên
   `-DGGML_NATIVE=ON` không còn gì để mở khoá.
2. **Decode bị chặn bởi memory bandwidth, không phải bởi lệnh.** `make tune` cho thấy
   decode đã kéo ~16.7 GB/s trên RAM single channel 25.6 GB/s, và 3 core đã đạt 93% tốc
   độ tốt nhất. Dù compiler sinh code tốt hơn, CPU vẫn phải chờ cùng số byte trọng số từ
   RAM cho mỗi token. Prefill compute-bound là nơi duy nhất compiler có thể tạo khác
   biệt, và ở đó bản prebuilt (GNU 11.4) còn nhỉnh hơn bản gcc 16 của mình 3%. Mức đó
   nằm trong biên nhiễu giữa các vòng (pp512 dao động 208–229 tok/s), nên mình không
   coi đó là kết quả.

**Ghi chú đo: lần chạy `make compare-builds` đầu tiên bị hỏng.** Nó cho ra prebuilt 21.1
→ bản build 25.4 tok/s (**1.20×**). Lần đó chạy ngay sau khi WSL khởi động lại (lúc mình
chuyển distro từ ổ C: đầy sang D:), và prebuilt chạy trước, lúc máy còn "lạnh". 21.1 tok/s
thấp hơn hẳn 31.3 tok/s của chính binary đó trong `make tune`, nên mình không tin con số
1.20×. Để kiểm tra, mình đo xen kẽ A/B 5 vòng (`bonus-build-compare-ab.csv`, `-r 3` mỗi
lần):

| Vòng | Metric | Prebuilt (tok/s) | Source build (tok/s) | Tỉ lệ |
|--:|:--|--:|--:|--:|
| 1 | tg128 | 26.5 | 28.1 | 1.06x |
| 2 | tg128 | 31.3 | 31.4 | 1.00x |
| 3 | tg128 | 31.1 | 32.0 | 1.03x |
| 4 | tg128 | 31.4 | 31.8 | 1.01x |
| 5 | tg128 | 31.4 | 31.2 | 0.99x |
| 1 | pp512 | 223.1 | 219.2 | 0.98x |
| 2 | pp512 | 228.9 | 222.8 | 0.97x |
| 3 | pp512 | 217.4 | 208.1 | 0.96x |

Vòng 1 cả hai bản đều chậm (máy còn lạnh). Từ vòng 2 đến 5, trung bình là
31.3 so với 31.6 tok/s (**1.01×**). Báo cáo phía trên là lần
`compare-builds` chạy lại lúc máy đã ấm, và khớp với A/B. Bài học: một speedup 1.20× đo
một lần, không xen kẽ, trên laptop có thể chỉ là thứ tự chạy.
