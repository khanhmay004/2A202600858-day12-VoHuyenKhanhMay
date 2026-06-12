# Deployment Information — Day 12 Lab

> **Student:** Vo Huyen Khanh May · **Course:** AICB-P1 · VinUniversity 2026
> **Deployed:** 2026-06-12 · all tests below are **real captured output**.

## Public URL

**https://ai-agent-production-rhdw.onrender.com**

## Platform

**Render** — Web Service · **Docker** runtime · **Free** plan · region **Singapore**.

- **Source:** GitHub `khanhmay004/day12_ha-tang-cloud_va_deployment`, branch `main`, **Root Directory = `06-lab-complete`**.
- **Build:** multi-stage Dockerfile → image ~**247 MB**, non-root user, `HEALTHCHECK`.
- **Start command** (override để dùng `$PORT` của Render + 1 worker cho rate-limit nhất quán):
  ```
  uvicorn app.main:app --host 0.0.0.0 --port $PORT --workers 1
  ```
- **Health check path:** `/health` · **Auto-deploy:** bật (push `main` → Render tự build lại).

> ⚠️ **Free tier ngủ sau ~15 phút** không có request → request đầu tiên sau khi ngủ mất ~50s (cold start), các request sau nhanh bình thường.

## Environment Variables (set trên Render — KHÔNG commit vào repo)

| Key | Value |
|---|---|
| `ENVIRONMENT` | `production` |
| `AGENT_API_KEY` | *secret* |
| `JWT_SECRET` | *secret* |
| `DAILY_BUDGET_USD` | `5.0` |
| `RATE_LIMIT_PER_MINUTE` | `20` |
| `OPENAI_API_KEY` | *(để trống → dùng mock LLM, không tốn tiền)* |

> Theo nguyên tắc 12-Factor + checklist nộp bài: **không có secret trong code/repo**; tất cả đặt qua env var trên platform.

## Test Commands & Results

> Thay `<AGENT_API_KEY>` bằng key đã đặt trên Render.

### 1. Health check (public)
```bash
curl https://ai-agent-production-rhdw.onrender.com/health
```
```json
{"status":"ok","version":"1.0.0","environment":"production","uptime_seconds":88.9,"total_requests":28,"checks":{"llm":"mock"},"timestamp":"2026-06-12T08:27:28Z"}
```
→ **HTTP 200**

### 2. Readiness probe (public)
```bash
curl https://ai-agent-production-rhdw.onrender.com/ready
```
```json
{"ready":true}
```
→ **HTTP 200**

### 3. Authentication required — không key → 401
```bash
curl -X POST https://ai-agent-production-rhdw.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"hello"}'
```
→ **HTTP 401** (chặn truy cập thiếu auth)

### 4. Với API key → 200
```bash
curl -X POST https://ai-agent-production-rhdw.onrender.com/ask \
  -H "X-API-Key: <AGENT_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is cloud deployment?"}'
```
```json
{"question":"What is cloud deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","model":"gpt-4o-mini","timestamp":"2026-06-12T08:27:29Z"}
```
→ **HTTP 200**

### 5. Rate limiting (20 req/phút → 429)
```bash
for i in $(seq 1 24); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST https://ai-agent-production-rhdw.onrender.com/ask \
    -H "X-API-Key: <AGENT_API_KEY>" \
    -H "Content-Type: application/json" \
    -d '{"question":"spam"}'
done
```
Kết quả thật (đã có 1 request thành công ở bước 4, nên tổng 20 request/phút thì bắt đầu chặn):
```text
Request  1..19 : HTTP 200
Request 20..24 : HTTP 429   ← vượt 20 req/phút → Too Many Requests
```

### Security headers (trên mọi response)
```text
HTTP/1.1 200 OK
x-content-type-options: nosniff
x-frame-options: DENY
```

## Screenshots

Đặt ảnh chụp vào thư mục `screenshots/`:
- `screenshots/render-dashboard.png` — service trạng thái **Live** trên Render
- `screenshots/health-200.png` — `/health` trả 200 (trình duyệt hoặc terminal)
- `screenshots/ratelimit-429.png` — nhận **429** khi vượt rate limit

> *(Phần screenshot bạn tự chụp từ Render dashboard + terminal — mình không truy cập được trình duyệt của bạn.)*

## Self-test (theo [DAY12_DELIVERY_CHECKLIST.md](DAY12_DELIVERY_CHECKLIST.md))

- [x] `/health` trả **200**
- [x] `/ask` không key → **401**
- [x] `/ask` có key → **200**
- [x] Rate limit → **429** sau 20 req/phút
- [x] Public URL truy cập được từ Internet
- [x] Không có secret trong repo (chỉ `.env.example`; key đặt trên Render)
