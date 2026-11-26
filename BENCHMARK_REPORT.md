# Benchmark & Observability Report — Day 14

**Họ tên:** Đậu Hoàng Minh — 2A202604011
**Ngày chạy:** 26/11/2026
**Môi trường:** MacBook Pro M1 (8 core: 4P + 4E, 16 GB RAM), macOS, Python 3.12.6; Docker Desktop 29.9 (VM 8 CPU / 7.7 GB). Stack production: mỗi agent giới hạn 0.5 CPU / 256 MB.

Nguồn số liệu: `reports/*.json` (load test, eval) và `benchmark-results/*.csv` (Locust).

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | LLM mock mất ~120–180 ms/call; 500 ms cho phép retrieval + hàng đợi nhẹ mà người dùng chat vẫn thấy "tức thì". Trùng ngưỡng mặc định của `load_test.py`. Alert production (`AgentHighLatencyP95`) đặt lỏng hơn ở 1 s để chỉ báo động khi lệch hẳn |
| Error rate `/ask` | < 2 % | Mock có ~1 % lỗi upstream ngẫu nhiên (`MOCK_ERROR_RATE=0.01`); 2 % chừa biên cho dao động mà vẫn bắt được sự cố thật. Alert critical đặt ở 5 % |
| Accuracy trên golden dataset | ≥ 85 % (gate CI); mục tiêu release 100 % | v1 chỉ đạt 4,3 %, v2 sau khi sửa KB đạt 100 %. Gate 85 % trong CI cho phép 3/23 câu sai trong lúc phát triển; bản release chạy `--min-accuracy 1.0` và cấm regression |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

300 request mỗi mức, `MOCK_SLOW_RATE=0.03` (mặc định). Files: `reports/load_v2_c{1,10,50,100}.json`.

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.44 | 192.1 | 219.1 | 1705.1 | 1.00 % | ✅ PASS |
| 10 | 37.49 | 189.9 | 220.1 | 1706.9 | 1.33 % | ✅ PASS |
| 50 | 109.44 | 192.8 | 270.6 | 1702.3 | 1.67 % | ✅ PASS |
| 100 | 61.49 | 357.6 | 3502.4 | 3782.8 | 0.33 % | ❌ FAIL (p95) |

**Nhận xét:** Từ 1 → 10 → 50, RPS tăng gần tuyến tính (4,4 → 37,5 → 109,4, tức ~8,4× rồi ~2,9×) trong khi p50 gần như đứng yên ~190 ms. Lý do: mỗi request chủ yếu là *chờ* (`asyncio.sleep` mô phỏng LLM), event loop xử lý song song được nhiều request đang chờ, nên thêm concurrency là thêm throughput. Ở 100, RPS **giảm** còn 61,5 và p50 tăng gần gấp đôi, p95 vọt lên 3,5 s: agent chạy 1 process, 1 event loop; khi có 100 request cùng lúc, thời gian CPU cho phần đồng bộ (parse JSON, Pydantic, ghi log, cập nhật metrics) bắt đầu xếp hàng, cộng với load generator chạy trên cùng máy tranh CPU. Đây là điểm bão hoà: muốn tăng tiếp phải chạy nhiều worker/instance chứ không phải đẩy thêm concurrency. Lưu ý thêm: mỗi lần chạy chỉ 300 request, ở c=50 và c=100 thời lượng đo chỉ 2,7 s và 4,9 s nên số liệu ở hai mức này dao động khá nhiều giữa các lần chạy.

### 2.2 Tail latency

300 request, concurrency 20, restart v2 với từng giá trị. Files: `reports/load_v2_slow_{0,0.03,0.1}.json`.

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 189.5 | 217.3 | 226.5 |
| 0.03 (mặc định) | 209.6 | 217.9 | 1656.0 |
| 0.1 | 415.7 | 1701.9 | 1712.2 |

**Nhận xét:** Chỉ số nhạy nhất là **p99** (và p95 khi tỉ lệ chậm vượt 5 %). Với 3 % request chậm thêm 1,5 s: avg chỉ tăng 10,6 % (189,5 → 209,6 ms), p95 gần như không đổi, nhưng p99 tăng **7,3 lần** (226,5 → 1656 ms), vì 3 % request chậm nằm gọn trong 5 % đuôi nên chỉ p99 "nhìn thấy". Khi tăng lên 10 %, nhóm chậm vượt 5 % đuôi nên p95 cũng nhảy lên 1,7 s. Average trộn 97 request nhanh với 3 request rất chậm thành một con số "trông ổn", che mất việc cứ 33 người dùng có 1 người phải chờ gần 2 giây. Vì vậy SLO nên đặt theo **p95/p99**: chúng mô tả trải nghiệm của nhóm người dùng tệ nhất, và với một user gửi nhiều request mỗi phiên thì xác suất họ gặp ít nhất 1 request đuôi là rất cao.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

Golden dataset 23 câu. Files: `reports/eval_v1.json`, `reports/eval_v2.json`, `reports/eval_v2_fixed.json`.

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3 % (1/23) | 91.3 % (21/23) | 100 % (23/23) |
| Avg tokens / câu | 58.3 | 108.8 | 114.1 |
| Tổng cost (USD) | 0.000420 | 0.000758 | 0.000795 |
| Cost / câu đúng (USD) | 0.00042015 | 0.00003607 | 0.00003455 |
| Quality gate | FAIL (gate 80 %) | PASS (gate 80 %, 0 regression so với v1) | PASS (gate 100 %, 0 regression so với v2) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` ("JWT khác gì API key trong xác thực?") | `token`, `hết hạn`, `chữ ký` | Knowledge base không có tài liệu nào chứa từ khoá của câu hỏi nên v2 rơi về câu trả lời chung chung. Thêm tài liệu kích hoạt bằng `["jwt"]`: API key cố định, không tự hết hạn; JWT là token có chữ ký số, chứa user và thời điểm hết hạn, server xác minh chữ ký mà không cần tra DB |
| `scaling-03` ("Kubernetes HPA scale dựa trên những chỉ số nào?") | `cpu`, `memory`, `custom metrics` | Thêm tài liệu kích hoạt bằng `["hpa", "kubernetes", "autoscal"]`: HPA scale số pod theo CPU, memory hoặc custom metrics (RPS, độ dài hàng đợi) |

Trước khi thêm, mình grep `golden_dataset.jsonl` để chắc chắn các từ khoá mới **chỉ** khớp đúng 2 câu này (không câu nào khác chứa `jwt`, `hpa`, `kubernetes`, `autoscal`), tránh việc tài liệu mới "giành" retrieval của câu đang đúng. Kết quả: 2 câu cải thiện, 0 regression. Đã chép thay đổi sang `production/utils/mock_llm.py` để stack Docker cũng đạt 100 %.

**Quality gate trong CI:** [.github/workflows/eval.yml](.github/workflows/eval.yml) chạy agent v2 rồi `python eval/run_eval.py --min-accuracy 0.85` cho mỗi pull request. `run_eval.py` trả **exit code 1** khi accuracy dưới ngưỡng hoặc có câu bị regression so với `--baseline`, làm step fail và job fail. Đặt job `quality-gate / eval` làm *required status check* trong branch protection thì PR không merge được cho tới khi eval pass.

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**

- v2 đắt hơn **mỗi câu**: tokens/câu +87 % (58,3 → 108,8) vì prompt kèm tài liệu retrieve được; tổng cost +80 %; p95 latency +38 ms (183,2 → 220,9) do thêm bước retrieval (~20 ms) và prompt dài hơn.
- Nhưng v2 rẻ hơn **mỗi câu đúng** khoảng **11,6 lần** (0,00042 → 0,0000361 USD), vì v1 gần như không trả lời đúng câu nào. "Cost / câu" khiến v1 trông rẻ; "cost / câu đúng" cho thấy tiền chi cho v1 gần như phí hoàn toàn. Đây là thước đo đúng để so 2 phiên bản.
- Sửa KB làm tăng thêm 5,3 token/câu (+5 %) nhưng đưa accuracy lên 100 % và cost / câu đúng còn giảm tiếp (0,00003607 → 0,00003455).

---

## 4. Observability stack (production)

Stack chạy bằng `docker compose -p k5-day14-observability up -d --build`. Grafana map sang cổng **3001** vì cổng 3000 trên máy đang có ứng dụng khác (dùng file override `ports: !override ["3001:3000"]`, không sửa `docker-compose.yml`). Unit test alert rules: `promtool test rules /p/alert_rules_test.yml` → `SUCCESS`.

> **Lỗi có sẵn trong Dockerfile của stack đã phải sửa:** container agent crash với `ModuleNotFoundError: No module named 'uvicorn'`. User `agent` được tạo với home `/app` (`useradd -d /app`), nên user-site của Python là `/app/.local`, trong khi package được copy vào `/home/agent/.local`. Thêm `ENV PYTHONUSERBASE=/home/agent/.local` thì chạy được.

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải: [screenshots/grafana-dashboard.png](screenshots/grafana-dashboard.png) (chụp lúc 14:56, cuối lượt Locust 100 user / 3 instance; thấy cả 2 lượt scale 1 và scale 3)
- [x] Một trace chậm trong Jaeger: [screenshots/jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png)
- [x] Một trace lỗi trong Jaeger (có event `exception`): [screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png)
- [x] Trang Alerts của Prometheus, **3 alert cùng firing**: [screenshots/prometheus-alert-firing.png](screenshots/prometheus-alert-firing.png)

**Trả lời Exercise 7.4** (số liệu query Prometheus lúc 14:58:40, Locust 30 user, 3 instance, trước khi bắt đầu thí nghiệm):

1. **p95 = 292,8 ms, avg = 228,0 ms** (p99 = 1798 ms). p95 chỉ cao hơn avg ~65 ms, còn p99 gấp gần 8 lần avg: phần lớn request đều đặn quanh 200 ms, nhưng có một nhóm nhỏ (~3 % request "slow" của mock LLM, +1,5 s) tạo đuôi dài. Khoảng cách giữa avg và p99 chính là tail latency mà average che mất.
2. **Có, Nginx chia khá đều:** req/s theo instance là 5,81 / 5,67 / 6,11 (lệch tối đa ~7 %). Lệch nhỏ đến từ round-robin theo kết nối và việc mỗi request có thời gian xử lý khác nhau.
3. **Input tokens/s (1350) gần gấp đôi output tokens/s (707)** vì prompt của v2 gồm system prompt + tài liệu retrieve được + câu hỏi, còn câu trả lời của mock chỉ là 1 đoạn tài liệu ngắn. Với LLM thật, output đắt hơn input ~4 lần mỗi token nên vẫn phải theo dõi cả hai.
4. `curl -i http://localhost:8080/metrics` trả **`HTTP/1.1 404 Not Found`**: Nginx chặn `/metrics`, không public ra ngoài. Metrics chứa thông tin nội bộ (route, version, chi phí, số lỗi) và có thể bị gọi liên tục để dò hệ thống; Prometheus scrape trực tiếp từng agent qua network nội bộ `obs_net` (DNS service discovery), không cần đi qua Nginx.

### 4.2 Thí nghiệm "phá hệ thống"

Cách làm: Locust 30 user chạy nền; lúc 14:58:44 áp dụng **A** (recreate 3 agent với `MOCK_ERROR_RATE=0.2`), lúc 14:58:59 áp dụng **D** (`docker pause k5-day14-observability-agent-1`). Script poll `GET /api/v1/alerts` mỗi 5 s và ghi lại lần đầu thấy mỗi trạng thái:

```text
14:57:42 locust 30 users started
14:58:44 [A] MOCK_ERROR_RATE=0.2 applied (agents recreated)
14:58:59 [D] docker pause k5-day14-observability-agent-1
14:58:59 LLMCostBurnRateHigh -> firing (activeAt=07:50:27Z)  | đã firing từ trước (xem ghi chú)
14:59:09 AgentInstanceDown   -> pending (activeAt=07:59:07Z) | +10s since D
14:59:19 AgentHighErrorRate  -> pending (activeAt=07:59:17Z) | +35s since A
14:59:40 AgentInstanceDown   -> firing                        | +41s since D
15:00:20 AgentHighErrorRate  -> firing                        | +96s since A
15:00:28 screenshots taken
15:00:51 restored (unpause, MOCK_ERROR_RATE default, locust stopped)
```

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` (> 5 % trong 1 phút) | **96 s** (pending sau 35 s, firing sau 96 s) | Stat "Error rate" chuyển đỏ; "Requests/s theo status code" xuất hiện đường 503; "LLM calls theo kết quả" tăng `error` |
| B | `MOCK_SLOW_RATE=0.3` | Không chạy | — | — |
| C | `LLM_MODEL=mock-llm-large` | Không đổi model; nhưng `LLMCostBurnRateHigh` (> $2/giờ) đã firing từ 14:50:27 chỉ vì **tải 100 user** ở benchmark scaling (Grafana: cost burn rate lên ~$6/giờ) | — | "Cost burn rate (USD/giờ)" vượt vạch đỏ $2 |
| D | `docker pause` 1 instance | `AgentInstanceDown` (`up == 0` trong 30 s) | **41 s** (pending sau 10 s, firing sau 41 s) | Stat "Instances up" giảm 3 → 2; "Requests/s theo instance" mất 1 đường |
| E | `AGENT_VERSION=v1` | Không chạy (theo đề: không alert nào firing) | — | — |

**Vì sao thời gian tới firing khác nhau:** thời gian ≈ thời gian để điều kiện thành đúng + `for` + chu kỳ đánh giá/scrape. D chỉ cần 1–2 lần scrape thất bại rồi `for: 30s` → 41 s. A phải chờ agent được recreate, `rate(...[1m])` tích luỹ đủ lỗi để vượt 5 % (35 s), rồi thêm `for: 1m` → 96 s.

**Ghi chú về D:** dùng `pause` chứ không `stop`, vì container bị stop biến mất khỏi Docker DNS nên Prometheus coi như target không tồn tại (không phải "down") và không có alert. Ngoài ra, request đang nằm trong instance bị pause bị treo: trong Jaeger có 1 trace `POST /ask` dài **109,5 s** (`ff32d062a7e3e768eb80d5c150111ca8`), chính là request bị kẹt suốt thời gian pause rồi mới xong khi unpause. Phía client, Nginx hết `proxy_read_timeout` sẽ trả 504 cho các request này, nhưng metric của agent không đếm chúng (request chưa được agent xử lý xong), nên panel Error rate không phản ánh. Lượt Locust trong thí nghiệm này mình không lưu CSV nên không có số 504 cụ thể.

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

Đổi sang v1 là lỗi **chất lượng**: request vẫn 200, latency và error rate còn đẹp hơn (v1 không retrieval nên nhanh và ít token hơn), nên mọi alert hạ tầng đều xanh. Cách phát hiện:
- **Quality gate trước deploy:** CI chạy `run_eval.py` với `--baseline`; v1 rơi từ 100 % xuống 4,3 % và có 22 câu regression → exit code 1, chặn merge/deploy.
- **Eval định kỳ trên bản đang chạy** (synthetic eval mỗi giờ) và export accuracy thành metric, alert khi giảm.
- **Metric đại diện cho chất lượng:** retrieval hit ratio (panel Grafana tụt về 0 khi chuyển v1), số token input/request giảm đột ngột, tỉ lệ câu trả lời fallback chung chung; alert khi lệch khỏi baseline.
- **Tín hiệu từ người dùng:** 👍/👎, tỉ lệ hỏi lại, kết hợp LLM-as-judge chấm mẫu trên traffic thật.
- **Theo dõi version:** bảng "Phiên bản đang chạy" và annotation deploy trên dashboard để gắn thay đổi chất lượng với lần deploy.

### 4.3 Debug bằng trace

Trace chậm nhất trong lúc chạy (không tính request bị kẹt vì `docker pause`), tìm trong log JSON theo `ms` lớn nhất của `/ask` 200, rồi tra trong Jaeger:

- `trace_id`: `d45e95d33b0566313ff80b623013c0e5`
- Tổng thời gian: **1727,0 ms** (span `POST /ask`)
- Span chiếm nhiều thời gian nhất: **`chat mock-llm-small` (1694,1 ms, 98,1 %)**; `agent.retrieve_context` chỉ 27,0 ms (1,6 %). Chậm nằm ở LLM upstream (request rơi vào nhóm "slow" +1,5 s của mock), không phải code của agent. Attributes: `gen_ai.request.model=mock-llm-small`, `gen_ai.usage.input_tokens=74`, `gen_ai.usage.output_tokens=37`.
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts": "2026-11-26T07:59:28.305703+00:00", "level": "INFO", "service": "observable-agent", "instance": "35d65bdd5aee", "event": "http_request", "trace_id": "d45e95d33b0566313ff80b623013c0e5", "span_id": "e9c4d3ca0e6f29dd", "request_id": "1914f25ae860", "method": "POST", "route": "/ask", "status": 200, "ms": 1725.7}
```

Trace lỗi ([screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png)): `trace_id` **`a296cef849f5259426a7aed9edd82343`**, `POST /ask` → 503, 97,3 ms. Span `chat mock-llm-small` (71,2 ms) có status error, `error.type=upstream_unavailable` và event `exception` với message `Upstream LLM error: 503 Service Unavailable (mock)`; lỗi lan lên `agent.run`. Log tương ứng:

```json
{"ts": "2026-11-26T08:00:48.511060+00:00", "level": "INFO", "service": "observable-agent", "instance": "35d65bdd5aee", "event": "llm_error", "trace_id": "a296cef849f5259426a7aed9edd82343", "span_id": "7a18aa18c3e83824", "request_id": "4d52683ac09e", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
{"ts": "2026-11-26T08:00:48.511557+00:00", "level": "INFO", "service": "observable-agent", "instance": "35d65bdd5aee", "event": "http_request", "trace_id": "a296cef849f5259426a7aed9edd82343", "span_id": "7a18aa18c3e83824", "request_id": "4d52683ac09e", "method": "POST", "route": "/ask", "status": 503, "ms": 96.4}
```

Lỗi này đến sớm (97 ms so với ~200 ms của request thành công) vì mock trả lỗi sau một nửa latency, giống provider thật thường trả 429/503 nhanh hơn một response đầy đủ.

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

Chạy qua Nginx, mỗi agent 0.5 CPU / 256 MB. Files: `benchmark-results/scale_1_stats.csv`, `benchmark-results/scale_3_stats.csv` (dòng `POST /ask`).

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 60.12 | 200 | 250 | 1700 | 70 / 7175 (0.98 %) |
| 3 | 59.61 | 200 | 230 | 1700 | 64 / 7111 (0.90 %) |

**Giải thích kết quả:** Scale lên 3 instance **gần như không đổi** RPS (60,1 → 59,6) và p50 (200 ms), p95 chỉ giảm nhẹ 250 → 230 ms, p99 giữ nguyên 1,7 s. Hai nút thắt:

1. **Load generator là hệ đóng (closed workload):** Locust có 100 user, mỗi user gửi request rồi nghỉ `between(0.5, 2)` giây (trung bình 1,25 s). Thông lượng tối đa ≈ 100 / (1,25 s + ~0,24 s latency) ≈ **67 req/s** tổng (aggregated đo được 65,9 req/s, gồm cả `/health`). Agent 1 instance đã phục vụ kịp mức tải này, nên thêm instance không tạo thêm request để xử lý.
2. **Latency bị chi phối bởi LLM:** panel "LLM latency p95" ~195–199 ms, chiếm gần hết thời gian request; p99 = 1,7 s đến từ 3 % request "slow" của mock LLM (thêm 1,5 s). Scale ngang không làm LLM nhanh hơn, chỉ tăng số request xử lý song song.

Phần cải thiện nhỏ ở p95 (−20 ms) đến từ việc mỗi instance chỉ được 0,5 CPU: ở 1 instance, lúc tải đỉnh phần xử lý CPU (JSON, metrics, tracing) bắt đầu xếp hàng; chia cho 3 instance thì hàng đợi đó biến mất. Muốn thấy lợi ích rõ của scale cần tăng tải (nhiều user hơn, giảm `wait_time`) tới khi 1 instance bão hoà, hoặc đo bằng mô hình *open workload* (giữ RPS cố định).

---

## 6. Kết luận

- 3 điều học được về benchmark & observability:
  1. **Average nói dối, percentile thì không:** 3 % request chậm làm p99 tăng 7,3 lần nhưng avg chỉ tăng 11 %. SLO và alert phải đặt theo p95/p99.
  2. **Benchmark hiệu năng không thay được benchmark chất lượng:** v1 nhanh hơn và rẻ hơn mỗi request nhưng chỉ đúng 4,3 %. Chỉ có eval trên golden dataset (và chỉ số cost / câu đúng) mới cho thấy điều đó; các thí nghiệm hạ tầng (error rate, latency, up) đều xanh khi chất lượng tụt.
  3. **Phải hiểu mô hình tải trước khi kết luận về scaling:** với closed workload, thông lượng bị chặn bởi số user và think time, scale 1 → 3 instance gần như không đổi gì. Kết quả benchmark phụ thuộc vào cách sinh tải nhiều như vào hệ thống được đo.
- Nếu đưa agent này lên production thật, sẽ thêm:
  - **Eval online / chất lượng:** chạy golden dataset định kỳ (vd mỗi giờ) với bản đang chạy, export `agent_eval_accuracy` thành metric và alert khi giảm; theo dõi retrieval hit ratio, tỉ lệ câu trả lời "chung chung" (fallback), feedback 👍/👎 của user.
  - **Giám sát ngoài agent:** synthetic check từ bên ngoài và metric của Nginx (tỉ lệ 502/504, upstream latency), vì lỗi ở tầng LB không hiện trong metric của agent.
  - **Error budget burn rate alert** (multi-window, vd 1h và 6h) thay cho ngưỡng cố định, alert theo **cost / ngày so với budget** và token/request bất thường (prompt injection, prompt quá dài), cùng metric `agent_question_length_chars` đã thêm ở Exercise 7.1.
