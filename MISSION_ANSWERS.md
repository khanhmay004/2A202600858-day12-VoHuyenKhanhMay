# Day 12 Lab — Mission Answers

> **Student:** Vo Huyen Khanh May
> **Course:** AICB-P1 · VinUniversity 2026
> **Môi trường chạy:** Windows 11 + Docker Desktop (WSL2 Ubuntu 22.04), Docker `29.5.2`, Compose `v5.1.4`.
> Tất cả ví dụ chạy **trong Docker** (không cài lib vào Python global). LLM dùng **mock** (không cần API key thật).
> Mọi output dưới đây là **kết quả chạy thật** đã chụp lại.

---

## Part 1: Localhost vs Production

### Exercise 1.1 — Anti-patterns trong [`01-localhost-vs-production/develop/app.py`](01-localhost-vs-production/develop/app.py)

1. **Hardcoded secrets** — `OPENAI_API_KEY` và `DATABASE_URL` (kèm password) viết thẳng trong code ([dòng 17–18](01-localhost-vs-production/develop/app.py#L17-L18)). Push lên GitHub là lộ ngay.
2. **Không có config management** — `DEBUG=True`, `MAX_TOKENS=500` hardcode ([dòng 21–22](01-localhost-vs-production/develop/app.py#L21-L22)); không đổi được theo môi trường mà không sửa code.
3. **`print()` thay vì logging — và in cả secret** ([dòng 33–34](01-localhost-vs-production/develop/app.py#L33-L34)): `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` vừa lộ secret trong log, vừa không structured.
4. **Không có health check endpoint** ([dòng 42](01-localhost-vs-production/develop/app.py#L42)) — platform (Railway/Render/K8s) không biết agent sống/chết để restart.
5. **Bind cứng + reload trong production** ([dòng 47–54](01-localhost-vs-production/develop/app.py#L47-L54)): `host="localhost"` (không nhận kết nối ngoài container), `port=8000` cứng (bỏ qua `$PORT` mà platform inject), `reload=True` (chỉ hợp lý cho dev).

*Bonus anti-patterns:* không có graceful shutdown, không có readiness probe, không validate input.

### Exercise 1.3 — Bảng so sánh `develop` vs `production`

So sánh [`develop/app.py`](01-localhost-vs-production/develop/app.py) với [`production/app.py`](01-localhost-vs-production/production/app.py) + [`production/config.py`](01-localhost-vs-production/production/config.py):

| Feature | Develop | Production | Tại sao quan trọng |
|---|---|---|---|
| **Config** | Hardcode trong code | Đọc từ env vars (`config.Settings`) | 12-Factor: đổi cấu hình theo môi trường, không sửa code |
| **Secrets** | Hardcode + `print()` ra log | Từ env, không bao giờ log | Bảo mật — tránh lộ key |
| **Logging** | `print()` | Structured **JSON** logging | Parse được bởi Loki/Datadog/CloudWatch |
| **Health check** | Không có | `GET /health` + `/ready` + `/metrics` | Liveness/readiness cho platform & load balancer |
| **Binding** | `localhost:8000` cứng | `0.0.0.0` + `PORT` từ env | Chạy được trong container; platform inject `PORT` |
| **Reload** | `reload=True` luôn | Chỉ reload khi `DEBUG=true` | Ổn định & hiệu năng trong production |
| **Graceful shutdown** | Không | SIGTERM handler + lifespan drain | Không rớt request đang xử lý khi deploy/restart |

**Checkpoint:** ✅ Hiểu vì sao hardcode secret nguy hiểm · ✅ Dùng env vars · ✅ Vai trò health check · ✅ Graceful shutdown là gì.

---

## Part 2: Docker

### Exercise 2.1 — Phân tích [`02-docker/develop/Dockerfile`](02-docker/develop/Dockerfile)

1. **Base image?** `python:3.11` (full distribution, ~1 GB).
2. **Working directory?** `/app` (`WORKDIR /app`).
3. **Vì sao copy `requirements.txt` TRƯỚC khi copy code?** Để tận dụng **Docker layer cache**: layer cài dependencies chỉ rebuild khi `requirements.txt` đổi. Sửa code (`app.py`) không làm cài lại lib → build nhanh hơn nhiều.
4. **CMD vs ENTRYPOINT?**
   - `CMD` = lệnh mặc định, **dễ override** bằng `docker run <image> <lệnh khác>`. Ở đây `CMD ["python", "app.py"]`.
   - `ENTRYPOINT` = executable cố định, args của `docker run` được **nối thêm** vào sau. Thường dùng khi container đóng vai trò như 1 binary cố định.

### Exercise 2.2 / 2.3 — Build & so sánh image size (kết quả thật)

```text
$ docker build -f 02-docker/develop/Dockerfile     -t my-agent:develop .
$ docker build -f 02-docker/production/Dockerfile  -t my-agent:production .
$ docker images | findstr my-agent
my-agent:production   236MB
my-agent:develop      1.66GB
```

| Image | Kiểu build | Base | Size |
|---|---|---|---|
| `my-agent:develop` | Single-stage | `python:3.11` (full) | **1.66 GB** |
| `my-agent:production` | Multi-stage | `python:3.11-slim` | **236 MB** |

- **Chênh lệch:** `(1660 − 236) / 1660 ≈ 85.8%` nhỏ hơn (~**7×**).
- Production **< 500 MB** ✅ (đạt yêu cầu lab). Lab-06 image = **247 MB**.
- **Vì sao nhỏ hơn?** Multi-stage: stage `builder` chứa gcc + build tools để cài deps; stage `runtime` chỉ copy site-packages đã cài + dùng base `slim`, không mang theo toolchain. Cộng thêm chạy **non-root user** + `HEALTHCHECK`.

Smoke-test image production: `GET /health` → `{"status":"ok","version":"2.0.0",...}` (HTTP 200).

**Checkpoint:** ✅ Hiểu cấu trúc Dockerfile · ✅ Lợi ích multi-stage · ✅ docker-compose orchestration · ✅ debug container (`docker logs`/`docker exec`).

---

## Part 3: Cloud Deployment

> **Deploy thật:** đã chọn **Render** (Docker runtime). URL công khai + bằng chứng test xem trong [`DEPLOYMENT.md`](DEPLOYMENT.md).

So sánh 3 platform (từ [`railway.toml`](03-cloud-deployment/railway/railway.toml), [`render.yaml`](03-cloud-deployment/render/render.yaml), [`cloudbuild.yaml`](03-cloud-deployment/production-cloud-run/cloudbuild.yaml)):

| Tiêu chí | Railway | Render | GCP Cloud Run |
|---|---|---|---|
| **Builder** | Nixpacks (auto) hoặc Dockerfile | `runtime: python` (hoặc docker) | Dockerfile + Cloud Build |
| **Start command** | `uvicorn app:app --host 0.0.0.0 --port $PORT` | `uvicorn app:app ... --port $PORT` | `gcloud run deploy ... --image ...` |
| **Health check** | `healthcheckPath = /health` | `healthCheckPath: /health` | health qua container `/health` |
| **PORT** | Platform inject `$PORT` | Platform inject `$PORT` | Platform inject `$PORT` (8080) |
| **Secrets** | `railway variables set ...` | `sync: false` (set trên dashboard) | Secret Manager (`--set-secrets`) |
| **Autoscale** | Cơ bản | Cơ bản (free: ngủ sau ~15') | `--min-instances 1 --max-instances 10` |
| **CI/CD** | Auto trên push | `autoDeploy: true` trên push | `cloudbuild.yaml`: test → build → push → deploy |
| **Độ khó / Free tier** | ⭐ / $5 credit | ⭐⭐ / free (web + redis) | ⭐⭐⭐ / 2M req free |

**Nhận xét:** Railway nhanh nhất cho prototype; Render = Infrastructure-as-Code (`render.yaml`) dễ tái lập, có Redis add-on free; Cloud Run mạnh nhất (CI/CD, min/max instances, Secret Manager) nhưng nhiều bước nhất.

**Checkpoint:** ✅ Deploy thành công ≥1 platform (Render) · ✅ Public URL chạy được · ✅ Set env vars trên cloud · ✅ Xem logs.

---

## Part 4: API Security

Chạy [`04-api-gateway/production`](04-api-gateway/production) (JWT + Rate Limit + Cost Guard) trong container. **Output thật:**

### Exercise 4.1–4.3 — Auth + Rate limiting

```text
=== /health ===
{"status":"ok","security":"JWT + RateLimit + CostGuard",...}

1) /ask WITHOUT token            → HTTP 401   (chặn truy cập thiếu auth)
2) POST /auth/token (student/demo123) → access_token (JWT, HS256, exp 60')
3) /ask WITH Bearer token        → HTTP 200
   {"answer":"Container là cách đóng gói app...","usage":{"requests_remaining":9,...}}

4) Rate limit (user = 10 req/phút), bắn liên tục:
   Request 1..9  → HTTP 200
   Request 10    → HTTP 429   ← (đã có 1 request ở bước 3, nên tổng 10 thành công rồi chặn)
   Request 11,12 → HTTP 429
```

- **Auth ở đâu, fail thì sao?** Dependency `verify_token` ([`auth.py`](04-api-gateway/production/auth.py#L46)) decode JWT bằng `JWT_SECRET`/HS256 → thiếu/sai token raise **401/403**.
- **Rate limit:** [`rate_limiter.py`](04-api-gateway/production/rate_limiter.py) — **sliding window** (deque timestamps trong 60s), user `10/phút`, admin `100/phút`; vượt → **429** kèm header `Retry-After`, `X-RateLimit-*`.
- **Auth bằng API Key** (kiểu đơn giản hơn) được minh hoạ ở Lab-06: `X-API-Key` sai/thiếu → **401**, đúng → **200** (xem Part 6).

### Exercise 4.4 — Cost Guard (giải thích cách làm)

[`cost_guard.py`](04-api-gateway/production/cost_guard.py): class `CostGuard` chống "bill sốc" từ LLM.
- **Pricing** (GPT-4o-mini tham khảo): input `$0.15/1M` tokens, output `$0.60/1M` tokens.
- **Hai mức ngân sách:** per-user `$1/ngày` và global `$10/ngày`.
- **`check_budget(user)`** gọi **trước** khi gọi LLM: vượt per-user → **402 Payment Required**; vượt global → **503**; đạt 80% → log cảnh báo.
- **`record_usage(user, in, out)`** gọi **sau** khi có response: cộng tokens, tính cost, cập nhật global. Reset theo ngày (`%Y-%m-%d`).
- **Hạn chế (đúng như comment trong file):** đang lưu **in-memory** → reset khi restart, không chia sẻ giữa nhiều instance. **Production nên lưu Redis** (key kiểu `cost:{user}:{YYYY-MM-DD}` + TTL) để cộng dồn atomic và đúng khi scale ngang.

**Luồng bảo vệ:** `Auth (401) → Rate limit (429) → Validation (422) → Cost (402/503) → Agent (200)`.

**Checkpoint:** ✅ API key auth · ✅ JWT flow · ✅ Rate limiting · ✅ Cost guard (+ hiểu vì sao cần Redis khi scale).

---

## Part 5: Scaling & Reliability

### Exercise 5.1 — Health & Readiness (kết quả thật, [`05-scaling-reliability/develop`](05-scaling-reliability/develop))

```text
GET /health → {"status":"ok","uptime_seconds":1.2,"version":"1.0.0","checks":{"memory":{"status":"ok","used_percent":7.1}}}
GET /ready  → {"ready":true,"in_flight_requests":1}
```
- **Liveness (`/health`)**: "còn sống không?" → non-200 thì platform **restart** container.
- **Readiness (`/ready`)**: "sẵn sàng nhận traffic chưa?" → **503** khi đang khởi động / shutdown / dependency (Redis) chưa sẵn sàng → load balancer **ngừng route** vào instance đó.

### Exercise 5.2 — Graceful shutdown (kết quả thật)

Gửi `SIGTERM` (qua `docker stop`) khi agent đang chạy → log:
```text
INFO: Started server process [1]
INFO: Application startup complete.
INFO: Shutting down
2026-06-12 07:33:50 INFO 🔄 Graceful shutdown initiated...
2026-06-12 07:33:50 INFO ✅ Shutdown complete
INFO: Application shutdown complete.
2026-06-12 07:33:50 INFO Received signal 15 — uvicorn will handle graceful shutdown
```
- Lifespan đặt `_is_ready=False` (request mới nhận 503), rồi **chờ in-flight requests xong** (tối đa 30s) trước khi thoát; `timeout_graceful_shutdown=30`.
- **Lưu ý kỹ thuật khi test bằng Docker:** phải để process Python là **PID 1** (`exec python ...`), nếu không SIGTERM đi vào `sh` chứ không vào uvicorn → shutdown không graceful.

### Exercise 5.3 — Stateless design

[`05-scaling-reliability/production/app.py`](05-scaling-reliability/production/app.py): session/history **không giữ trong memory** mà lưu **Redis** (`save_session`/`load_session`/`append_to_history`, key `session:{id}`, TTL 3600s). Nhờ vậy **bất kỳ instance nào** cũng đọc/ghi được state của user.

### Exercise 5.4 — Load balancing

[`nginx.conf`](05-scaling-reliability/production/nginx.conf): `upstream agent_cluster { server agent:8000; }` round-robin tới các instance qua Docker DNS; header `X-Served-By $upstream_addr`.

### Exercise 5.5 — Test stateless khi scale=3 (kết quả thật)

`docker compose up --scale agent=3` (3 agent + redis + nginx) rồi chạy `test_stateless.py` qua nginx:
```text
Session ID: 3305b7a5-36c0-4f10-9f0c-f6345c92418d
Request 1: [instance-69c335]   Request 2: [instance-b1d4ad]
Request 3: [instance-69c335]   Request 4: [instance-487ee1]
Request 5: [instance-b1d4ad]
Instances used: {'instance-b1d4ad', 'instance-69c335', 'instance-487ee1}
✅ All requests served despite different instances!
--- Conversation History --- Total messages: 10
✅ Session history preserved across all instances via Redis!
```
→ 5 request được phục vụ bởi **3 instance khác nhau**, nhưng **10 messages vẫn liền mạch** nhờ Redis. Đây là minh chứng rõ nhất cho stateless + horizontal scaling.

**Checkpoint:** ✅ Health/readiness · ✅ Graceful shutdown · ✅ Stateless (Redis) · ✅ Load balancing (nginx) · ✅ Test multi-instance.

---

## Part 6: Final Project — Lab 06 (kết quả thật)

Source = [`06-lab-complete/`](06-lab-complete/) (giữ nguyên logic tham chiếu). Chạy `docker compose up --build`:

```text
GET /health → {"status":"ok","version":"1.0.0","environment":"staging","checks":{"llm":"mock"}}
GET /ready  → {"ready":true}
POST /ask  (no  X-API-Key)  → HTTP 401
POST /ask  (with X-API-Key) → HTTP 200  {"answer":"Deployment là quá trình đưa code..."}
Rate limit (1 worker, 20/min): request 1..20 → 200, request 21..23 → 429
GET /metrics → {"total_requests":25,"error_count":0,"daily_cost_usd":0.0004,"daily_budget_usd":5.0}
Security headers: x-content-type-options: nosniff · x-frame-options: DENY
```

**`python check_production_ready.py` → 20/20 (100%) — 🎉 PRODUCTION READY.**
Image multi-stage = **247 MB** (< 500 MB).

### Ghi chú trung thực về bản tham chiếu (đã sửa các lỗi hạ tầng để chạy được)

Bản `06-lab-complete` ban đầu **không chạy được**; mình chỉ sửa lỗi **hạ tầng/compat** (không đổi tính năng):
1. **`utils/` thiếu trong build context** → copy `utils/mock_llm.py` vào `06-lab-complete/utils/` cho thư mục tự chứa (đúng cấu trúc deliverable).
2. **Dockerfile sai home user** → `-d /app` trong khi package `--user` nằm ở `/home/agent/.local` ⇒ Python không tìm thấy `uvicorn` (crash loop). Sửa home thành `/home/agent` (giống Dockerfile 02-production đang chạy tốt).
3. **`MutableHeaders.pop` không tồn tại** ([main.py](06-lab-complete/app/main.py)) ⇒ **mọi request 500**. Sửa thành `if "server" in headers: del headers["server"]` — **đúng cách mà repo đã fix sẵn ở `04-api-gateway`**.

### Hạn chế đã biết (in-memory) — đúng tinh thần bài học

- Rate limiter & cost guard của Lab-06 là **in-memory per-process**. Với `--workers 2` (CMD mặc định), mỗi worker giữ bucket riêng ⇒ limit thực tế bị "nhân đôi" và không nhất quán (khi test 23 request qua 2 worker, **không** thấy 429). Vì vậy khi deploy/test rate-limit, mình chạy **1 worker** để giới hạn xác định (đã thấy 429 sau request thứ 20).
- Muốn **đúng stateless** khi scale ngang nhiều instance → chuyển rate-limit/cost-guard sang **Redis** (như Section 05 đã làm cho session). Đây là hướng nâng cấp tiếp theo.

---

## Tổng kết

| Part | Trạng thái | Bằng chứng |
|---|---|---|
| 1 — Localhost vs Production | ✅ | Anti-patterns + bảng so sánh (đọc code) |
| 2 — Docker | ✅ | Build thật: 1.66 GB vs 236 MB; smoke-test /health |
| 3 — Cloud Deployment | ✅ | So sánh 3 platform + deploy Render → `DEPLOYMENT.md` |
| 4 — API Security | ✅ | 401/200, JWT, rate-limit 429, cost guard |
| 5 — Scaling & Reliability | ✅ | health/ready, graceful shutdown, stateless 3-instance |
| 6 — Lab Complete | ✅ | check_production_ready 100%, image 247 MB, deploy Render |
