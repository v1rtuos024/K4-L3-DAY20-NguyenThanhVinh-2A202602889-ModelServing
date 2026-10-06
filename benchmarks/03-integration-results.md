# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 11021.9 | 11022.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 5952.6 | 5952.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8458.1 | 8458.2 |

Mean per stage (ms): embed **0.0** � retrieve **0.1** �
llm **8477.5** � total **8477.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it accounts for **SLOs** (Service Level Objectives) and **TPOT** (Throughput at Saturation).

Here is the breakdown of why this makes goodput superior:

*   **Goodput respects SLOs:** It counts only requests per second that met the Target Throughput and Time-to-Failure (TTFT) targets. This ensures the system meet

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, the model avoids the wasted space that would occur if all memory were contiguous, thereby freeing up more memory for the model's actual computation.

**When does splitting prefill and decode help?**

> Based on the context provided, splitting prefill and decode helps primarily to **avoid redundant computation and memory bandwidth usage**.

Here is the breakdown of why this is the case:
1.  **Prefill is compute-bound**: It requires significant processing power (CPU/GPU) to generate the initial token embeddings.
2.  **Decode is memory-bandwidth-bound**: It requires significant memory bandwidth to 


## Which N16-N19 pieces are real

* **N16 (Cloud / IaC):** **Stubbed** — chạy trực tiếp trên máy local, không triển khai hạ tầng cloud.
* **N17 (Data pipeline):** **Stubbed** — sử dụng dữ liệu tĩnh định nghĩa sẵn trong bộ nhớ (`TOY_DOCS`).
* **N18 (Lakehouse):** **Stubbed** — không có hệ thống lưu trữ lakehouse ngoài; context lưu trong Python list.
* **N19 (Vector + features):** **Stubbed** — không dùng vector DB hay embedding service; fallback sang keyword overlap đơn giản (`embed = 0.0 ms`, `retrieve = 0.1 ms`).
*(N20 Serving là **real**, chạy inference qua `llama-server`.)*

### Reflection

* **Giai đoạn chiếm ưu thế có đúng kỳ vọng?**
  **Đúng như kỳ vọng.** LLM chiếm **8,477.5 ms (100.0% tổng latency)**. Do retrieval là tìm kiếm từ khóa trong bộ nhớ chỉ mất 0.1 ms, quá trình sinh token tự hồi quy tuần tự trên CPU/GPU local hoàn toàn chi phối thời gian xử lý. Ngay cả trong production RAG có vector DB thực tế (10–50 ms), giai đoạn LLM decode vẫn luôn là bottleneck chính.

* **Nếu phải giảm latency 2×, sẽ tấn công vào đâu và vì sao?**
  Tôi sẽ tấn công duy nhất vào **giai đoạn LLM** (theo định luật Amdahl, tối ưu bước retrieval vốn chỉ mất 0.1 ms sẽ không mang lại bất kỳ cải thiện đáng kể nào):
  1. **Giới hạn số token sinh ra (`max_tokens` / prompt engineering):** Thời gian decode tỉ lệ thuận tuyến tính với số lượng token đầu ra; giảm một nửa độ dài câu trả lời sẽ giảm gần một nửa tổng thời gian.
  2. **Prompt caching / Prefix caching:** Tái sử dụng KV cache cho system prompt và context chung để loại bỏ thời gian prefill trùng lặp.
  3. **Tối ưu phần cứng / Speculative decoding:** Sử dụng model draft nhỏ hơn hoặc kernel tối ưu để tăng tốc độ decode tok/s.
