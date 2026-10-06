# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Thành Vinh
**MSSV:** 2A202602889
**Cohort:** K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 AMD64
- **CPU:** AMD Ryzen 7 5825U with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 13.8 GB
- **Accelerator:** Vulkan (AMD Radeon Graphics)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
*(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)*

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Lab chạy trực tiếp trên Windows 11. Hệ thống tự động tải bản prebuilt llama.cpp b10488 Vulkan cho Windows x64. Backend Vulkan nhận diện chính xác iGPU AMD Radeon Graphics và offload toàn bộ 99 layers (`ngl=99`) mà không gặp lỗi biên dịch hay thiếu thư viện.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3326 | 778 / 861 | 30.3 / 31.4 | 2669 / 2700 / 2700 | 33.0 |
| UD-Q2_K_XL | 0.39 | 3186 | 808 / 877 | 29.9 / 30.9 | 2715 / 2778 / 2778 | 33.5 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit chỉ tiết kiệm 0.11 GB disk và decode nhanh hơn vỏn vẹn ~1.5% (33.0 lên 33.5 tok/s), nhưng TTFT lại chậm hơn. Hoàn toàn không đáng đổi vì chất lượng suy luận của bản 2-bit bị suy giảm rõ rệt, câu trả lời dễ lặp từ và kém mạch lạc.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.34 | 20000 | 34000 | 34000 | 7.1 | 0.0% |
| 50 | 0.38 | 35000 | 55000 | 58000 | 11.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.12×
- **P95 tăng:** 1.62×
- **Effective concurrency ở 50 users:** 11.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.26 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà ở ≤ 10 users: throughput chỉ tăng 1.12× khi tải tăng 5×, và effective concurrency đạt 11.9 (gấp 3 lần 4 slots). Latency tăng 1.62× thuần túy là queue time vì requests_deferred lên 46 trong khi 4 slots luôn bận (3.26/4). Để nâng goodput@SLO, tôi tăng `--parallel` lên 8 trước tiên để mở thêm slot giải phóng hàng đợi.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local host | stub |
| N17 Data pipeline | In-memory TOY_DOCS | stub |
| N18 Lakehouse | List/dict memory | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | llama-server | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 8477.5 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck tuyệt đối (100% latency, 8.48s), đúng như kỳ vọng vì sinh token tuần tự bị nghẽn compute/memory bandwidth trong khi retrieval chỉ mất 0.1ms. Để giảm latency 2×, tôi sẽ tấn công vào LLM bằng cách giới hạn max output tokens (giảm 2× số token sinh) và bật prompt caching.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Quét số lượng CPU threads từ -t 4 lên -t 16 khi chạy ngl=99

```
before:  34.3 tok/s (-t 4)
after:   37.1 tok/s (-t 16)
speedup: 1.08×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Kết quả đo sweep thread (`make tune`) cho thấy đường cong tốc độ sinh từ (`tg128`) gần như đi ngang trên toàn dải từ 1 đến 32 threads (chênh lệch giữa điểm thấp nhất và cao nhất chỉ 1.08×). Điểm knee xuất hiện nhẹ tại `-t 8` (số nhân vật lý) và đạt đỉnh ở `-t 16` (số luồng logic), nhưng mức tăng là không đáng kể so với kỳ vọng tăng tốc trên CPU thông thường.

Cơ chế đằng sau hiện tượng này là do tham số `ngl=99` đã chuyển toàn bộ 28 layers tính toán của mô hình sang GPU thông qua backend Vulkan. Khi các ma trận trọng số và phép tính attention được thực thi trực tiếp trên GPU, CPU chỉ đảm nhận việc tokenize đầu vào, dispatch kernel và sampling nhẹ. Nút thắt hiệu năng nằm ở băng thông bộ nhớ của iGPU chứ không nằm ở số lõi CPU; do đó tăng số luồng CPU không đem lại cải thiện thông lượng rõ rệt.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** 

**Numbers:**

```
before:  
after:   
speedup: 
```

**Điều này nói lên gì mà deck chưa nói:**

*(để trống nếu bạn không làm phần này)*

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Mức độ ảnh hưởng của hàng đợi (queue time) lên P95: khi server hết slot `--parallel`, thời gian chờ đợi trong hàng đợi tăng vọt và chiếm gần như toàn bộ độ trễ người dùng cảm nhận, minh họa trực quan sự khác biệt giữa raw throughput và goodput@SLO.

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
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng AI Assistant (Antigravity) để hỗ trợ phân tích định luật Little, giải thích cơ chế continuous batching và rà soát định dạng báo cáo.
