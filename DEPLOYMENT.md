# Deployment Information

> Agent được deploy là agent Part 6 trong `06-lab-complete/` (đạt `PASS 12/12`), kèm một Redis trên cùng nền tảng.

## Public URL

<!-- TODO: điền URL sau khi chạy `railway domain` -->
https://<your-agent>.up.railway.app

## Platform

Railway (build bằng `06-lab-complete/Dockerfile`, cấu hình trong `06-lab-complete/railway.toml`)

## Cách deploy

```bash
npm i -g @railway/cli
railway login
cd 06-lab-complete
railway init                       # tạo project mới
railway add --database redis       # tạo Redis trong cùng project
railway up                         # tạo service agent từ Dockerfile
# Đặt biến cho service agent (KHÔNG ghi giá trị thật vào repo):
railway variables --set "AGENT_API_KEY=<random-key>" \
                  --set 'REDIS_URL=${{Redis.REDIS_URL}}' \
                  --set "ENVIRONMENT=production" \
                  --set "RATE_LIMIT_PER_MINUTE=10" \
                  --set "DAILY_BUDGET_USD=1.0" \
                  --set "MONTHLY_BUDGET_USD=10.0"
railway up                         # redeploy với biến mới
railway domain                     # nhận public URL
```

`PORT` do Railway tự cấp; `CMD` trong Dockerfile đọc `${PORT:-8000}`.

## Test Commands

Đặt `BASE` là URL public và `AGENT_API_KEY` là key đã đặt trên Railway. Mỗi lệnh chỉ in mã HTTP, chụp màn hình không lộ key.

```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"     # 200
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"      # 200
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -d '{"question":"Hello"}'   # 401
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" -H 'Content-Type: application/json' -H "X-API-Key: $AGENT_API_KEY" -d '{"question":"Hello"}'   # 200
```

Kết quả:

```text
TODO: dán output 4 lệnh trên (mong đợi 200, 200, 401, 200)
```

Kiểm tra thêm bằng script chấm:

```bash
cd 06-lab-complete && python check_production_ready.py --public-url "$BASE"
```

## Environment Variables Set

| Biến | Giá trị | Nguồn |
|---|---|---|
| `PORT` | do Railway cấp | Railway |
| `AGENT_API_KEY` | (secret, không ghi ở đây) | `railway variables` |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` (tham chiếu tới service Redis) | `railway variables` |
| `ENVIRONMENT` | `production` | `railway variables` |
| `RATE_LIMIT_PER_MINUTE` | `10` | `railway variables` |
| `DAILY_BUDGET_USD` | `1.0` | `railway variables` |
| `MONTHLY_BUDGET_USD` | `10.0` | `railway variables` |
| `LOG_LEVEL` | `INFO` (mặc định trong code) | — |

## Screenshots

- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
