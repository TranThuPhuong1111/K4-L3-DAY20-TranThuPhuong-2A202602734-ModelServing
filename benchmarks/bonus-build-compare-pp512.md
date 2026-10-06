# Bonus B1 - Prebuilt vs source build

Host `Linux-x86_64` · CPU `11th Gen Intel(R) Core(TM) i5-11400H @ 2.70GHz`
Vector extensions detected: AVX-512, AVX2
llama.cpp `b10488` both sides · `threads=6` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `pp512`, 3 repetitions

| Binary | Built for | pp512 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 223.2 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 217.5 | 0.97x |

On this machine, **they are within 3% -- no meaningful difference**.

before: 223.2 tok/s (prebuilt release)
after:  217.5 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.97x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.
A gap this small usually means the prebuilt binary already dispatches to the right kernels at runtime (releases ship one libggml-cpu-*.so per microarchitecture and pick via CPUID), or that this workload is bandwidth-bound rather than instruction-bound. Both are real findings -- say which one you think it is.


## Your explanation

**0.97×: bản prebuilt nhỉnh hơn 3%, nằm trong biên nhiễu.** Ở 3 vòng A/B xen kẽ
(`bonus-build-compare-ab.csv`), trung bình là 223.1 (prebuilt) so với 216.7
tok/s (source), tức 0.97×, trong khi cùng một binary dao động 208–229 tok/s
giữa các vòng.

Prefill là compute-bound, nên nếu compile có tác dụng thì phải thấy ở đây. Không thấy
vì prebuilt đã nạp `libggml-cpu-icelake.so` (chọn theo CPUID), tức đã dùng AVX-512 +
VNNI giống bản `-march=native` (tigerlake) của mình. Khác biệt còn lại giữa hai bản là
phiên bản compiler (GNU 11.4 so với gcc 16.2), và với các kernel ggml viết tay bằng
intrinsics thì compiler không đổi được nhiều. Giải thích chi tiết và ghi chú về lần chạy
hỏng nằm ở `bonus-build-compare-tg128.md`.
