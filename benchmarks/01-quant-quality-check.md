# 01 - So chất lượng: Q4_K_M vs UD-Q2_K_XL

Cùng 3 câu hỏi, gửi tới `serve.py` (Q4_K_M) rồi `serve.py --compare` (UD-Q2_K_XL),
`temperature=0`, `max_tokens=160`, port 8095, `-t 6`, CPU only. Câu trả lời dưới đây được
chép nguyên văn từ output (output dài bị cắt ở `max_tokens`).

| Câu hỏi | Q4_K_M | UD-Q2_K_XL |
|:--|:--|:--|
| Vì sao decode trên CPU bị chặn bởi memory bandwidth? | Có nhắc đến chờ dữ liệu từ RAM/cache nhưng lý giải sai (gắn với "kích thước RAM") | Bịa ra GPU trong câu hỏi về CPU, lặp lại cả đoạn |
| Thủ đô Úc, 17 × 23? | **Canberra**, **391**: đúng cả hai | **Sydney** (sai), **381** (sai), rồi lặp `17 * 23 equals 381` tới hết budget |
| 2 câu về Hà Nội (tiếng Việt) | Trôi chảy, nhưng bịa địa danh ("chùa Bái Bội Châu") | Câu đúng ngữ pháp nhưng nội dung rỗng, lặp một câu 5+ lần |

**Kết luận:** Q4_K_M chưa hoàn hảo (model 0.8B vẫn bịa chi tiết), nhưng Q2 hỏng ở mức
dùng không được: sai fact cơ bản, sai phép nhân, và rơi vào vòng lặp (repetition loop)
ở 2/3 câu hỏi.

## Output thô — Q4_K_M

```text
Q: Explain in 3 sentences why LLM decoding on a CPU is limited by memory bandwidth.
A: The primary limitation arises because the CPU's memory bandwidth is constrained by the physical size of the RAM, which is often far smaller than the total data volume required to process the entire input sequence. Consequently, the CPU cannot simultaneously fetch and process all data simultaneously, leading to a bottleneck where the system must wait for data to arrive at the next available memory location, effectively slowing down the decoding process.

Furthermore, the CPU's internal cache is designed to hold only a fraction of the data in a single burst, meaning that as the input grows larger, the CPU must wait longer for the next chunk of data to be loaded into its cache before it can begin processing the current sequence. This sequential nature of data access means that the CPU cannot utilize parallelism to speed up the decoding, and the overall throughput is
---
Q: What is the capital of Australia, and what is 17 * 23?
A: Here are the answers to your questions:

1.  **Capital of Australia**: The capital city of Australia is **Canberra**. It is located in the central part of the country, just south of the Great Australian Bight.

2.  **Mathematical Calculation**:
    To find the result of $17 \times 23$, you can use the distributive property or standard multiplication:
    $$17 \times 23 = 17 \times (20 + 3)$$
    $$= 17 \times 20 + 17 \times 3$$
    $$= 340 + 51$$
    $$= 391$$

**Final Answer:**
*   Capital
---
Q: Viết 2 câu giới thiệu về Hà Nội bằng tiếng Việt.
A: Hà Nội là thành phố lớn nhất của Việt Nam, nổi tiếng với lịch sử lâu đời và văn hóa phong phú, nơi có nhiều di tích lịch sử quan trọng như Cố đô, chùa Bái Bội Châu và các công trình kiến trúc đẹp mắt.
---
```

## Output thô — UD-Q2_K_XL

```text
Q: Explain in 3 sentences why LLM decoding on a CPU is limited by memory bandwidth.
A: The limitation of LLM decoding on a CPU is constrained by memory bandwidth because the vast majority of the model's input data is already processed by the GPU's memory bandwidth. This means that even if the GPU can process thousands of tokens per second, the bottleneck occurs when the GPU's memory bandwidth is saturated, causing the model to stall and outputting the wrong answer.

This limitation creates a bottleneck where the GPU's memory bandwidth is the limiting factor, causing the model to stall and output the wrong answer.

The limitation of LLM decoding on a CPU is limited by memory bandwidth because the vast majority of the model's input data is already processed by the GPU's memory bandwidth. This means that even if the GPU can process thousands of tokens per second, the bottleneck occurs when the GPU's memory bandwidth
---
Q: What is the capital of Australia, and what is 17 * 23?
A: The capital of Australia is **Sydney**.

The capital of the United Kingdom is **London**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 * 23** equals **381**.

**17 *
---
Q: Viết 2 câu giới thiệu về Hà Nội bằng tiếng Việt.
A: Hà Nội là một trong những thành phố lớn nhất của Việt Nam, với một lịch sử phát triển rất dài và rộng lớn.

Đúng theo quy mô, Hà Nội được xây dựng như một đô thị lớn, với các khu vực lớn nhất trên toàn quốc.

Đặc trưng của Hà Nội là sự kết hợp giữa văn hóa và công nghiệp.

Đặc trưng của Hà Nội là sự kết hợp giữa văn hóa và công nghiệp.

Đặc trưng của Hà Nội là sự kết hợp giữa văn hóa và công nghiệp.

Đặc trưng của Hà Nội là sự kết hợp giữa văn hóa và công nghiệp.

Đặc trưng của Hà Nội là sự kết hợp giữa văn hóa và công nghiệp.

Đặc trưng của Hà Nội là sự kết hợp giữa văn hóa và công
---
```
