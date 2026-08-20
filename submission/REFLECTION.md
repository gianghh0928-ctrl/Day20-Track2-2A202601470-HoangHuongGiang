# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Hoàng Hương Giang
**Cohort:** Track 2
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux 6.6.87.2-microsoft-standard-WSL2 (Ubuntu on WSL2, x86_64)
- **CPU:** AMD Ryzen 5 7520U with Radeon Graphics
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2
- **RAM:** 5.8 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** prebuilt release b10488 (llama-b10488-bin-ubuntu-x64.zip)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (WSL2)

**Setup story** (≤ 80 chữ): Ban đầu khi kiểm tra setup thì WSL2 chỉ nhận 2.8 GB RAM và thiếu python3-venv. Tôi đã sửa file .wslconfig để cấp 6GB RAM cho WSL, sau đó cài python3-venv rồi chạy make setup tải model và runtime tự động.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 16155 | 599 / 872 | 56.7 / 66.8 | 3876 / 4944 / 4944 | 17.6 |
| UD-Q2_K_XL | 0.39 | 11548 | 880 / 1339 | 57.1 / 73.8 | 4479 / 5620 / 5620 | 17.5 |

**Quan sát** (≤ 60 chữ): Bản 2-bti chỉ giúp giảm được một chút kích thước khoảng 0,11GB và từ đó rút ngắn thời gian load model. Tuy nhiên, TTFT của bản 2 bit lại chậm hơn hẳn so với bản 4 bit. 
Đối với model nhỏ như 0,8B thì việc sử dụng bản 2 bit dường như không mang lại hiệu quả. Việc đánh đổi độ chính xác và TTPT để tiết kiệm 110 MB RAM là không đáng. 

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.44 | 23000 | 30000 | 30000 | 9.2 | 0.0% |
| 50 | 0.36 | 25000 | 55000 | 55000 | 8.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.81×
- **P95 tăng:** 1.83×
- **Effective concurrency ở 50 users:** 8.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.82 / 4 slots

**Saturation reading** (≤ 80 chữ): Hệ thống đã bão hòa trước mức 50 users: 
- Khi tải tăng 5x: thông lượng không tăng mà giảm nhẹ. 
- Độ trễ P95 tăng lên: Chủ yếu là Queue time do 4 slots xử lý đã lấp đầy và có tới 46 request bị hoãn. 
Để nâng cao goodpu, tôi sẽ điều chỉnh là:
- Đặt thread = 2 để tốc độ decode nhanh nhất. 
- Cấu hình cơ chế từ chối tải hoặc giới hạn hàng đợi. 


---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local simulated environment | stub |
| N17 Data pipeline | In-memory document dataset | stub |
| N18 Lakehouse | Dummy lakehouse store | stub |
| N19 Vector + features | In-memory keyword overlap | stub |
| N20 Serving | `llama-server` (Qwen3.5 0.8B) | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 8218.9 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): Giai đoạn tốn thời gian nhất là LLM generation (8.2s, 100%), đúng kỳ vọng vì retrieval chỉ là so khớp từ khóa in-memory, còn LLM inference trên CPU phải prefill context và sinh token. Nếu cần giảm 2x độ trễ, tôi sẽ dùng Prompt Caching cho context cố định và lượng tử hóa KV cache.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Hạ số luồng CPU từ 4 luồng (-t 4) xuống 2 luồng (-t 2) theo kết quả sweep tối ưu

```
before:  20.3 tok/s (ở -t 4)
after:   22.0 tok/s (ở -t 2)
speedup: 1.08x (và 9.17x so với -t 16)
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Quá trình sinh từ (decode) của mô hình LLM chủ yếu bị giới hạn bởi băng thông bộ nhớ RAM chứ không phải sức mạnh tính toán. Với mô hình nhỏ Qwen3.5 0.8B chạy trên CPU Ryzen 5 7520U, chỉ cần 2 luồng là đã đủ để khai thác tối đa băng thông bộ nhớ và bộ nhớ đệm L3 Cache (4MB).

Khi tăng lên 4 luồng vật lý hay 8, 16 luồng logic, hệ thống sẽ xảy ra hiện tượng tranh chấp bộ nhớ đệm (cache thrashing), nghẽn bus bộ nhớ và chi phí chuyển đổi ngữ cảnh (context switching) giữa các luồng logic. Việc này khiến CPU mất nhiều thời gian để đồng bộ hơn là xử lý tính toán, dẫn đến hiệu năng bị sụt giảm mạnh mẽ (ở 16 luồng tốc độ chỉ còn 2.4 tok/s, chậm hơn 9.17 lần so với 2 luồng). Vì vậy, đặt 2 luồng mang lại tốc độ tối ưu nhất trên máy của tôi.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:**

**Numbers:**

```
before:
after:
speedup:
```

**Điều này nói lên gì mà deck chưa nói:**



---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm tôi ngạc nhiên nhất là việc tăng số luồng CPU (threads) quá nhiều không giúp mô hình chạy nhanh hơn mà ngược lại làm tốc độ tụt hơn 9 lần do bị nghẽn băng thông bộ nhớ và tranh chấp cache L3.

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
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo GitHub ở chế độ **public**
- [x] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
