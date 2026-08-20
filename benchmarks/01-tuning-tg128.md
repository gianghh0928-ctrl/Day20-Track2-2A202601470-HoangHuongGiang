# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 14.2 | 65% |
| 2 | 22.0 | 100% |
| 4 | 20.3 | 92% |
| 8 | 7.8 | 35% |
| 16 | 2.4 | 11% |

**Best**: `-t 2` at 22.0 tok/s
**Slowest tested**: `-t 16` at 2.4 tok/s (9.27x spread)
**Against the physical-core default** (`-t 4`, 20.3 tok/s): 1.08x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Your explanation
Điểm tối ưu là tại 2 luồng với tốc độ 22.0 tok/s. Khi tăng lên thì hiệu năng sẽ giảm và nghiêm trọng hơn khi lên 8 hay 16. 
Nguyên nhân là do quá trình sinh từ của mô hình LLM chủ yếu dựa vào băng thông bộ nhớ. Với mô hình nhỏ thì 2 luồng đã đủ khai tháng hết băng thông. Khi nâng lên 4, 8, 16 thì sẽ xảy ra hiện tượng tranh chấp bộ nhớ đệm, nghẽn, giữa các luồng logic làm lấn thời gian tính toán => Sụt giảm hiệu năng mạnh mẽ. 
