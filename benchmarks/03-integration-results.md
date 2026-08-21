# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 13103.5 | 13103.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 6567.3 | 6567.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 4985.9 | 4986.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **8218.9** · total **8219.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput is more useful than raw throughput** because it focuses on the specific metrics that directly influence system performance and resource utilization.

Here is the breakdown based on the text:

*   **Goodput** counts only requests per second that met the **TTFT (Time to First Fill)** and **TPOT (Time to Prefill)** targets.
*   **Throughput at saturation** ign

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value pairs in non-contiguous pages.

Specifically, it addresses the issue where most GPU memory is wasted due to the internal fragmentation of contiguous memory blocks. By storing keys and values in non-contiguous pages, PagedAttention allows the engine to utilize more of the available memory, effe

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context explicitly states that prefill is compute-bound and decode is memory-bandwidth-bound. By splitting them into separate pools, the system avoids the memory bandwidth bottleneck of the decode operation, allowing it to proceed efficiently.


## Which N16-N19 pieces are real
Khai báo các thành phần trong Pipeline:
- **N16 Cloud/IaC:** Stub (môi trường local giả lập)
- **N17 Data pipeline:** Stub (danh sách documents cố định)
- **N18 Lakehouse:** Stub
- **N19 Vector + features:** Stub (sử dụng thuật toán so khớp từ khóa keyword overlap)
- **N20 Serving:** **Real** (`llama-server` thật phục vụ mô hình Qwen3.5 0.8B)
Giai đoạn chiếm nhiều thời gian là LLM generation. Kết quả này khớp với kỳ vọng vì retrieval là in-memory search nhanh, LLM inference trên CPU lại phải prefill context dài và sinh token tự hồi quy.
Để giảm một nửa độ trễn, tôi sẽ áp dụng Prompt Caching cho system prompt/context cố định, lượng tử háo KV cache, chạy ở số thread tối ưu là 2 luồng.
