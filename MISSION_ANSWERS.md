# Day 14 — Mission Answers

> Môi trường chạy: MacBook Pro M1 (8 core, 16 GB RAM), macOS, Python 3.12, Docker Desktop 29.9 (VM 8 CPU / 7.7 GB).
> Mọi output dưới đây là output thật từ các lệnh đã chạy trên máy này ngày 26/11/2026.
>
> **Ghi chú về cổng:** trên máy có một service khác đang giữ `127.0.0.1:8000`, nên ở Part 4 và Part 5 (develop) mình chạy app mẫu bằng `PORT=8010`. Lệnh và kết quả giống hệt khi chạy ở cổng 8000.

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns trong `01-localhost-vs-production/develop/app.py`

| # | Anti-pattern | Dòng | Vì sao nguy hiểm |
|---|---|---|---|
| 1 | Hardcode secret: `OPENAI_API_KEY = "sk-hardcoded-..."` và `DATABASE_URL` có cả mật khẩu `password123` | 17–18 | Push lên GitHub là lộ key ngay; muốn đổi key phải sửa code và deploy lại |
| 2 | **Log ra secret**: `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` | 34 | Secret nằm trong log, ai đọc log (hoặc hệ thống log tập trung) cũng thấy |
| 3 | Dùng `print()` thay vì logging có cấu trúc, log cả nội dung câu hỏi và câu trả lời | 33, 38 | Không có level, timestamp, không parse được; lộ dữ liệu người dùng |
| 4 | Không có health check endpoint | 42 | Platform không biết khi nào app treo để restart (đã test: `GET /health` → 404) |
| 5 | `host="localhost"` | 51 | Chỉ nhận kết nối từ chính máy đó → chạy trong container thì bên ngoài không gọi được |
| 6 | Port cố định `port=8000` | 52 | Railway/Render cấp port qua biến `PORT`; port cứng → app không nhận traffic |
| 7 | `reload=True` và `DEBUG = True` | 21, 53 | Reloader theo dõi file, tốn tài nguyên, không dùng cho production |
| 8 | Config (`MAX_TOKENS`, `DEBUG`) viết cứng trong code | 21–22 | Đổi môi trường dev/staging/prod phải sửa code (vi phạm 12-Factor) |
| 9 | Không xử lý shutdown, không lifespan | — | Không đóng kết nối, không có chỗ dọn dẹp khi platform gửi SIGTERM |

### Exercise 1.2: Chạy bản develop

```text
$ curl -i -X POST "http://localhost:8000/ask?question=Hello"
HTTP/1.1 200 OK
content-type: application/json

{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}

$ curl -i http://localhost:8000/health
HTTP/1.1 404 Not Found

{"detail":"Not Found"}
```

Log của server, **secret bị in ra**:

```text
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
[DEBUG] Response: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.
```

Nó chạy, nhưng không production-ready: không có health check, lộ secret trong log, chỉ bind `localhost`.

### Exercise 1.3: So sánh develop và production

Chạy bản production (`ENVIRONMENT=development PORT=8000 python app.py`):

```text
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":3.7,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T07:07:00.997717+00:00"}
$ curl http://localhost:8000/ready
{"ready":true}
$ curl -X POST localhost:8000/ask -H 'Content-Type: application/json' -d '{"question":"Hello"}'
{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). ...","model":"gpt-4o-mini"}
```

| Feature | Develop | Production | Tại sao quan trọng? |
|---|---|---|---|
| Config | Hardcode trong code (`OPENAI_API_KEY`, `DEBUG`, `MAX_TOKENS`) | `config.py` đọc env var, `.env.example` liệt kê biến; `validate()` fail fast nếu thiếu `AGENT_API_KEY` ở production | Một image chạy được mọi môi trường; secret không vào git |
| Health check | Không có (404) | `/health` (liveness) và `/ready` (readiness, 503 khi chưa sẵn sàng) | Platform biết khi nào restart, LB biết khi nào route traffic |
| Logging | `print()`, in cả secret | Logging dạng JSON, không log secret, chỉ log độ dài câu hỏi | Parse được bởi log aggregator, không lộ dữ liệu |
| Shutdown | Đột ngột | `lifespan` đánh dấu `is_ready=False` rồi dọn dẹp; uvicorn chờ request đang chạy | Không cắt ngang request khi deploy/scale |
| Host / Port | `localhost:8000` cứng | `0.0.0.0` + `PORT` từ env | Chạy được trong container và trên cloud |
| Reload | `reload=True` luôn bật | Chỉ khi `DEBUG=true` | Không tốn tài nguyên ở production |
| CORS | Không cấu hình | `ALLOWED_ORIGINS` từ env | Giới hạn origin được gọi API |

**Quan sát thêm khi chạy thật bản production** (2 điểm chưa hoàn hảo của chính code mẫu):

1. Log JSON không hiện ra. Dòng đầu log là `WARNING:root:OPENAI_API_KEY not set — using mock LLM` (định dạng mặc định). Lý do: `config.py` gọi `logging.warning(...)` lúc import, làm Python tự gọi `basicConfig()` với format mặc định và level WARNING trước khi `app.py` gọi `basicConfig(format=JSON, level=INFO)`. Lần gọi thứ hai không có tác dụng, nên toàn bộ log INFO dạng JSON bị nuốt.
2. Hàm `handle_sigterm` tự đăng ký không bao giờ chạy (log không có dòng "Received SIGTERM"), vì `uvicorn.run()` đăng ký handler SIGTERM của nó sau khi import app và đè lên. Shutdown vẫn êm (`Waiting for application shutdown… Application shutdown complete`) là nhờ uvicorn. Ở Part 6 mình dùng `lifespan` và không tự đăng ký signal handler để tránh chuyện này.

---

## Part 2: Docker

### Exercise 2.1: Đọc `02-docker/develop/Dockerfile`

1. **Base image:** `python:3.11`, bản Python đầy đủ dựa trên Debian (có sẵn gcc, header, nhiều thư viện), khoảng 1 GB.
2. **Working directory:** `/app` (`WORKDIR /app`).
3. **Vì sao `COPY requirements.txt` trước:** Docker cache theo từng layer. Copy riêng `requirements.txt` rồi `pip install` thì layer cài thư viện chỉ build lại khi `requirements.txt` đổi. Nếu `COPY . .` trước, sửa 1 dòng code cũng làm mất cache và phải cài lại toàn bộ dependencies.
4. **CMD và ENTRYPOINT:** `CMD` là lệnh/tham số mặc định, bị thay thế khi `docker run image <lệnh khác>`. `ENTRYPOINT` là executable cố định của container; tham số của `docker run` (hoặc `CMD`) được nối thêm vào sau, muốn đổi phải dùng `--entrypoint`. Hay dùng `ENTRYPOINT ["uvicorn"]` + `CMD ["app:app", "--port", "8000"]` để vừa cố định chương trình vừa đổi được tham số.

### Exercise 2.2: Build và chạy

```text
$ docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
$ docker run -d -p 8000:8000 my-agent:develop
$ curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker?"
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":2.9,"container":true}
```

### Exercise 2.3: Multi-stage build

- **Stage 1 (`builder`, `python:3.11-slim`):** cài `gcc`, `libpq-dev`, rồi `pip install --user -r requirements.txt` vào `/root/.local`. Stage này chỉ dùng để build, không được deploy.
- **Stage 2 (`runtime`, `python:3.11-slim`):** tạo user non-root `appuser`, chỉ `COPY --from=builder /root/.local`, copy `main.py` và `utils/`, chạy bằng `USER appuser`, có `HEALTHCHECK`, CMD `uvicorn ... --workers 2`.
- **Vì sao nhỏ hơn:** image cuối không chứa compiler, header, apt cache hay pip cache của stage builder; base `slim` thay cho bản full (vốn kèm cả bộ build tools của Debian).

Kích thước đo bằng `docker images`:

```text
REPOSITORY:TAG      SIZE
my-agent:advanced   265MB
my-agent:develop    1.67GB

$ docker image inspect ... --format '{{index .RepoTags 0}} {{.Size}}'
my-agent:develop 1672165583
my-agent:advanced 265493197
```

- Develop: **1672 MB** (1 672 165 583 bytes)
- Production: **265 MB** (265 493 197 bytes)
- Chênh lệch: production nhỏ hơn **84,1%** ((1 672 165 583 − 265 493 197) / 1 672 165 583)

### Exercise 2.4: Docker Compose stack

```text
                 :80 / :443
   client ───────────▶ ┌────────┐
                       │ nginx  │  reverse proxy + LB (upstream agent_backend → agent:8000)
                       └───┬────┘
          network "internal" (bridge, không publish port nào khác)
                           ▼
                       ┌────────┐  REDIS_URL=redis://redis:6379/0
                       │ agent  │──────────────────────────┐
                       └───┬────┘                          ▼
                           │ QDRANT_URL=http://qdrant:6333 ┌────────┐
                           ▼                               │ redis  │ (volume redis_data)
                       ┌────────┐                          └────────┘
                       │ qdrant │ (volume qdrant_data)
                       └────────┘
```

- 4 service được start: `agent`, `redis`, `qdrant`, `nginx`.
- Chúng giao tiếp qua network nội bộ `internal`, gọi nhau bằng **tên service** (Docker DNS): `redis:6379`, `qdrant:6333`, `agent:8000`.
- Chỉ `nginx` mở port ra ngoài (80, 443); agent không expose trực tiếp.
- `agent` có `depends_on` với `condition: service_healthy` cho redis và qdrant, nên chỉ start khi 2 service này đã healthy.

```text
$ docker compose -p k5-day14-docker up -d --build
SERVICE   STATUS                            PORTS
agent     Up 2 seconds (health: starting)   8000/tcp
nginx     Up 2 seconds                      0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
qdrant    Up 13 seconds (healthy)           6333-6334/tcp
redis     Up 12 seconds (healthy)           6379/tcp

$ curl http://localhost/health
{"status":"ok","uptime_seconds":1.5,"version":"2.0.0","timestamp":"2026-11-26T07:27:55.169796"}
$ curl -X POST http://localhost/ask -H "Content-Type: application/json" -d '{"question": "Explain microservices"}'
{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

---

## Part 3: Cloud Deployment

> Lớp K5 deploy chính agent Part 6 (`06-lab-complete/`), xem [DEPLOYMENT.md](DEPLOYMENT.md).

### Nền tảng chính: Render (Exercise 3.2)

- **URL: https://day14-agent-43am.onrender.com** (Render Free, Singapore; agent Docker + Key Value Redis)
- Tự kiểm CP6: `/health` **200**, `/ready` **200** (`{"status":"ready","redis":"ok"}`), `/ask` không key **401**, có key **200**; rate limit trên cloud trả 429 sau 10 request/phút; `check_production_ready.py --public-url` → ✅.
- Screenshot: [screenshots/dashboard.png](screenshots/dashboard.png), [screenshots/running.png](screenshots/running.png), [screenshots/test.png](screenshots/test.png). Chi tiết: [DEPLOYMENT.md](DEPLOYMENT.md).
- Railway (Exercise 3.1): đã viết `railway.toml` và đăng nhập CLI, nhưng `railway init` báo *"Your trial has expired. Please select a plan to continue using Railway"* (gói Free sau trial chỉ cho 1 project, tài khoản đã có 1 project khác), nên chọn Render làm nền tảng chính. Mục "nền tảng còn lại" làm bằng phần so sánh bên dưới.
- Cách set biến môi trường trên cloud: `AGENT_API_KEY` sinh ngẫu nhiên và đặt bằng `render services create --env-var ...` (không có trong repo); `REDIS_URL` trỏ tới địa chỉ nội bộ của Key Value `redis://red-…:6379`; `PORT` do Render cấp (10000). Xem log bằng `render logs -r <service-id>`.

### So sánh `railway.toml` và `render.yaml`

So sánh 2 file mình viết trong `06-lab-complete/`:

| Khía cạnh | `railway.toml` | `render.yaml` | Vì sao khác |
|---|---|---|---|
| Phạm vi | Chỉ cấu hình **build/deploy của 1 service** | **Blueprint toàn bộ hạ tầng**: web service + Key Value (Redis) trong cùng file | Railway tạo Redis bằng dashboard/CLI (`railway add`), Render cho khai báo nhiều service dạng Infrastructure-as-Code |
| Định dạng | TOML, các bảng `[build]`, `[deploy]` | YAML, danh sách `services:` | Quy ước của từng nền tảng |
| Build | `builder = "DOCKERFILE"`, `dockerfilePath` | `runtime: docker`, `dockerfilePath: ./06-lab-complete/Dockerfile`, `dockerContext: ./06-lab-complete`, `buildFilter` chỉ build lại khi `06-lab-complete/**` đổi | Railway upload thư mục hiện tại bằng `railway up`; Render build từ GitHub repo (đường dẫn tính từ gốc repo) nên phải chỉ rõ thư mục con và nên lọc để sửa file khác không trigger deploy |
| Health check | `healthcheckPath = "/health"`, `healthcheckTimeout = 60` | `healthCheckPath: /health` | Cùng ý tưởng, khác tên khoá |
| Biến môi trường / secret | Không có trong file; đặt bằng `railway variables --set AGENT_API_KEY=...` | Khai báo trong file: giá trị thường ghi thẳng, secret `AGENT_API_KEY` dùng `generateValue: true` (Render tự sinh, không có trong repo), `REDIS_URL` lấy bằng `fromService ... property: connectionString` | Render cần biết trước danh sách biến khi tạo Blueprint, và tự nối URL Redis giữa các service |
| Region / plan | Chọn trên dashboard | `plan: free`, `region: singapore` ghi trong file | Blueprint là IaC nên ghi cả cấu hình hạ tầng |
| Restart | `restartPolicyType = "ON_FAILURE"`, tối đa 3 lần | Render tự quản lý | Railway cho chỉnh chính sách restart |
| Port | Railway cấp `PORT`, Dockerfile đọc `${PORT:-8000}` | Render cũng cấp `PORT`, dùng chung Dockerfile | Cùng một image chạy được trên cả hai nhờ đọc port từ env |

### Exercise 3.3: Pipeline CI/CD Cloud Run (`cloudbuild.yaml` + `service.yaml`)

1. **test:** Cloud Build chạy container `python:3.11-slim`, cài dependencies và chạy `pytest tests/`; test fail thì pipeline dừng.
2. **build:** `docker build` image, gắn 2 tag `gcr.io/$PROJECT_ID/ai-agent:$COMMIT_SHA` và `:latest`, dùng `--cache-from :latest` để tận dụng layer cache (chạy sau khi `test` pass, `waitFor: [test]`).
3. **push:** đẩy cả 2 tag lên Container Registry.
4. **deploy:** `gcloud run deploy ai-agent` với image theo đúng `COMMIT_SHA` (truy vết được bản nào đang chạy, rollback dễ), region `asia-southeast1`, min 1 / max 10 instance, 512Mi / 1 CPU, timeout 60s; secret `OPENAI_API_KEY` lấy từ **Secret Manager** (`--set-secrets`), không nằm trong image.
5. `service.yaml` mô tả cùng service theo kiểu khai báo (Knative): `containerConcurrency: 80`, autoscale theo target 80 request đồng thời, `minScale: 1` tránh cold start, `livenessProbe` gọi `/health`, `startupProbe` gọi `/ready`, `AGENT_API_KEY` và `OPENAI_API_KEY` lấy từ `secretKeyRef`.

---

## Part 4: API Security

### Exercise 4.1: API key (`04-api-gateway/develop`)

- **Check ở đâu:** hàm `verify_api_key` (dòng 39–54) đọc header `X-API-Key` qua `APIKeyHeader(name="X-API-Key", auto_error=False)`, được inject vào `/ask` bằng `Depends(verify_api_key)` (dòng 70).
- **Sai key:** thiếu key → **401**; có key nhưng sai → **403** (Part 6 yêu cầu cả hai trường hợp đều 401, mình đã làm như vậy trong `06-lab-complete/app/auth.py`).
- **Rotate key:** key đọc từ env `AGENT_API_KEY`, nên đổi biến môi trường trên platform rồi restart/redeploy là xong, không sửa code. Để không làm gián đoạn client, nên hỗ trợ **nhiều key cùng lúc** trong thời gian chuyển đổi (vd `AGENT_API_KEYS=old,new`), báo client đổi sang key mới rồi mới xoá key cũ; lưu key trong secret manager và so sánh bằng `secrets.compare_digest`.

```text
$ curl -i -X POST http://localhost:8010/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'   # không có key
HTTP/1.1 401 Unauthorized
content-type: application/json

{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

$ curl -i -X POST http://localhost:8010/ask -H "X-API-Key: wrong-key" ...
HTTP/1.1 403 Forbidden
content-type: application/json

{"detail":"Invalid API key."}

$ curl -i -X POST http://localhost:8010/ask -H "X-API-Key: secret-key-123" ...
HTTP/1.1 200 OK
content-type: application/json

{"question":"Hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

### Exercise 4.2: JWT (`04-api-gateway/production`)

**Luồng JWT:**
1. Client `POST /auth/token` với username/password → `authenticate_user` kiểm tra trong `DEMO_USERS`.
2. `create_token` tạo payload `{sub, role, iat, exp = now + 60 phút}`, ký HS256 bằng `JWT_SECRET`.
3. Client gửi `Authorization: Bearer <token>` ở mỗi request.
4. `verify_token` giải mã, kiểm tra chữ ký và hạn (`exp`); sai/hết hạn → 401. Server **không cần tra database** mỗi request vì thông tin user và role nằm trong token đã ký.

Lưu ý: `JWT_SECRET` có giá trị mặc định cứng `"super-secret-change-in-production-please"`; nếu quên đặt env ở production thì ai cũng ký được token giả. Nên fail fast khi thiếu biến này.

```text
$ curl -i -X POST http://localhost:8010/auth/token -d '{"username":"student","password":"demo123"}'
HTTP/1.1 200 OK
content-type: application/json

{"access_token":"eyJ***REDACTED***","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer eyJ***..."}

$ curl -i -X POST http://localhost:8010/ask ...   # không có token
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer

{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

$ curl -i -X POST http://localhost:8010/ask -H "Authorization: Bearer $TOKEN" -d '{"question":"Explain JWT"}'
HTTP/1.1 200 OK

{"question":"Explain JWT","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":9,"budget_remaining_usd":2.1e-05}}
```

### Exercise 4.3: Rate limiting

- **Algorithm:** sliding window log. Mỗi user có một `deque` timestamp; mỗi request xoá các timestamp cũ hơn 60 giây rồi đếm.
- **Limit:** user thường (`student`, role `user`) **10 request/phút**; admin (`teacher`) 100 request/phút.
- **"Bypass" cho admin:** không bỏ qua hẳn, mà chọn limiter theo role trong token (`rate_limiter_admin if role == "admin" else rate_limiter_user`). Admin dùng limiter 100 req/phút.
- **Hạn chế:** state nằm trong RAM của process, nên mỗi instance đếm riêng (3 instance = 30 req/phút thật) và mất khi restart. Ở Part 6 mình chuyển sang Redis sorted set.

Request "Explain JWT" ở 4.2 đã dùng 1 lượt, nên vòng lặp nhận 9 lần 200 rồi bị chặn:

```text
$ for i in {1..20}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8010/ask -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d "{\"question\":\"Test $i\"}"; done
req  1 -> 200
req  2 -> 200
req  3 -> 200
req  4 -> 200
req  5 -> 200
req  6 -> 200
req  7 -> 200
req  8 -> 200
req  9 -> 200
req 10 -> 429
req 11 -> 429
...
req 20 -> 429

$ curl -i ... (request thứ 21)
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795678248
retry-after: 59

{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}

$ curl http://localhost:8010/me/usage -H "Authorization: Bearer $TOKEN"
{"user_id":"student","date":"2026-11-26","requests":10,"input_tokens":40,"output_tokens":292,"cost_usd":0.000181,"budget_usd":1.0,"budget_remaining_usd":0.999819,"budget_used_pct":0.0}
```

### Exercise 4.4: Cost guard

**Bản mẫu** (`04-api-gateway/production/cost_guard.py`): lưu usage theo user theo ngày **trong RAM**; trước khi gọi LLM, hết budget toàn cục ($10/ngày) → 503, hết budget của user ($1/ngày) → **402**, dùng ≥ 80% thì log cảnh báo; sau khi gọi LLM, `record_usage` cộng token và cost.

**Cách mình làm** (theo yêu cầu "$10/tháng mỗi user, lưu Redis, reset đầu tháng"):

```python
import redis
from datetime import datetime, timezone

r = redis.from_url(os.environ["REDIS_URL"], decode_responses=True)
MONTHLY_BUDGET_USD = 10.0

def check_budget(user_id: str, estimated_cost: float) -> bool:
    month = datetime.now(timezone.utc).strftime("%Y-%m")
    key = f"budget:{user_id}:{month}"           # key theo tháng → tự "reset" đầu tháng
    new_total = r.incrbyfloat(key, estimated_cost)  # cộng nguyên tử: an toàn khi nhiều instance
    r.expire(key, 32 * 24 * 3600)               # tự xoá sau ~1 tháng
    if new_total > MONTHLY_BUDGET_USD:
        r.incrbyfloat(key, -estimated_cost)     # hoàn lại phần vừa giữ chỗ
        return False                            # caller trả HTTP 402
    return True
```

Khác với lời giải mẫu (`GET` rồi mới `INCRBYFLOAT`): mình cộng trước rồi kiểm tra, nên 2 request song song ở 2 instance không cùng lọt qua khi budget chỉ còn đủ cho 1 request (tránh race condition kiểu check-then-act).

Trong Part 6 (`06-lab-complete/app/cost_guard.py`), cost guard tính chi phí theo token (`input/1K × $0.00015 + output/1K × $0.0006`), cộng dồn bằng `INCRBYFLOAT` vào 2 key `cost:day:YYYY-MM-DD` (TTL 2 ngày) và `cost:month:YYYY-MM` (TTL 32 ngày), vượt `DAILY_BUDGET_USD` hoặc `MONTHLY_BUDGET_USD` → **402**.

**Rate limit và cost guard bổ sung nhau:** rate limit chặn *tốc độ* (spam, bot, DoS), cost guard chặn *tổng tiền*. Một user gửi đều 10 req/phút với prompt rất dài không vi phạm rate limit nhưng vẫn có thể đốt hết ngân sách; ngược lại cost guard một mình không chặn được burst làm quá tải server.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks (`05-scaling-reliability/develop`)

```text
$ curl http://localhost:8010/health
{"status":"ok","uptime_seconds":3.7,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T07:30:43.099916+00:00","checks":{"memory":{"status":"ok","used_percent":77.3}}}
$ curl http://localhost:8010/ready
{"ready":true,"in_flight_requests":1}
```

- **Liveness (`/health`):** "process còn sống không?" Fail → platform **restart** container. Không được phụ thuộc vào dependency bên ngoài, nếu không Redis chết một lúc sẽ làm mọi instance bị restart liên tục.
- **Readiness (`/ready`):** "có nên gửi traffic vào không?" Fail (503) → load balancer **ngừng route** tới instance đó nhưng không restart; dùng khi đang khởi động, đang shutdown hoặc mất kết nối Redis/DB.

### Exercise 5.2: Graceful shutdown

Gửi request rồi `kill -TERM` process ngay sau 50 ms:

```text
$ curl -X POST "http://localhost:8010/ask?question=Long%20task" & ; kill -TERM $PID
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."} [HTTP 200]
process còn sống? no (đã thoát)
exit code: 0
```

Log server: uvicorn nhận SIGTERM → `Shutting down` (ngừng nhận kết nối mới) → request đang chạy vẫn hoàn thành `POST /ask ... 200 OK` → `Waiting for application shutdown` → lifespan `Graceful shutdown initiated... Shutdown complete` → `Finished server process`. Request **hoàn thành**, process thoát với exit code 0.

### Exercise 5.3: Stateless design

`05-scaling-reliability/production/app.py` không giữ state trong RAM: session và conversation history lưu trong Redis bằng `SETEX session:{session_id}` (có TTL, giữ tối đa 20 message gần nhất). Vì vậy request kế tiếp của cùng một user có thể vào bất kỳ instance nào. Nếu lưu trong dict của process, mỗi instance có memory riêng: user bị chia sang instance khác sẽ "mất" lịch sử, và restart/scale down là mất sạch.

### Exercise 5.4: Load balancing (3 instance + Nginx)

```text
$ docker compose -p k5-day14-scaling up -d --build --scale agent=3
NAME                       STATUS                    PORTS
k5-day14-scaling-agent-1   Up 12 seconds (healthy)   8000/tcp
k5-day14-scaling-agent-2   Up 12 seconds (healthy)   8000/tcp
k5-day14-scaling-agent-3   Up 12 seconds (healthy)   8000/tcp
k5-day14-scaling-nginx-1   Up 12 seconds             0.0.0.0:8080->80/tcp
k5-day14-scaling-redis-1   Up 23 seconds (healthy)   6379/tcp

# 10 request POST /chat → instance phục vụ
req 1 -> instance-0a9342
req 2 -> instance-b3b9af
req 3 -> instance-0a9342
req 4 -> instance-425ea1
req 5 -> instance-b3b9af
req 6 -> instance-0a9342
req 7 -> instance-425ea1
req 8 -> instance-b3b9af
req 9 -> instance-0a9342
req 10 -> instance-425ea1
```

Nginx (`upstream agent_cluster { server agent:8000; }`) chia đều cho 3 instance theo round-robin qua Docker DNS. Thử tắt 1 instance:

```text
$ docker stop k5-day14-scaling-agent-1
req 1 -> instance-0a9342 HTTP 200
req 2 -> instance-425ea1 HTTP 200
req 3 -> instance-0a9342 HTTP 200
req 4 -> instance-425ea1 HTTP 200
req 5 -> instance-0a9342 HTTP 200
req 6 -> instance-425ea1 HTTP 200
```

Traffic tự chuyển sang 2 instance còn lại, không request nào lỗi (nhờ `proxy_next_upstream error timeout http_503`).

### Exercise 5.5: `test_stateless.py`

```text
Session ID: e2fdbd6f-a8b6-4b3c-9fb0-0b81b4046055

Request 1: [instance-b3b9af]  Q: What is Docker?
Request 2: [instance-0a9342]  Q: Why do we need containers?
Request 3: [instance-425ea1]  Q: What is Kubernetes?
Request 4: [instance-b3b9af]  Q: How does load balancing work?
Request 5: [instance-0a9342]  Q: What is Redis used for?
------------------------------------------------------------
Total requests: 5
Instances used: {'instance-0a9342', 'instance-b3b9af', 'instance-425ea1'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
  [user]: What is Docker?...
  [assistant]: Container là cách đóng gói app để chạy ở mọi nơi. Build once...
  ...
✅ Session history preserved across all instances via Redis!
```

**Giải thích:** 5 câu của cùng một session đi qua cả 3 instance khác nhau, nhưng history đọc lại đủ 10 message (5 user + 5 assistant) đúng thứ tự. Mỗi instance ghi vào cùng key `session:{id}` trong Redis, nên instance nào đọc cũng thấy đủ. Đây chính là điều kiện để scale ngang: thêm/bớt instance không làm mất dữ liệu người dùng.

---

## Part 6: Final Project — `python check_production_ready.py`

Chạy đầy đủ (có Docker) trong `06-lab-complete/`:

```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 43/43 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

Chi tiết đáng chú ý trong output: image **297 MB**; `docker stop` tắt êm với exit code 0 trong **0,5 s**; rate limit trả 429 sau đúng 5 request (`RATE_LIMIT_PER_MINUTE=5`) và **vẫn 429 sau khi restart agent** (state nằm trong Redis); Redis mất kết nối → `/ready` 503 còn `/health` 200.

Test thêm bằng `docker compose up` (agent + Redis thật):

```text
health:  {"status":"ok"} 200
ready:   {"status":"ready","redis":"ok"} 200
no key:  {"detail":"Invalid or missing API key"} 401
key:     {"question":"What is docker?","answer":"Container là cách đóng gói app ..."} 200
empty:   {"error":"invalid_request","details":[{"field":"question","msg":"String should have at least 1 character"}]} 422
badjson: {"error":"invalid_request","details":[{"field":"1","msg":"JSON decode error"}]} 422
# docker compose stop redis
redis down /health: 200 /ready: 503 /ask: {"error":"service_unavailable","detail":"State store unavailable, retry later"} 503
```

Các quyết định thiết kế chính:

| Thành phần | Cách làm |
|---|---|
| Auth | `APIKeyHeader(auto_error=False)` + `secrets.compare_digest`; thiếu/sai → 401; chưa cấu hình `AGENT_API_KEY` thì từ chối hết (fail closed) |
| Rate limit | Sliding window 60 s bằng Redis sorted set trong 1 transaction `MULTI/EXEC` (`ZREMRANGEBYSCORE → ZADD → ZCARD → EXPIRE`); request bị từ chối được `ZREM` để không chiếm chỗ; header `Retry-After` tính từ entry cũ nhất; key Redis là hash SHA-256 của API key, không lưu key gốc |
| Cost guard | `INCRBYFLOAT` theo ngày và theo tháng (UTC), 402 khi vượt `DAILY_BUDGET_USD` / `MONTHLY_BUDGET_USD` |
| Redis client | `redis.from_url` lazy (không ping lúc import) nên app khởi động được khi Redis chưa sẵn sàng; timeout 2 s |
| Logging | `JsonFormatter` riêng cho logger `agent` (`propagate=False`), middleware ghi `{"event":"request","path","status","ms"}` cho mọi request, không log API key |
| Lỗi | Handler riêng cho validation (422), `RedisError` (503), `Exception` (500 `internal_error`), không trả stack trace |
| Shutdown | Dùng `lifespan` của FastAPI, không tự đăng ký signal handler để không đè handler của uvicorn |
| Docker | Builder tạo venv `/opt/venv`, runtime `python:3.11-slim` chỉ copy venv + `app/` + `utils/`, user `app` non-root, `HEALTHCHECK` bằng `urllib`, `CMD sh -c "exec uvicorn ... --port ${PORT:-8000}"` để uvicorn là PID 1 |

---

## AI log

Theo [RULES.md](RULES.md) mục 1: bài này có dùng AI (Claude Code, chạy trong VS Code trên máy mình).

| # | Hỏi / yêu cầu AI | AI làm / trả lời gì | Mình tự sửa và kiểm tra thế nào |
|---|---|---|---|
| 1 | Đọc đề, tóm tắt bài lab cần làm những gì | Tóm tắt từ README, RUBRIC, SUBMISSION, CHECKPOINTS: các Part, điểm, file phải nộp, điều cấm (không sửa script chấm, không commit `.env`) | Đối chiếu lại với RUBRIC và SUBMISSION |
| 2 | Hoàn thành các `TODO` của Part 6 | Viết `app/` (config, auth, redis_client, rate_limiter, cost_guard, main), Dockerfile multi-stage, `.dockerignore`, compose, `.env.example`, `railway.toml`, `render.yaml` sau khi đọc `check_production_ready.py` | Chạy `python check_production_ready.py` bản đầy đủ: **PASS 12/12**; test tay bằng `docker compose` (401/200/422, tắt Redis → 503). Phát hiện compose không truyền `AGENT_API_KEY` từ shell (key đúng vẫn 401) → sửa compose rồi chạy lại script chấm |
| 3 | Chạy các bài Part 1, 2, 4, 5 và lấy output | AI chạy các lệnh trong đề, ghi output; phát hiện cổng 8000 bị service khác chiếm nên chạy Part 4/5 develop ở 8010 | Đọc lại output, đối chiếu mã HTTP với yêu cầu (401/403/200/429), che token trước khi dán |
| 4 | Trả lời câu hỏi Part 1–5 | Soạn câu trả lời dựa trên code mẫu và output thật; chỉ ra 2 lỗi nhỏ của code mẫu Part 1 (log JSON bị nuốt vì `basicConfig` gọi 2 lần; handler SIGTERM bị uvicorn đè) | Kiểm tra lại trong log `p1_prod` (dòng `WARNING:root:` và không có log JSON) |
| 5 | Part 7: load test, eval, sửa knowledge base, CI job | Chạy `load_test.py` 4 mức concurrency và 3 mức `MOCK_SLOW_RATE`, chạy eval v1/v2, thêm 2 tài liệu (JWT, Kubernetes HPA) vào `KNOWLEDGE_BASE`, viết `.github/workflows/eval.yml`; thêm metric `agent_question_length_chars` | Kiểm tra từ khoá mới chỉ khớp đúng 2 câu sai (grep golden dataset) để không gây regression; eval cuối **23/23, 0 regression, Quality gate PASS** |
| 6 | Part 7 stack production | Phát hiện Dockerfile của stack observability crash (`No module named 'uvicorn'`) do user `agent` có home `/app` → thêm `ENV PYTHONUSERBASE`; phát hiện `docker compose run locust` tự bật lại 2 agent nên chạy benchmark scale với `--no-deps`; chụp screenshot bằng Chrome headless | Kiểm tra số container agent trước mỗi lần chạy Locust; số trong BENCHMARK_REPORT lấy trực tiếp từ file JSON/CSV |
| 7 | Deploy agent Part 6 | Railway trial hết hạn → chuyển Render; AI sửa `render.yaml` (đường dẫn từ gốc repo, `generateValue`), validate bằng `render blueprints validate`, tạo Key Value và web service bằng Render CLI (mình tự đăng nhập CLI), sinh `AGENT_API_KEY` ngẫu nhiên lưu ngoài repo | 4 lệnh tự kiểm 200/200/401/200, `/ready` báo `redis: ok`, rate limit 429 trên cloud, `check_production_ready.py --public-url` ✅; log Render xác nhận `PORT=10000` |
| 8 | Viết `MISSION_ANSWERS.md`, `BENCHMARK_REPORT.md`, `DEPLOYMENT.md` | Soạn từ output và report đã chạy | Đối chiếu từng con số với file trong `reports/` và `benchmark-results/` |
