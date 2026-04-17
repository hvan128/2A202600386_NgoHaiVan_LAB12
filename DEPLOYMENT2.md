# Deployment 2 — Vinmec AI Agent (dự án chính — Lab 12 submission)

> **Student:** Ngô Hải Văn (2A202600386)
> **Date:** 2026-04-17
> **Repo:** https://github.com/hvan128/Lab12_Vinmec_2A202600386
> **Báo cáo chi tiết:** `BAO_CAO_LAB12_VINMEC.md` trong repo

---

## Public URL

**Production:** https://lab12.hvan.it.com

Dự án thật đang chạy 24/7 — Next.js 14 chatbot Vinmec với streaming gpt-4o-mini,
Postgres persistence, full CI/CD pipeline.

---

## Platform

| Component | Chi tiết |
|-----------|---------|
| **Hosting** | VPS CentOS `root@157.66.100.59` + Docker Compose |
| **Edge / HTTPS** | Cloudflare Tunnel (`--protocol http2`) — không mở port VPS |
| **Container registry** | GHCR — 2 images: `...:latest` (runtime, 345 MB) + `...-migrate:latest` (full node_modules) |
| **CI/CD** | GitHub Actions: push `main` → build → push GHCR → SSH deploy → smoke test |
| **Database** | Postgres 16 (persistent volume), auto-migrate + seed mỗi deploy |
| **App port layout** | VPS `localhost:3003` → container `:3000` (Next.js standalone) |

> **Không dùng Railway/Render/Cloud Run** — chọn VPS + GHA để học CI/CD pipeline
> thật và tránh giới hạn free tier. Xem `DEPLOYMENT1.md` cho sample Python
> deploy Railway/Render.

---

## Demo API Key (dùng cho grader chấm Lab 12)

UI `https://lab12.hvan.it.com` **không cần key** (same-origin tự bypass).
Chỉ external test tools (curl/Postman) cần key:

```
AGENT_API_KEY=1e93a43bffdd1906cd5828943dd79b5ef5e99350103bcde32b34011f75ee945b
```

Bảo vệ:
- Rate-limit **10 req/min/bucket**
- Cost guard **$0.5/tháng/bucket** (dễ trigger 402 khi test)
- Key rotate sau deadline 17/4/2026: `gh secret set AGENT_API_KEY ...`

Seed users có sẵn: `user-an`, `user-binh`, `user-cuong`.

---

## Test Commands

### 1. Health Check

```bash
curl https://lab12.hvan.it.com/api/health
```

Expected:
```json
{
  "status": "ok",
  "version": "<git-sha>",
  "environment": "production",
  "uptime_seconds": 42,
  "timestamp": "2026-04-17T09:00:00.000Z"
}
```

### 2. Readiness Check (DB + OpenAI key)

```bash
curl https://lab12.hvan.it.com/api/ready
# → {"ready":true,"checks":{"database":"ok","openai_key":"ok"}}
```

### 3. Auth required (no key → 401)

```bash
curl -X POST https://lab12.hvan.it.com/api/chat \
  -H "Content-Type: application/json" \
  -d '{"messages":[],"userId":"test"}'
# → 401 {"error":"Missing API key. Include header: X-API-Key: <key>"}
```

### 4. Auth OK (with valid key → streaming 200)

```bash
export API_KEY=1e93a43bffdd1906cd5828943dd79b5ef5e99350103bcde32b34011f75ee945b

curl -X POST https://lab12.hvan.it.com/api/chat \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages":[{"role":"user","content":"Xin chào, tôi cần đặt lịch khám"}],"userId":"user-an"}'
# → text/event-stream từ gpt-4o-mini
```

### 5. Rate Limit (10 req/min)

```bash
for i in {1..12}; do
  curl -s -o /dev/null -w "req $i: %{http_code}\n" \
    -X POST https://lab12.hvan.it.com/api/chat \
    -H "X-API-Key: $API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"messages":[{"role":"user","content":"ping"}],"userId":"t"}'
done
# req 1-10: 200
# req 11-12: 429 (Retry-After: 60)
```

### 6. Metrics (protected)

```bash
curl -H "X-API-Key: $API_KEY" https://lab12.hvan.it.com/api/metrics
# → {"month":"2026-04","monthlyBudget":0.5,"keys":[{"key":"1e93a43b","spentUsd":0.0034}]}
```

### 7. Cost Guard demo (spam → 402)

```bash
# ~600 requests > $0.5 → HTTP 402
for batch in {1..60}; do
  for i in {1..10}; do
    curl -s -o /dev/null -w "batch $batch req $i: %{http_code}\n" \
      -X POST https://lab12.hvan.it.com/api/chat \
      -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
      -d '{"messages":[{"role":"user","content":"ping"}],"userId":"stress"}'
  done
  sleep 61  # đợi rate-limit window reset
done
# Sau khi chi vượt $0.5: HTTP 402 {"error":"Monthly budget exceeded ($0.5)..."}
```

---

## Environment Variables (production)

Set tự động bởi GitHub Actions deploy workflow vào `/opt/lab12-vinmec/.env` trên VPS.

| Variable | Value | Nguồn |
|----------|-------|-------|
| `DATABASE_URL` | `postgresql://postgres:***@db:5432/vinmec_ai?schema=public` | Compose internal DNS |
| `OPENAI_API_KEY` | `sk-proj-...` | GitHub Secret |
| `OPENAI_MODEL` | `gpt-4o-mini` | Hardcoded |
| `AGENT_API_KEY` | 64-byte hex | GitHub Secret (generated) |
| `ADMIN_KEY` | `vinmec-demo-2026` | GitHub Secret |
| `CLOUDFLARE_TUNNEL_TOKEN` | `eyJ...` | GitHub Secret |
| `POSTGRES_PASSWORD` | 48-char hex | GitHub Secret (generated) |
| `RATE_LIMIT_PER_MINUTE` | `10` | Workflow |
| `MONTHLY_BUDGET_USD` | `0.5` | Workflow |
| `REQUIRE_AUTH` | `true` | Workflow |
| `NEXT_PUBLIC_APP_URL` | `https://lab12.hvan.it.com` | Workflow |
| `APP_VERSION` | `${GITHUB_SHA}` | Workflow (per deploy) |

---

## CI/CD Pipeline

```
git push origin main
    │
    ▼
┌───────────────────────────────────────┐
│ GitHub Actions: build (3-4 phút)      │
│  - Checkout, Buildx cache             │
│  - Build 2 images (runtime + migrate) │
│  - Push GHCR (:latest + :sha7)        │
└──────────────────┬────────────────────┘
                   ▼
┌───────────────────────────────────────┐
│ deploy (30 giây)                      │
│  - SSH root@157.66.100.59             │
│  - scp docker-compose.yml             │
│  - Write .env atomically              │
│  - docker compose --profile tunnel    │
│    pull && up -d                      │
│    (migrate auto-run: depends_on      │
│     service_completed_successfully)   │
└──────────────────┬────────────────────┘
                   ▼
┌───────────────────────────────────────┐
│ smoke-test (10-60 giây)               │
│  - curl /api/health → 200             │
│  - POST /api/chat no-key → 401        │
└───────────────────────────────────────┘
```

**Total: ~5 phút** từ `git push` đến production.

---

## GitHub Secrets

| Secret | Mô tả |
|--------|-------|
| `VPS_HOST` | `157.66.100.59` |
| `VPS_USER` | `root` |
| `VPS_SSH_KEY` | SSH private key để GHA đăng nhập VPS |
| `OPENAI_API_KEY` | Gọi OpenAI API |
| `AGENT_API_KEY` | Middleware auth cho `/api/chat` |
| `ADMIN_KEY` | Query param key cho `/admin/feedback` |
| `CLOUDFLARE_TUNNEL_TOKEN` | Tunnel từ Cloudflare → VPS |
| `POSTGRES_PASSWORD` | DB password |

Set tất cả qua `gh secret set <NAME> --repo hvan128/Lab12_Vinmec_2A202600386`.

---

## Cloudflare Tunnel Config

- Tunnel name: `lab12-vinmec`
- Public Hostname: `lab12.hvan.it.com` → `HTTP://localhost:3003`
- Protocol: HTTP/2 (TCP 443) — fallback vì VPS firewall block QUIC/UDP
- HTTPS tự động, không mở port 80/443 trên VPS

---

## Clone repo → chạy local

```bash
git clone https://github.com/hvan128/Lab12_Vinmec_2A202600386.git
cd Lab12_Vinmec_2A202600386
cp .env.example .env  # điền OPENAI_API_KEY
docker compose up --build
# → http://localhost:3003
```

Flow tự động: build 2 images → start postgres → migrate + seed (users, doctors, faqs,…) → start app. Bỏ qua cloudflared (profile `tunnel` chỉ activate trên VPS).

---

## Production Readiness — 9/9 checklist Lab 12

| # | Yêu cầu | Trạng thái |
|---|---------|-----------|
| 1 | Dockerfile multi-stage < 500 MB | ✅ 345 MB (3-stage, non-root, tini) |
| 2 | API key auth | ✅ `X-API-Key`, 401 khi sai |
| 3 | Rate limit 10 req/min | ✅ Sliding window (verified 429 req 11+) |
| 4 | Cost guard $/tháng | ✅ Per-key monthly budget (hiện $0.5 để test) |
| 5 | `/health` + `/ready` | ✅ Liveness + readiness check DB + OpenAI |
| 6 | Graceful shutdown SIGTERM | ✅ `tini` PID 1 + `stop_grace_period: 30s` |
| 7 | Stateless | ✅ Conversation + feedback trong Postgres |
| 8 | Config env vars, không hardcode | ✅ `.env` runtime, `.gitignore`, GitHub Secrets |
| 9 | Public URL | ✅ https://lab12.hvan.it.com |

**Extras vượt chuẩn:**
- ✅ CI/CD tự động (GitHub Actions → GHCR → VPS)
- ✅ Auto seed DB (3 users + 30 doctors + 10 depts + 30 faqs + 7 branches + 10 guides)
- ✅ Post-deploy smoke test
- ✅ Protected `/api/metrics` endpoint (cost audit)
- ✅ Same-origin UI bypass (UX + security balance)
- ✅ Separate migrate image (full node_modules, không bloat runtime)
