# Lab 25B — Report

**Họ tên / nhóm:** Vu Duy Diep — MSV: 2A202602703
**Commit:** `b3a80252435e7286bf3033d1df6209c3a2ab6e8a`
**Điểm tự động:** 84/88; 4 điểm SLO ẩn và 12 điểm report do giảng viên chấm.

Commit trên chứa code và kết quả kiểm chứng bài nộp. Báo cáo được bổ sung ở commit tiếp theo; một file không thể chứa hash của chính commit đang tạo ra nó.

## 1. Kết quả Game Day (naive vs resilient)

| Băng | Client | Availability | p95 (s) | Amplification | Retry ratio | Cost/turn | Recovery (s) | PASS |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | resilient | 1.0 | 4.78 | 1.063 | 0.001 | 2.104 | 0.0 | PASS |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | resilient | 1.0 | 9.847 | 1.037 | 0.005 | 2.105 | 0.0 | PASS |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | resilient | 1.0 | 4.718 | 1.065 | 0.035 | 2.146 | 0.0 | PASS |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | resilient | 1.0 | 5.684 | 1.013 | 0.002 | 2.069 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.03 | 16.03 | 2.124 | N/A | FAIL |
| G5 | resilient | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | N/A | PASS |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | resilient | 1.0 | 6.099 | 1.053 | 0.017 | 2.202 | 0.0 | PASS |

Nguồn: `gameday/naive.json`, `gameday/resilient.json`. Resilient có 0 vi phạm retry-after, 0 retry lỗi client và 0 route không tương thích trên cả 6 băng. G2 và G4 có primary_share_after_restore = 1,0. Recovery = 0 nghĩa là tỷ lệ thành công đã đạt tiêu chí ngay lúc provider phục hồi nhờ fallback; N/A ở G5 vì không có mốc phục hồi để đo.

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Lý do và bằng chứng |
|---|---:|---|
| attempt_timeout_s | 8 | G6 đạt availability 1,0 và p95 6,099 s; tăng lên 30 s làm availability còn 0,9488. |
| max_attempts_per_provider | 2 | Một retry tối đa; G3 retry_ratio 0,035, dưới ngưỡng 0,10. |
| min_time_left_s | 1 | Không cấp attempt khi deadline còn dưới 1 s; test deadline PASS. |
| backoff_base_s / cap_s | 0,25 / 4 | Full jitter tránh đồng bộ retry; G2 amplification 1,037; 21/21 test Part A PASS. |
| retry_budget_ratio / min_per_s / max_tokens | 0,10 / 0,5 / 20 | G3 retry_ratio 0,035; thí nghiệm breaker vô hiệu hóa cho thấy budget giảm retry_ratio từ 0,158 xuống 0,068. |
| quota_cooldown_s | 300 | G5 availability 0,8179, 0 retry lỗi 400 và không failover vô ích; billing được xử lý riêng, không coi là provider hỏng. |
| breaker.window_size / min_calls | 20 / 10 | G2 giảm amplification từ 9,904 xuống 1,037; tránh trip vì ít kết quả; 13/13 test breaker PASS. |
| breaker.failure_rate_threshold | 0,50 | G6 có availability 1,0; G3 vẫn đạt 1,0 với retry_ratio 0,035 khi lỗi chỉ 30%. |
| breaker.slow_call_s / slow_rate_threshold | 5 / 0,60 | G4 không cần lỗi HTTP vẫn phát hiện chậm; p95 giảm từ 13,764 xuống 5,684 s. |
| breaker.open_duration_s | 10 | G2 recovery 0 s và 100% traffic trở lại primary sau phục hồi; đạt yêu cầu ≤30 s. |
| half_open_max_probes / half_open_successes | 2 / 2 | Giới hạn request đang bay khi thăm dò; test 50 request đồng thời chỉ được 2 permit. |

Cấu hình client mặc định đã đạt toàn bộ tiêu chí nên giữ nguyên. Các số trên chứng minh cả bộ cấu hình; không khẳng định từng tham số riêng lẻ là tối ưu nếu chưa làm ablation.

| Loại lỗi | Breaker | Retry cùng provider | Failover | Cooldown |
|---|---|---|---|---|
| CLIENT_ERROR | Release permit, không ghi lỗi | Không | Không, GiveUp | Không |
| QUOTA_EXHAUSTED | Release permit | Không | Ngay | 300 s |
| RATE_LIMITED | Release permit | Không retry ngay | Ngay | retry-after (1 s nếu thiếu) |
| OVERLOADED / SERVER_ERROR / TIMEOUT | Ghi failure | Chỉ khi còn lượt, budget và deadline; full jitter | Khi retry không khả dụng | Không |

Route lọc tool capability, bỏ provider đang cooldown hoặc breaker từ chối. Nếu mọi provider hết khả dụng, chỉ chờ cooldown ngắn ≤5 s và còn đủ deadline. Permit mang generation giúp bỏ qua kết quả cũ; on_cancel giải phóng probe.

## 3. Ba thí nghiệm

Nguồn: `experiments.json`; mọi thí nghiệm dưới đây dùng seed 25.

### 3.1. Retry budget khi breaker cấu hình sai

Đặt breaker.min_calls = 1.000.000.000 và max_attempts_per_provider = 5. Giữ budget mặc định hoặc tắt bằng ratio=100, max_tokens=1e9, min_per_s=1e6.

| Băng | Budget | Availability | Amplification | Retry ratio | Recovery (s) |
|---|---|---:|---:|---:|---:|
| G2 | budget_on | 0.9983 | 1.378 | 0.058 | 0.0 |
| G3 | budget_on | 1.0 | 1.136 | 0.068 | 0.0 |
| G2 | budget_off | 0.9754 | 1.58 | 0.378 | 24.0 |
| G3 | budget_off | 1.0 | 1.162 | 0.158 | 0.0 |

Ở G2, budget giảm amplification 1,580 → 1,378 và recovery 24 → 0 s khi breaker không hoạt động. Ở G3, budget giảm retry_ratio 0,158 → 0,068, giúp đạt giới hạn 0,10. Khi breaker hoạt động đúng, G2 amplification còn 1,037 và retry_ratio 0,005; breaker cắt provider không khỏe, còn budget giới hạn lượng retry trước khi đủ mẫu hoặc khi breaker cấu hình sai. Hai lớp bảo vệ bổ sung nhau.

### 3.2. Timeout 30 giây

| Băng | Timeout | Availability | p95 của lượt thành công (s) |
|---|---:|---:|---:|
| G4 | 8 | 1.0 | 5.684 |
| G4 | 30 | 1.0 | 5.608 |
| G6 | 8 | 1.0 | 6.099 |
| G6 | 30 | 0.9488 | 5.362 |

G4 vẫn được breaker phát hiện slow success nên tăng timeout không làm p95 tăng đáng kể. G6 có region treo; timeout 30 s tiêu hết deadline của một số lượt trước khi có thể fallback, khiến availability 1,0 → 0,9488 và FAIL ngưỡng 0,95. p95 chỉ tính trên lượt thành công nên có thể giảm dù trải nghiệm tổng thể tệ hơn. Timeout cần dựa trên phân bố latency của provider, deadline toàn lượt và thời gian cần dành cho fallback, thay vì dùng toàn bộ 30 s cho một attempt.

### 3.3. Bỏ lọc tool calling

Chỉ thay ladder trong đối tượng thí nghiệm bằng toàn bộ thứ tự provider; không sửa code nộp. G6 có 235 incompatible_routes và availability giảm 1,0 → 0,9015. Theo góc nhìn sản phẩm, model rẻ có thể viết câu trả lời nhưng không thực hiện được bước gọi tool cần cho hoàn tiền; chuyển yêu cầu sang nó làm mất chức năng chứ không cứu được lượt. Bản nộp lọc capability và có 0 incompatible_routes.

## 4. Workflow hoàn tiền

| Executor | Completion | Crashes | Refund trùng | Email trùng | Refund mất | Overpaid (cent) | Wasted LLM/crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1574200 | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Nguồn: `workflows.json`, 500 session. Idempotency key ổn định theo session và step ngăn payment/email thực thi lại khi response mất hoặc worker crash sau commit. Checkpoint chỉ ghi sau khi step thành công và lưu result để worker mới bỏ qua step đã làm. Chỉ có key thì vẫn phải lặp lại LLM và các bước đọc; chỉ có checkpoint thì khoảng trống giữa payment commit và checkpoint vẫn có thể tạo refund trùng. Ghi checkpoint trước payment có thể khiến worker chết rồi resume bỏ qua khoản tiền chưa chuyển, làm mất refund. WorkerCrash được truyền ra runner, chỉ ServiceError/ServiceTimeout được retry tối đa 5 lần.

## 5. Alerting và error budget

| Sự cố | Mong đợi | Naive | Burn-rate | Độ trễ burn-rate (phút) |
|---|---|---|---|---:|
| checkout_bug | page | Page | Page | 27 |
| provider_outage | page | Page | Page | 4 |
| deploy_blip | none | Page sai | Không page | — |
| slow_leak | ticket | Bỏ sót | Ticket | 1337 |
| degraded_evening | page | Page | Page | 291 |
| night_spike | none | Page sai | Không page | — |
| small_bump | none | Page sai | Không page | — |

Nguồn: `slo.json`. Burn-rate tạo 3 page, 1 ticket và 0 false page; naive có 7 page, 0 ticket và 3 false page. Tổng 25,901,755 request, 55,618 lỗi, budget cho phép 129,508.8 lỗi. Đã tiêu 42,95%, còn 57,05%; availability 99,7853%, không đóng băng release.

Prefix sum xây dựng một lần, mỗi phép tính cửa sổ O(1). Mỗi rule yêu cầu đồng thời cửa sổ dài và ngắn đạt burn rate; mỗi đợt liên tục chỉ sinh một event. small_bump ở 12% lỗi trong 20 phút tương đương burn 24× lúc đang xảy ra, nhưng trung bình 1h chỉ khoảng 8× nếu phần còn lại không lỗi, dưới ngưỡng page 14,4×. Số liệu thực tế xác nhận không page; chỉ ngưỡng 5 phút của naive lại tạo false page.

## 6. Postmortem — G2 với client naive (blameless)

### Tóm tắt

Alpha mất 75% năng lực từ t=120 đến t=300 s. Client naive có ba tầng retry, không breaker/fallback/budget, khuếch đại 3.481 logical call thành 34.475 attempt và availability chỉ đạt 45,6%. Provider hồi phục lúc t=300 nhưng hệ thống cần thêm 155 s để đạt tiêu chí recovery vì hàng đợi và retry tiếp tục duy trì quá tải.

### Ảnh hưởng

Có 1.285/2.362 lượt thất bại trong băng 600 s; p95 của lượt thành công 15,034 s. Giai đoạn t=140–449 có các cửa sổ tỷ lệ call thành công 0%; recovery được grader ghi nhận tại t=455, tức 335 s sau trigger, không phải outage provider kéo dài 335 s. Giả sử tháng 30 ngày chạy đều 4 lượt/s, không có sự cố khác và SLO user-turn 99,5%, tổng 10.368.000 lượt có budget 51.840 lượt lỗi; sự cố tiêu 1.285/51.840 = 2,48% budget tháng. Giả định traffic đều dùng để ước tính tháng, tách biệt với dataset SLI Part E.

![G2: tải attempt và tỷ lệ call thành công](gameday/G2_overload_storm.png)

### Timeline

Số dưới đây tính trực tiếp từ `gameday/naive_G2_overload_storm_timeseries.csv`: cửa sổ 10 s bắt đầu tại t, attempts/s là tổng attempts_alpha chia 10, success ratio là tổng calls_ok / tổng calls_done. Đây là tỷ lệ logical call, khác availability theo user turn.

| t (s) | Sự kiện/quan sát | Attempts/s | Call OK / done | Success ratio |
|---:|---|---:|---:|---:|
| 110 | Trước sự cố, hệ thống ổn định | 9.1 | 92/92 | 100.0% |
| 120 | Alpha mất 75% năng lực; kết quả đang bay còn thành công | 6.2 | 32/32 | 100.0% |
| 140 | Retry storm hình thành | 214.2 | 0/47 | 0.0% |
| 290 | Quá tải trước lúc phục hồi | 146.1 | 0/41 | 0.0% |
| 300 | Provider phục hồi; client vẫn không thành công | 38.8 | 0/18 | 0.0% |
| 400 | 100 s sau phục hồi vẫn bị retry giữ tải cao | 77.9 | 0/40 | 0.0% |
| 450 | Bắt đầu có lại call thành công | 13.7 | 45/70 | 64.3% |
| 480 | Cửa sổ đã thành công hoàn toàn | 10.6 | 124/124 | 100.0% |

### Trigger và root cause

Trigger là provider mất GPU trong 180 s. Root cause phía client là retry ba tầng (tối đa 36 attempt/call), delay cố định không jitter, không tôn trọng retry-after, không retry budget, breaker hay fallback. Bản naive có 2.737 client_timeout, 3.194 overloaded_error và 26.298 rate_limit_exceeded; 28.381 vi phạm retry-after. Đây là thiếu cơ chế bảo vệ và kiểm chứng sự cố trong thiết kế, không quy trách nhiệm cá nhân.

Khi client timeout, request cũ vẫn chiếm năng lực xử lý phía server. Retry tạo request mới → hàng đợi dài hơn → timeout tăng → retry tiếp tục tăng; tải này tự duy trì cả khi GPU đã hồi phục. Tại t=400, client còn gửi 77,9 attempt/s trong khi trước sự cố ở t=110 chỉ 9,1 attempt/s; success ratio vẫn 0%. Đây là vòng lặp metastable failure được số liệu mô phỏng hỗ trợ.

### Điều giúp và điều làm tệ hơn

Provider phục hồi năng lực và hàng đợi cuối cùng được giải tỏa giúp hệ thống trở lại. Retry liên tục, bỏ qua retry-after và không chuyển provider kéo dài sự cố. Resilient sử dụng breaker, cooldown, budget và fallback: cùng 2.362 lượt đều thành công, amplification 1,037 và recovery 0 s; cost/good turn 2,105 thay vì 4,627.

### Action items

| # | Hành động | Loại | Owner (vai trò) | Kiểm chứng |
|---|---|---|---|---|
| 1 | Dùng một tầng retry với jitter, retry-after và budget | Prevent | Nhóm SDK/client | CI chạy G1/G3 nhiều seed: 0 retry-after violation, retry_ratio ≤0,10 ở G3. |
| 2 | Breaker phát hiện lỗi và call chậm, probe giới hạn, fallback lọc capability | Mitigate | Nhóm ShopAgent | G2 amplification ≤1,2, recovery ≤30 s; G4 p95 ≤10 s; G6 0 incompatible route. |
| 3 | Theo dõi attempt/logical call, latency, rate-limit và burn-rate | Detect | SRE/on-call | Dataset SLO: 0 false page; cảnh báo retry storm được diễn tập bằng G2. |
| 4 | Diễn tập provider phục hồi nhưng hàng đợi vẫn đầy trước release | Prevent | Nhóm QA/release | G1–G6 và C1 PASS ở ít nhất ba seed; lưu CSV/JSON để review. |

## 7. Một điểm yếu còn lại

G2 đạt availability 100% nhưng p95 vẫn 9,847 s, p99 15,586 s; các lượt đầu gặp queue/timeout phải chờ trước khi fallback. Nếu cả bốn provider cùng chậm, fallback không còn nhiều giá trị; cache chỉ giúp FAQ công khai, không cứu workflow gọi tool hoặc dữ liệu cá nhân. Có thể bổ sung giới hạn concurrency và load shedding theo deadline, rồi đo lại latency, availability và chi phí dưới tải; không thêm retry vô hạn.

## 8. Cache (Part G)

| Client | Availability | FAQ availability | Hit ratio | Stale served | Wrong answers | Outdated answers | Cost/turn | PASS |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.0 | 0 | 0 | 0 | 1.386 | FAIL |
| cached | 0.912 | 0.964 | 0.652 | 156 | 0 | 3 | 0.916 | PASS |

Nguồn: `cache/resilient.json`, `cache/cached.json`. Client không cache FAIL tiêu chí của băng cache là kết quả đối chứng mong đợi. Bản cached PASS mọi check ở seed 25, 26, 27. Cache chỉ giữ Answer do provider tạo; loại câu cá nhân, giới hạn cùng prompt_version, cosine threshold, LRU, TTL, invalidate theo source. Chỉ phục vụ stale sau GiveUp và lookup lại lúc đó để tôn trọng invalidation mới nhất.

| Tham số | Giá trị nộp | Lý do |
|---|---:|---|
| similarity_threshold | 0,90 | 0,80 tạo 157 câu sai dù TTL/stale đã chỉnh; 0,90 tạo 0 câu sai với hit 65,2%. |
| ttl_s | 60 | TTL 300 tạo 95 câu lỗi thời; TTL 60 chỉ 3 và hit 65,2%, cao hơn 59,1% ở TTL 30. |
| max_stale_s | 180 | FAQ availability 96,4% thay vì 87,33% khi tắt stale, phục vụ 156 câu stale. |
| max_entries | 500 (giữ mặc định) | Giới hạn LRU; toàn bộ 17 test cache PASS. |

Cấu hình starter 0,80 / 600 / 0 tạo 181 wrong_answers (other_question), 80 outdated_answers và FAQ availability 85,87%. Chỉ ba tham số trên được đổi; không hard-code băng, thời điểm hay tài liệu trong cache.

### 8.1. Ngưỡng similarity

Giữ TTL=60, stale=180; chỉ thay threshold.

| Threshold | Hit ratio | Wrong answers | Cost/turn |
|---:|---:|---:|---:|
| 0.8 | 0.666 | 157 | 0.966 |
| 0.9 | 0.652 | 0 | 0.916 |
| 0.97 | 0.58 | 0 | 0.965 |

Ngưỡng 0,80 nhầm các câu có embedding gần nhau nhưng ý khác (wrong_answers_by_reason = other_question). Catalog mô phỏng có các cặp gần mà khác như “Chính sách hoàn tiền của shop thế nào?” / “Bao lâu thì nhận được tiền hoàn?”, đổi size / đổi màu, và phí ship nội thành / phí ship đi tỉnh (`sim/faq.py`). Báo cáo metric chỉ gộp lỗi vào other_question, không cung cấp số riêng cho từng cặp. Không có other_user, old_prompt hoặc ignored_invalidation vì code đã kiểm soát scope và webhook. Ngưỡng 0,97 vẫn an toàn nhưng hit giảm 65,2% → 58,0% và cost tăng 0,916 → 0,965; chọn 0,90 theo kết quả mô phỏng, cần đánh giá lại trên dữ liệu thật trước production.

### 8.2. TTL

Giữ threshold=0,90, stale=180; chỉ thay TTL.

| TTL (s) | Hit ratio | Outdated answers | FAQ availability |
|---:|---:|---:|---:|
| 30 | 0.591 | 2 | 0.9634 |
| 60 | 0.652 | 3 | 0.964 |
| 300 | 0.731 | 95 | 0.964 |

Webhook chỉ giúp những thay đổi được thông báo. Bảng phí ship đổi im lặng nên không có sự kiện để invalidate; TTL giới hạn thời gian dùng câu trả lời cũ trong luồng bình thường. Nếu lỗi provider xảy ra đồng thời với thay đổi im lặng, stale-if-error có thể kéo dài tuổi câu trả lời đến TTL + max_stale_s; đây là đánh đổi cần ghi rõ.

### 8.3. Tắt stale-if-error

Giữ threshold=0,90 và TTL=60, đặt max_stale_s=0: FAQ availability giảm 0,964 → 0,8733, hit ratio 0,652 → 0,561, stale_served 156 → 0 và cost/turn 0,916 → 0,975. Cửa sổ stale 180 s cho phép dùng lại FAQ an toàn trong outage 120 s mà vẫn invalidate nội dung có webhook và tách prompt version.

### Câu hỏi

- Câu hỏi cá nhân phụ thuộc trạng thái mới của đơn hàng, refund hoặc điểm; cache theo user vẫn có thể trả thông tin cũ và embedding gần nhau chưa chắc cùng đơn. Lab yêu cầu loại hoàn toàn câu cá nhân; với sản phẩm thật cần cơ chế truy xuất và freshness riêng trước khi cân nhắc cache.
- Cache giúp SLI user vẫn tốt khi provider hỏng; cần theo dõi riêng lỗi/timeout của attempt gửi upstream, trạng thái breaker, fallback share, stale_served và tuổi câu trả lời. Page theo health/burn-rate upstream bên cạnh SLO user để on-call thấy outage dù FAQ đang được phục vụ từ cache.

## Kiểm chứng và cách chạy lại

- 82/82 unit test PASS; G1–G6, workflow và C1 PASS trên ba seed 25, 26, 27; SLO public PASS.
- `reports/scoreboard.json`: 84/88; 4 điểm hidden chưa có dataset để chấm. Không dùng số public để khẳng định kết quả hidden.
- Ruff kiểm tra các file TODO PASS khi bỏ RUF100. Năm cảnh báo RUF100 của `noqa: F401` có sẵn được giữ nguyên theo yêu cầu không sửa code mặc định.
- Môi trường `.venv` ban đầu có Python trampoline hỏng; dùng `.lab-venv` và Python riêng `.lab-python` để kiểm chứng, không sửa môi trường ban đầu.

```powershell
.\.lab-venv\Scripts\python.exe -m pytest -q -p no:cacheprovider
.\.lab-venv\Scripts\python.exe scripts/run_gameday.py --client both
.\.lab-venv\Scripts\python.exe scripts/run_gameday.py --scenarios configs/scenarios_cache.yaml --client resilient,cached --out reports/cache
.\.lab-venv\Scripts\python.exe scripts/run_workflows.py --executor both
.\.lab-venv\Scripts\python.exe scripts/run_slo.py --alerter both
.\.lab-venv\Scripts\python.exe scripts/grade.py --seeds 25,26,27
```

Lưu ý khi nộp bằng git: `.gitignore` mặc định loại `reports/*.json`, `reports/gameday/` và `reports/cache/`; cần đưa scoreboard, hình và dữ liệu báo cáo vào gói nộp hoặc add force có chủ đích. Không cần nộp môi trường Python.
