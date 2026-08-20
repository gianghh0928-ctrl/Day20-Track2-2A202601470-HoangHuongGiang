# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 16155 | 599 / 872 | 56.7 / 66.8 | 3876 / 4944 / 4944 | 17.6 |
| UD-Q2_K_XL | 0.39 | 11548 | 880 / 1339 | 57.1 / 73.8 | 4479 / 5620 / 5620 | 17.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Your observation
Bản 2-bti chỉ giúp giảm được một chút kích thước khoảng 0,11GB và từ đó rút ngắn thời gian load model. Tuy nhiên, TTFT của bản 2 bit lại chậm hơn hẳn so với bản 4 bit. 
Đối với model nhỏ như 0,8B thì việc sử dụng bản 2 bit dường như không mang lại hiệu quả. Việc đánh đổi độ chính xác và TTPT để tiết kiệm 110 MB RAM là không đáng. 
