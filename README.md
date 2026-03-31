# Balanced Architecture (Detailed)

## Your Current Stack (From Code Analysis)

| Layer | What you have |
|-------|--------------|
| API | FastAPI + Uvicorn |
| Task Queue | **Celery** with Redis broker (already implemented) |
| Queues | `ai_processing`, `reports`, `default` (SLA, notifications, webhooks) |
| Scheduler | **Celery Beat** — hourly SLA sweep |
| AI | OpenAI GPT-4o/5.4, DALL-E 3, TTS-1 |
| Storage | AWS S3 (PDFs, images, audio, reports) |
| Database | PostgreSQL 17 (SQLAlchemy, pool_size=10, max_overflow=20) |
| Auth | JWT in httponly cookies, RBAC (6 roles) |
| Multi-tenant | tenant_id on every query, tenant-scoped S3 paths |

---

## AWS Services — Why Each One, What It Does

### 1. ECS Fargate (API) — 2 Fixed Tasks

**What it runs:** Your FastAPI app (`uvicorn src.main:app`)

**Why Fargate, not EC2:**
- No server patching, no SSH, no OS management
- Pay per task (vCPU + memory), not per instance
- Health checks auto-restart crashed containers

**Why 2 fixed tasks, not auto-scaling:**
Your API is lightweight — the heavy work (LLM calls, report generation) is offloaded to Celery workers. The API endpoints are fast:

| Endpoint category | What it does | Avg response time |
|------------------|-------------|-------------------|
| Auth (login, refresh, me) | JWT decode, DB lookup | 50–150ms |
| Lesson list/detail | DB query with eager loading | 20–200ms |
| Workflow transitions | DB update + audit log | 50–150ms |
| Page review/score | DB update | 20–30ms |
| Upload PDF | Validate + extract text + queue Celery task | 5–30s (returns early) |
| Dashboard stats | Aggregate COUNT queries | 50–200ms |
| Notifications | Simple CRUD | 10–50ms |

The only slow API call is PDF upload, and even that returns immediately after queuing. The API itself doesn't need scaling — 2 tasks across 2 AZs gives you high availability.

**Spec:** 1 vCPU, 2 GB memory per task

---

### 2. ECS Fargate (Celery Workers) — Your Real Compute

This is where the actual work happens. You need **3 separate worker groups**, matching your existing queue routing in `celery_app.py`:

#### Worker Group A: `ai_processing` queue (1–2 tasks)

**What it runs:**
```bash
celery -A src.celery_app worker -Q ai_processing -c 2 --prefetch-multiplier=1
```

**What it processes:** `process_lesson` task from `lesson_tasks.py`

Per page, this task does:
1. `llm_service.generate_lms_content()` — GPT call (5–8s)
2. DALL-E 3 image generation (8–15s)
3. TTS audio generation (3–5s)
4. S3 uploads for image + audio (1–2s)
5. DB inserts: Page + 14 ContentOutputs + Questions + Options

**Time per page:** 18–30 seconds
**Time per 100-page lesson:** 30–50 minutes (pages run via `asyncio.gather`)

**Why 1–2 tasks:** Each task runs 2 concurrent workers (`-c 2`). Each worker holds a DB session for the full lesson duration. More workers = more DB connections held open. Since the bottleneck is OpenAI API latency (not CPU), adding more workers doesn't help much — you're waiting on network, not computing.

**Spec:** 1 vCPU, 3 GB memory (images + audio in memory)

#### Worker Group B: `reports` queue (1 task)

**What it runs:**
```bash
celery -A src.celery_app worker -Q reports -c 2 --prefetch-multiplier=1
```

**What it processes:** `generate_report` task from `report_tasks.py`

Per report, this task does:
1. Eager loads lesson + all pages + all outputs + all questions (1 DB query, 50–200ms)
2. Generates file (Excel/Word/HTML/CSV) — 500ms–3s
3. Uploads to S3 — 200–800ms
4. Updates Report record + sends notification

**Time per report:** 2–8 seconds
**Concurrency:** Low — reports are infrequent, 1 task with 2 workers is plenty

**Spec:** 0.5 vCPU, 2 GB memory (openpyxl can be memory-heavy for large lessons)

#### Worker Group C: `default` queue (1 task)

**What it runs:**
```bash
celery -A src.celery_app worker -Q default -c 4 --prefetch-multiplier=1
```

**What it processes:**
- `periodic_sla_sweep` — hourly scan for overdue lessons, sends email + in-app notifications
- `notification_tasks` — async notification delivery
- `webhook_tasks` — webhook payload delivery (HMAC-signed)

**Time per task:** < 5 seconds each
**Concurrency:** These are fast, lightweight tasks. 4 concurrent workers handles bursts easily.

**Spec:** 0.25 vCPU, 0.5 GB memory

---

### 3. Celery Beat (Scheduler) — 1 Tiny Task

**What it runs:**
```bash
celery -A src.celery_app beat
```

**What it does:** Triggers `periodic_sla_sweep` every hour (your existing `beat_schedule` config).

**Why separate task:** Beat must be a singleton — running 2 beat processes would duplicate every scheduled task. Keep it isolated.

**Spec:** 0.25 vCPU, 0.5 GB memory

---

### 4. ElastiCache Redis — Celery Broker + Result Backend

**What it does:**
- **Message broker:** Celery tasks are published here and consumed by workers
- **Result backend:** Task status (PENDING, STARTED, SUCCESS, FAILURE) stored here
- **Task tracking:** Your `celery_app.conf.task_track_started = True` uses this

**Why Redis, not SQS:**
Your `celery_app.py` already uses Redis (`REDIS_URL` env var). Redis gives you:
- Sub-millisecond message delivery (SQS has 20–50ms polling delay)
- Result backend included (SQS can't store results)
- Celery Beat persistence (stores schedule state)
- Future use: rate limiting, caching, session store

**Why ElastiCache, not self-hosted Redis:**
- Automatic failover (Multi-AZ replica)
- No patching, no backup management
- Cluster mode available if you outgrow single node

**Spec:** cache.t3.micro (1 node, Multi-AZ), 0.5 GB

---

### 5. RDS PostgreSQL 17 (Multi-AZ) — Your Database

**What it stores:** Everything — users, tenants, lessons, pages, content outputs, questions, allocations, audit logs, notifications, reports, webhooks, prompt packs.

**Why Multi-AZ:**
Your Celery workers hold long-running DB sessions (30–50 min for a lesson). If the database goes down mid-processing:
- Without Multi-AZ: lesson processing fails, worker crashes, data partially written
- With Multi-AZ: automatic failover in < 60s, worker reconnects (SQLAlchemy `pool_pre_ping=True` handles this)

**Connection pool math:**
| Component | Connections used |
|-----------|-----------------|
| API Task 1 | pool_size=10 + overflow=20 = up to 30 |
| API Task 2 | up to 30 |
| AI Worker (2 concurrent) | 2 sessions held for 30–50 min each |
| Report Worker (2 concurrent) | 2 short sessions |
| Default Worker (4 concurrent) | 4 short sessions |
| Beat | 0 (doesn't use DB directly) |
| **Total worst case** | ~68 connections |

RDS db.t3.medium supports 150 connections by default. You're safe.

**Spec:** db.t3.medium, Multi-AZ, 20 GB gp3, 7-day backup retention

---

### 6. S3 — File Storage

**What it stores:**

| Content | S3 path pattern | Presigned URL expiry |
|---------|----------------|---------------------|
| Uploaded PDFs | `uploads/{tenant_id}/{filename}` | 1 hour |
| Page preview images | `uploads/{lesson_prefix}/page_{N}.png` | 7 days |
| DALL-E generated images | `uploads/{lesson_prefix}/page_{N}.png` | 7 days |
| TTS audio files | `uploads/{lesson_prefix}/page_{N}_read_aloud.mp3` | 7 days |
| Reports (Excel/Word/etc) | `reports/{tenant_id}/{lesson_name}_{report_id}.{ext}` | 24 hours |

**Why S3, not EFS/EBS:**
- PDFs are write-once, read-many — perfect for object storage
- Presigned URLs let the frontend download directly from S3 (no backend proxy)
- 99.999999999% durability
- Pay only for what you store

---

### 7. ALB (Application Load Balancer)

**What it does:**
- Routes `/api/v1/*` requests to the 2 API Fargate tasks
- SSL termination (HTTPS → HTTP to containers)
- Health checks on `/api/v1/health` endpoint
- Distributes load across 2 AZs

**Why ALB, not NLB:**
Your API uses HTTP/HTTPS with path-based routing. ALB is purpose-built for this. NLB is for TCP/UDP traffic.

---

### 8. CloudFront — CDN for Frontend

**What it does:**
- Serves your React SPA (built `dist/` folder) from S3
- Caches static assets at edge locations
- SPA routing: 403/404 → `/index.html` with 200 status

**Why needed:**
Without CloudFront, every page load hits S3 directly from the user's browser. CloudFront caches at 400+ edge locations = faster load times globally.

---

### 9. CloudWatch — Monitoring

**What to monitor:**

| Metric | Source | Alert threshold |
|--------|--------|----------------|
| Celery queue depth (ai_processing) | Custom metric from Redis LLEN | > 10 tasks waiting |
| API 5xx error rate | ALB metrics | > 1% of requests |
| RDS connections | RDS metrics | > 100 connections |
| RDS CPU | RDS metrics | > 80% sustained |
| ECS task health | ECS metrics | Any task unhealthy |
| Worker task failure rate | Custom metric from Celery | > 10% failure rate |
| Redis memory usage | ElastiCache metrics | > 80% |

---

### 10. Secrets Manager

**What it stores:**
- `OPENAI_API_KEY`
- `JWT_SECRET`
- `DB_PASSWORD`
- `SMTP_PASSWORD`
- `AWS_SECRET_ACCESS_KEY` (for S3 operations within containers)

**Why not env vars in task definition:**
ECS task definitions are visible in the console and API. Anyone with ECS access can read plain-text env vars. Secrets Manager encrypts at rest and injects at runtime.

---

### 11. Route 53 + ACM

**Route 53:** DNS routing — `app.yourdomain.com` → CloudFront, `api.yourdomain.com` → ALB
**ACM:** Free SSL certificates for both domains

---

## Architecture Diagram

```
                         ┌──────────────┐
                         │   Route 53   │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │  CloudFront  │
                         │  + ACM SSL   │
                         └──────┬───────┘
                                │
               ┌────────────────┼────────────────┐
               │                                 │
        ┌──────▼──────┐                   ┌──────▼──────┐
        │ S3 Bucket   │                   │     ALB     │
        │ (React SPA) │                   │  (HTTPS)    │
        └─────────────┘                   └──────┬──────┘
                                                 │
                                    ┌────────────┼────────────┐
                                    │                         │
                              ┌─────▼─────┐             ┌────▼──────┐
                              │ ECS API-1 │             │ ECS API-2 │
                              │ (FastAPI) │             │ (FastAPI) │
                              └─────┬─────┘             └─────┬─────┘
                                    │                         │
                                    └────────────┬────────────┘
                                                 │
                         ┌───────────────────────┼───────────────────────┐
                         │                       │                       │
                  ┌──────▼──────┐         ┌──────▼──────┐         ┌─────▼──────┐
                  │ ElastiCache │         │    RDS      │         │  S3 Bucket │
                  │  (Redis)    │         │ PostgreSQL  │         │  (Files)   │
                  │  Broker +   │         │  Multi-AZ   │         │            │
                  │  Results    │         │             │         │            │
                  └──────┬──────┘         └──────▲──────┘         └─────▲──────┘
                         │                       │                      │
            ┌────────────┼────────────┐          │                      │
            │            │            │          │                      │
     ┌──────▼─────┐ ┌───▼────┐ ┌────▼────┐     │                      │
     │  AI Worker │ │ Report │ │ Default │     │                      │
     │  1-2 tasks │ │ Worker │ │ Worker  │     │                      │
     │ queue:     │ │ queue: │ │ queue:  │     │                      │
     │ ai_process │ │reports │ │default  │─────┘──────────────────────┘
     │            │ │        │ │(SLA,    │
     │ GPT+DALLE  │ │ Excel  │ │ notify, │
     │ +TTS+S3    │ │ Word   │ │ webhook)│
     └────────────┘ │ HTML   │ └─────────┘
                    └────────┘
                         │
                  ┌──────▼──────┐
                  │ Celery Beat │
                  │ (Scheduler) │
                  │ hourly SLA  │
                  └─────────────┘
```

---
