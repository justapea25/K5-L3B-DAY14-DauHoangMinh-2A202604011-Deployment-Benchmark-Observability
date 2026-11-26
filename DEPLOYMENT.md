# Deployment Information

> Agent được deploy là agent Part 6 trong `06-lab-complete/` (đạt `PASS 12/12`), kèm một Render Key Value (Redis-compatible) cùng region.

## Public URL

https://day14-agent-43am.onrender.com

## Platform

**Render** (gói Free, region Singapore), build từ `06-lab-complete/Dockerfile`, cấu hình tương ứng `06-lab-complete/render.yaml`.

> Lúc đầu định dùng Railway (`railway.toml` vẫn giữ trong repo), nhưng tài khoản Railway đã hết trial ("Your trial has expired. Please select a plan to continue using Railway") nên chuyển sang Render, theo RUBRIC chọn Railway hoặc Render đều được.
>
> Gói Free của Render cho web service ngủ sau ~15 phút không có traffic; request đầu tiên sau đó mất khoảng 30–60 giây để khởi động lại (cold start). Nếu lệnh test đầu tiên bị timeout, chờ một lúc rồi chạy lại.

## Cách deploy

| Tài nguyên | Loại | Plan / region | ID |
|---|---|---|---|
| `day14-agent` | Web service, runtime Docker, root directory `06-lab-complete` | free / singapore | `srv-dc3un2nf3r2c739p91eg` |
| `day14-redis` | Key Value (Redis 8.1), không mở ra internet | free / singapore | `red-dc3umqt6e07s73dlfkig` |

Cách 1 — Blueprint (dashboard): New → Blueprint → chọn repo → Blueprint Path `06-lab-complete/render.yaml` → Apply. `render.yaml` đã qua `render blueprints validate` (`"valid": true`).

Cách 2 — Render CLI (cách đã dùng thực tế):

```bash
render login
render workspace set <workspace-id>
render kv create --name day14-redis --plan free --region singapore
render services create --name day14-agent --type web_service --runtime docker \
  --repo https://github.com/<user>/<repo> --branch main --root-directory 06-lab-complete \
  --plan free --region singapore --health-check-path /health \
  --build-filter-path '06-lab-complete/**' \
  --build-filter-ignored-path '06-lab-complete/benchmark-observability/**' \
  --env-var ENVIRONMENT=production \
  --env-var AGENT_API_KEY=<random, sinh bằng openssl rand -hex 24> \
  --env-var REDIS_URL=redis://<key-value-id>:6379 \
  --env-var RATE_LIMIT_PER_MINUTE=10 --env-var DAILY_BUDGET_USD=1.0 --env-var MONTHLY_BUDGET_USD=10.0
```

`PORT` do Render tự cấp (10000); `CMD` trong Dockerfile đọc `${PORT:-8000}` nên không cần cấu hình.

## Test Commands

`BASE` là URL public, `AGENT_API_KEY` là key đã đặt trên Render (không ghi trong repo). Mỗi lệnh chỉ in mã HTTP nên chụp màn hình không lộ key.

```bash
BASE=https://day14-agent-43am.onrender.com
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả:

```text
$ curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"
200
$ curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"
200
$ curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" ... (không có key)
401
$ curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" ... -H "X-API-Key: $AGENT_API_KEY"
200

$ curl -s "$BASE/ready"
{"status":"ready","redis":"ok"}
$ curl -s -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"What is docker?"}'
{"question":"What is docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}

# Rate limit trên cloud (RATE_LIMIT_PER_MINUTE=10, đã dùng 2 request ở trên):
200 200 200 200 200 200 200 200 429 429 429 429

$ python check_production_ready.py --static-only --public-url "$BASE" | grep "Public URL"
     ✅ Public URL: GET https://day14-agent-43am.onrender.com/health → 200
```

Log khởi động trên Render (`render logs`), mỗi dòng log của app là JSON:

```text
{"ts": "2026-11-26T08:20:17.553373+00:00", "lvl": "INFO", "logger": "agent", "event": "startup", "environment": "production", "port": 10000, "redis_ok": true}
INFO:     Uvicorn running on http://0.0.0.0:10000 (Press CTRL+C to quit)
```

Deploy `dep-dc3un2vf3r2c739p9240` từ commit `8c7e073`, trạng thái **Live** (bắt đầu 08:19:23Z, xong 08:20:21Z).

## Environment Variables Set

| Biến | Giá trị | Ghi chú |
|---|---|---|
| `PORT` | do Render cấp | không tự đặt |
| `AGENT_API_KEY` | (secret, không ghi ở đây) | sinh ngẫu nhiên 48 ký tự hex |
| `REDIS_URL` | `redis://red-dc3umqt6e07s73dlfkig:6379` | địa chỉ nội bộ của Key Value, chỉ truy cập được từ service Render cùng region |
| `ENVIRONMENT` | `production` | |
| `RATE_LIMIT_PER_MINUTE` | `10` | |
| `DAILY_BUDGET_USD` | `1.0` | |
| `MONTHLY_BUDGET_USD` | `10.0` | |
| `LOG_LEVEL` | `INFO` | mặc định trong code |

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
- [Render logs — log JSON của agent](screenshots/render-logs.png)
