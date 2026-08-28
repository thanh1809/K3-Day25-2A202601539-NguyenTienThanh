# Báo cáo Kỹ thuật Độ tin cậy Hệ thống (Reliability Engineering Report)

**Dự án:** LLM Agent Reliability Gateway & Resilience Engineering  
**Tác giả:** Nguyễn Tiến Thành  
**Mã sinh viên:** 2A202601539  
**Ngày hoàn thành:** 27/08/2026  
**Môi trường thử nghiệm:** Python 3.13 / Docker Redis 7-Alpine / Pytest 9.1.1  

---

## 1. Tóm tắt Kiến trúc Hệ thống (Architecture Summary)

Hệ thống Gateway phân tán được thiết kế theo mô hình phòng thủ theo chiều sâu (Defense-in-Depth) với các lớp bảo vệ liên hoàn:

1. **Lớp Semantic Cache & Privacy Guard:**
   - Tra cứu ngữ nghĩa qua N-gram Cosine Similarity (kết hợp tách từ và 3-gram ký tự).
   - Kiểm duyệt quyền riêng tư chủ động (Privacy Guardrails) ngăn chặn việc cache dữ liệu nhạy cảm (mật khẩu, số tài khoản, số dư, SSN).
   - Cơ chế False-Hit Detection ngăn chặn việc trả về nhầm kết quả cho các câu hỏi cùng cấu trúc nhưng khác thông số năm/ID (ví dụ: hạn chót 2024 vs 2026).
   - Hỗ trợ cả **In-Memory Cache** (single-node) và **SharedRedisCache** (multi-instance cluster).
2. **Lớp Circuit Breaker State Machine:**
   - Máy trạng thái 3 giai đoạn chuẩn mực: `CLOSED` <-> `OPEN` <-> `HALF_OPEN`.
   - Cơ chế fail-fast ngắt mạch khi lỗi vượt ngưỡng `failure_threshold` giúp bảo vệ hệ thống hạ tầng và tránh retry storm.
   - Thử nghiệm thăm dò (probe) khi hết thời gian `reset_timeout_seconds` để tự động phục hồi khi provider ổn định trở lại.
3. **Lớp Routing & Fallback Chain:**
   - Ưu tiên Cache Hit -> Primary Provider -> Backup Provider -> Static Fallback (Degraded Response).

### Sơ đồ luồng xử lý (ASCII Architecture Diagram)

```
                       [ User Request / Agent Query ]
                                     |
                                     v
                       +---------------------------+
                       |   Reliability Gateway     |
                       +---------------------------+
                                     |
                                     v
                       +---------------------------+
                       |    Privacy Guard Check    |
                       +---------------------------+
                         /                       \
            (Sensitive) /                         \ (Safe to Cache)
                       v                           v
               [ Bypass Cache ]           +-------------------+
                       |                  |   Cache Lookup    |
                       |                  | (Memory / Redis)  |
                       |                  +-------------------+
                       |                    /               \
                       |            (Hit)  /                 \ (Miss)
                       |                  v                   v
                       |          [ Return Cached ]   +--------------------+
                       |          [ Score >= Thresh]  | Circuit Breaker A  |
                       |                              |  (Primary Provider)|
                       |                              +--------------------+
                       |                                /        \
                       |                       (Closed)/          \ (Open / Failed)
                       |                              v            v
                       |                      [ Provider A ]   +--------------------+
                       |                       (Success?)      | Circuit Breaker B  |
                       |                       /        \      |  (Backup Provider) |
                       |                (Yes) /          \(No) +--------------------+
                       |                     v            \       /        \
                       +--------------> [ Save Cache ]     \ (Closed)     \ (Open/Fail)
                                             |              v    v          v
                                             |        [ Provider B ]  +--------------------+
                                             |         (Success?)     |  Static Fallback   |
                                             |         /        \     | (Degraded Message) |
                                             |  (Yes) /          \(No)+--------------------+
                                             |       v            \          |
                                             +-> [ Return Resp ]   +---------> v
                                                 (Primary/Fallback)     [ Return 200 Degraded ]
```

---

## 2. Bảng thông số cấu hình (Configuration Table)

| Tham số cấu hình | Giá trị | Giải thích & Căn cứ kỹ thuật (Rationale) |
|---|---:|---|
| `failure_threshold` | `3` | Số lần thất bại liên tiếp để chuyển mạch từ CLOSED sang OPEN. Ngưỡng 3 đủ để lọc các lỗi mạng thoáng qua (network blip) nhưng không gây trễ lớn cho người dùng. |
| `reset_timeout_seconds` | `0.5s` | Khoảng thời gian mạch giữ ở trạng thái OPEN trước khi cho phép 1 request probe thử nghiệm ở HALF_OPEN. Đảm bảo provider có thời gian hồi phục và gateway nhanh chóng khôi phục luồng chính. |
| `success_threshold` | `1` | Số request thăm dò thành công ở HALF_OPEN để đóng mạch (CLOSED) hoàn toàn. |
| `cache.ttl_seconds` | `300s` | Thời gian sống (TTL) của bản ghi cache (5 phút), cân bằng giữa độ mới của dữ liệu LLM và tỷ lệ cache hit. |
| `cache.similarity_threshold` | `0.92` | Ngưỡng tương đồng cosine n-gram. Qua thực nghiệm, giá trị 0.92 cho phép bắt chính xác các câu hỏi diễn đạt tương tự mà loại bỏ 100% false-hit do sai lệch ngữ cảnh. |
| `load_test.requests` | `100` | Số lượng request mỗi kịch bản kiểm thử tải và chaos injection để có mẫu thống kê phân vị chuẩn xác. |

---

## 3. Định nghĩa và Đánh giá SLO (Service Level Objectives)

| Chỉ số (SLI) | Mục tiêu (SLO Target) | Kết quả thực tế | Đạt chuẩn (Met?) | Đánh giá kỹ thuật |
|---|---|---:|:---:|---|
| **Availability** | >= 99.0% (Kịch bản chuẩn) | **100.0%** (all_healthy) | **MET** | Không có lỗi nào xảy ra khi hệ sinh thái provider hoạt động ổn định. |
| **Latency P95** | < 2500 ms | **315.56 ms** | **MET** | Đạt thời gian phản hồi vượt mức kỳ vọng gấp ~8 lần so với trần SLO. |
| **Fallback Success Rate** | >= 95.0% (Khi Primary lỗi) | **100.0%** (primary_timeout_100) | **MET** | 100% lưu lượng chuyển mạch mượt mà sang Backup Provider mà không rớt request. |
| **Cache Hit Rate** | >= 10.0% | **43.75%** | **MET** | Tỷ lệ cache hit thực tế đạt rất cao, giúp tiết kiệm chi phí gọi LLM. |
| **Recovery Time** | < 5000 ms | **779.63 ms** | **MET** | Mạch tự động probe và khôi phục về CLOSED chỉ sau ~0.7-0.8 giây. |

---

## 4. Bảng số liệu Tổng hợp Metrics (Metrics Summary)

Dữ liệu được trích xuất trực tiếp từ `reports/metrics.json` và `reports/metrics.csv`:

| Chỉ số (Metric) | Giá trị thực nghiệm | Ý nghĩa & Phân tích |
|---|---:|---|
| `total_requests` | 400 | Tổng số lượng request chạy qua toàn bộ 4 kịch bản chaos simulation |
| `availability` | 74.25% | Tỷ lệ khả dụng tổng hợp (bao gồm cả kịch bản sập toàn bộ provider 100%) |
| `error_rate` | 25.75% | Tỷ lệ lỗi (chỉ xảy ra khi toàn bộ provider đồng loạt chết ở kịch bản cascade) |
| `latency_p50_ms` | 250.83 ms | Độ trễ trung vị (P50) toàn hệ thống |
| `latency_p95_ms` | 315.56 ms | Độ trễ phân vị 95th |
| `latency_p99_ms` | 319.37 ms | Độ trễ phân vị 99th (rất ổn định, không có đuôi trễ quá lớn) |
| `fallback_success_rate` | 37.20% | Tỷ lệ cứu vãn thành công qua Backup Provider |
| `cache_hit_rate` | 43.75% | Tỷ lệ câu hỏi được giải quyết trực tiếp từ Cache |
| `circuit_open_count` | 27 | Tổng số lần Circuit Breaker kích hoạt mở mạch để bảo vệ hệ thống |
| `recovery_time_ms` | 779.63 ms | Thời gian trung bình để mạch tự phục hồi từ OPEN về CLOSED |
| `estimated_cost` | $0.057160 | Tổng chi phí thực tế tiêu tốn cho LLM Providers |
| `estimated_cost_saved` | $0.175000 | Ước tính chi phí tiết kiệm được nhờ tầng Semantic Cache |

---

## 5. So sánh Hiệu năng: Có Cache vs Không có Cache (Cache Comparison)

Bảng đối chiếu thực nghiệm thực hiện trên cùng tập 400 request và cấu hình chaos:

| Tiêu chí đo lường | Không dùng Cache (No Cache) | Dùng Cache (With Memory Cache) | Dùng Redis Cache (Shared Redis) | Mức cải thiện (Delta) |
|---|---:|---:|---:|---|
| **Latency P50** | `267.06 ms` | `240.52 ms` | `273.33 ms` | **Giảm 26.54 ms (-10%)** khi dùng Memory Cache |
| **Latency P95** | `314.34 ms` | `309.07 ms` | `317.68 ms` | **Giảm 5.27 ms** |
| **Circuit Breaker Open Trips** | `72 lần` | `26 lần` | `25 lần` | **Giảm 63.9% số lần ngắt mạch** |
| **Chi phí tiêu thụ (Cost)** | `$0.129234` | `$0.055984` | `$0.040534` | **Tiết kiệm tới 68.6% chi phí** |
| **Tỷ lệ Cache Hit** | `0.00%` | `45.25%` | `69.00%` | **Tăng từ 0% lên tới 69%** |

> **Nhận xét quan trọng:** Tầng Cache không chỉ giúp giảm độ trễ và tiết kiệm chi phí gọi API LLM, mà còn đóng vai trò là "tấm đệm giảm chấn" (Shock Absorber) cực kỳ hiệu quả: số lần Circuit Breaker bị trip giảm từ 72 lần xuống chỉ còn 25-26 lần vì rất nhiều request đã được xử lý ở tầng cache trước khi chạm tới Provider.

---

## 6. Đánh giá Tầng Redis Shared Cache (Redis Shared Cache Evaluation)

### Tại sao In-Memory Cache là chưa đủ trong môi trường Production?
- Trong hệ thống microservices hoặc multi-replica (Kubernetes/Docker Swarm), mỗi instance gateway có bộ nhớ RAM độc lập.
- Cache trong RAM của Instance A sẽ không thể chia sẻ cho Instance B, dẫn đến hiện tượng **Cache Cold Start lặp lại**, lãng phí tài nguyên và chi phí gọi LLM trùng lặp.
- Khi instance khởi động lại hoặc auto-scale, toàn bộ cache in-memory bị xóa sạch.

### Giải pháp với `SharedRedisCache`
- `SharedRedisCache` gom toàn bộ bộ nhớ đệm về cụm Redis tập trung.
- Sử dụng cấu trúc Redis Hash (`HSET`/`HGETALL`) với cơ chế `EXPIRE` tự động giải phóng bộ nhớ.
- Tra cứu nhanh O(1) qua key băm MD5 cho exact match và quét `SCAN` + N-gram Cosine Similarity cho semantic match.

### Bằng chứng trạng thái dùng chung (Shared State Verification)
Kết quả chạy kiểm thử test case `test_shared_state_across_instances`:
```python
c1 = SharedRedisCache(redis_url="redis://localhost:6379/0", ttl_seconds=60, similarity_threshold=0.5, prefix="rl:test:shared:")
c2 = SharedRedisCache(redis_url="redis://localhost:6379/0", ttl_seconds=60, similarity_threshold=0.5, prefix="rl:test:shared:")
c1.set("shared query", "shared response")
cached, _ = c2.get("shared query")
assert cached == "shared response"  # PASS -> Hai instance độc lập cùng nhìn thấy dữ liệu
```

### Dữ liệu thực tế trong Redis (Redis CLI Inspection)
```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:9e413fd814eb"
2) "rl:cache:734852f3cf4a"
3) "rl:cache:3dab98c0e49e"
4) "rl:cache:da61fb49b4f6"
5) "rl:cache:fff10da1c72c"
...

$ docker compose exec redis redis-cli HGETALL "rl:cache:9e413fd814eb"
1) "response"
2) "[backup] reliable answer for: What should I do when API calls return 429?"
3) "query"
4) "What should I do when API calls return 429?"
```

---

## 7. Đánh giá Chi tiết các Kịch bản Chaos (Chaos Scenarios Breakdown)

| Kịch bản (Scenario) | Hành vi kỳ vọng (Expected) | Hành vi thực tế quan sát được (Observed) | Trạng thái (Pass/Fail) |
|---|---|---|:---:|
| `all_healthy` | Cả 2 provider khỏe mạnh; 100% request xử lý qua Primary hoặc Cache; Circuit Breaker giữ CLOSED. | Toàn bộ 100 request thành công, 0 lỗi, 0 lần ngắt mạch; mạch luôn ở trạng thái CLOSED. | **PASS** |
| `primary_timeout_100` | Primary hỏng 100%; Circuit Breaker Primary ngắt sang OPEN; 100% lưu lượng tự động fallback sang Backup. | Mạch Primary trip sang OPEN sau 3 lần fail; toàn bộ request sau đó fail-fast và chuyển sang Backup thành công 100%. | **PASS** |
| `primary_flaky_50` | Primary chập chờn (fail 50%); Circuit Breaker dao động giữa CLOSED <-> OPEN <-> HALF_OPEN <-> CLOSED. | Ghi nhận các đợt ngắt mạch tạm thời và phục hồi thành công qua probe request ở HALF_OPEN, kết hợp tải giữa Primary và Backup. | **PASS** |
| `cascade_failure_all_down` | Cả Primary và Backup đều sập 100%; cả 2 Circuit Breaker đều OPEN; trả về Degraded Static Fallback an toàn. | Hệ thống trả về 100% Static Fallback message (`The service is temporarily degraded...`), không crash ứng dụng. | **PASS** |

---

## 8. Phân tích Điểm yếu & Phương án Khắc phục cho Production (Failure Analysis)

### Điểm yếu còn tồn tại:
1. **Quét Semantic Scan O(N) trong Redis:**
   - Hiện tại `SharedRedisCache.get()` sử dụng `SCAN` và tính toán tương đồng cục bộ trên toàn bộ key có prefix. Khi số lượng key lên tới hàng triệu, việc scan sẽ làm tăng tải CPU và độ trễ mạng.
2. **Trạng thái Circuit Breaker lưu cục bộ (Local State per Node):**
   - Bộ đếm failure count và trạng thái circuit breaker hiện nằm trong memory của từng process gateway. Khi chạy 50 replicas, một node bị lỗi có thể không đồng bộ trạng thái mở mạch với các node khác.
3. **Chưa có cơ chế Rate Limiting / Token Bucket:**
   - Khi có spike lưu lượng đột biến từ một user độc hại, gateway có thể gửi quá nhiều request làm sập Backup provider.

### Giải pháp kỹ thuật đề xuất:
1. **Tích hợp Redis Vector Search (RediSearch / HNSW Index):**
   - Lưu trữ embedding vector trực tiếp trong Redis và sử dụng thuật toán KNN (K-Nearest Neighbors) với độ phức tạp O(log N) thay cho linear scan.
2. **Distributed Circuit Breaker qua Redis Lua Scripting:**
   - Đồng bộ hóa trạng thái Circuit Breaker qua các lệnh nguyên tử (Atomic `INCR`, `EXPIRE`) hoặc Redis Pub/Sub để khi một replica mở mạch, toàn bộ cluster đều fail-fast ngay lập tức.
3. **Thêm Middleware Leaky/Token Bucket Rate Limiter:**
   - Bổ sung tầng giới hạn tốc độ theo User ID / API Key trước khi đưa request vào pipeline định tuyến.

---

## 9. Định hướng Nâng cấp Tiếp theo (Next Steps)

1. **Triển khai Cost-Aware Dynamic Routing:**
   - Tự động theo dõi ngân sách chi phí theo thời gian thực: khi chi phí đạt 80% hạn mức ngày, tự động chuyển hướng ưu tiên sang các model giá rẻ; khi đạt 100%, chuyển sang chế độ Cache-Only hoặc Static Fallback.
2. **Xây dựng Dashboard Giám sát Prometheus & Grafana:**
   - Xuất các metrics chuẩn OpenTelemetry/Prometheus (Counter, Histogram cho Latency P50/P95/P99, Gauge cho Circuit State) để theo dõi trực quan và cảnh báo qua Slack/PagerDuty.
3. **Kiểm thử Fuzzing & Property-Based Testing với `hypothesis`:**
   - Tự động sinh hàng triệu chuỗi chuyển trạng thái ngẫu nhiên để chứng minh về mặt toán học rằng Circuit Breaker không bao giờ rơi vào trạng thái bế tắc (deadlock) hoặc retry storm.

---

## 10. Kết luận Nghiệm thu (Verification & Sign-off)

- [x] **100% Tests Passed:** Đạt 42/42 tests (`pytest -v`), toàn bộ 7 xfail tests đã tự động chuyển thành Unexpected PASS (XPASS).
- [x] **Strict Type Safety:** Đạt 100% kiểm tra kiểu nghiêm ngặt với `mypy src --strict` (0 lỗi).
- [x] **Clean Code:** Đạt 100% chuẩn linting và formatting với `ruff check` (0 lỗi).
- [x] **Redis Shared Cache:** Triển khai hoàn chỉnh trên Docker Redis, kiểm chứng đồng bộ đa instance.
- [x] **Báo cáo và Metrics:** Xuất đầy đủ `reports/metrics.json`, `reports/metrics.csv`, `reports/final_report.md`, `reports/report.md` và `report.md`.
