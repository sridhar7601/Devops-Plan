# LMS AI Platform — Production AWS Architecture

## Application Overview

A multi-tenant Learning Management System with AI-powered content generation. The platform enables organizations to upload educational PDFs and automatically generate structured lesson content — summaries, translations, questions, diagrams, and audio — using OpenAI GPT-4o, DALL-E 3, and TTS models.

## Current Technology Stack

| Layer | Technology |
|-------|-----------|
| API Server | FastAPI (Python 3.12) + Uvicorn |
| Async Task Queue | Celery with Redis broker |
| Task Queues | `ai_processing`, `reports`, `default` (SLA, notifications, webhooks) |
| Periodic Scheduler | Celery Beat — hourly SLA compliance sweep |
| AI Services | OpenAI GPT-4o (content), DALL-E 3 (images), TTS-1 (audio) |
| File Storage | AWS S3 — PDFs, AI-generated images, audio, exported reports |
| Database | PostgreSQL 17 — SQLAlchemy ORM with connection pooling |
| Authentication | JWT tokens in httponly cookies, role-based access control (6 roles) |
| Multi-tenancy | Tenant-scoped data isolation on all queries and S3 paths |

---

## AWS Services — Why Each One, What It Does

### 1. ECS Fargate (API) — Auto-Scalable

**What it runs:** FastAPI application server (`uvicorn src.main:app`)

**Why Fargate, not EC2:**
- No server patching, no SSH access, no OS management
- Pay per task (vCPU + memory), not per idle instance
- Health checks auto-restart crashed containers
- Seamless horizontal scaling without capacity planning

**API endpoint response profile:**

| Endpoint category | What it does | Avg response time |
|------------------|-------------|-------------------|
| Auth (login, refresh, me) | JWT decode, DB lookup | 50–150ms |
| Lesson list/detail | DB query with eager loading | 20–200ms |
| Workflow transitions | DB update + audit log | 50–150ms |
| Page review/score | DB update | 20–30ms |
| Upload PDF | Validate + extract text + queue Celery task | 5–30s (returns early) |
| Dashboard stats | Aggregate COUNT queries | 50–200ms |
| Notifications | Simple CRUD | 10–50ms |

Most API calls complete in under 200ms. PDF upload is the slowest but returns immediately after queuing the AI processing task to Celery.

**Scaling:** Minimum 2 tasks (high availability across 2 AZs), scales out based on configurable metrics (request count, CPU utilization, or custom CloudWatch metrics). Scaling policy to be defined based on observed production load patterns.

**Spec:** 1 vCPU, 2 GB memory per task | Min: 2, Max: configurable

---

### 2. ECS Fargate (Celery Workers) — Background Processing Engine

This is where the heavy compute happens. Three separate worker groups, each mapped to a dedicated Celery queue for independent scaling and isolation:

#### Worker Group A: `ai_processing` queue — Auto-Scalable

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

Each task runs 2 concurrent Celery workers (`-c 2`). Each worker holds a DB session for the full lesson duration (15–50 min). The bottleneck is OpenAI API latency, not CPU — but scaling out more tasks increases parallel lesson throughput when multiple uploads are queued.

**Scaling:** Minimum 1 task, scales out based on queue depth (configurable). Each additional task doubles concurrent lesson processing capacity.

**Spec:** 1 vCPU, 3 GB memory per task (images + audio buffered in memory) | Min: 1, Max: configurable

#### Worker Group B: `reports` queue

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

#### Worker Group C: `default` queue

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

### 3. Celery Beat (Scheduler) — Singleton

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
    ┌──────────────────────────────────────────────────────────────────────────┐
    │                            USERS / BROWSERS                             │
    └────────────────────────┬─────────────────────────┬───────────────────────┘
                             │                         │
                     Frontend Requests           API Requests
                             │                         │
                    ┌────────▼────────┐       ┌────────▼────────┐
                    │   CloudFront    │       │   Route 53      │
                    │   (CDN)         │       │   (DNS)         │
                    └────────┬────────┘       └────────┬────────┘
                             │                         │
                    ┌────────▼────────┐       ┌────────▼────────┐
                    │   S3 Bucket     │       │      ALB        │
                    │  (React SPA)    │       │  (HTTPS + SSL)  │
                    │  Static Assets  │       │  Health Checks  │
                    └─────────────────┘       └────────┬────────┘
                                                       │
                                          ┌────────────┼────────────┐
                                          │            │            │
                                    ┌─────▼─────┐ ┌───▼─────┐ ┌───▼─────┐
                                    │  ECS API  │ │ ECS API │ │ ECS API │
                                    │  Task 1   │ │ Task 2  │ │ Task N  │
                                    │ (FastAPI) │ │(FastAPI)│ │(FastAPI)│
                                    └─────┬─────┘ └────┬────┘ └────┬────┘
                                          │            │            │
                                          └────────────┼────────────┘
                                                       │ Auto-Scalable
                                                       │ (min: 2)
                                          ┌────────────┴────────────┐
                                          │                         │
                                 ┌────────▼────────┐       ┌───────▼────────┐
                                 │  ElastiCache     │       │  Secrets       │
                                 │  (Redis)         │       │  Manager       │
                                 │  Celery Broker   │       │  (API Keys,    │
                                 │  + Result Store  │       │   DB Creds,    │
                                 │  + Rate Limits   │       │   JWT Secret)  │
                                 └────────┬─────────┘       └────────────────┘
                                          │
                         ┌────────────────┼────────────────┐
                         │                │                │
                  ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
                  │  AI Worker  │  │   Report    │  │  Default   │
                  │  Tasks      │  │   Worker    │  │  Worker    │
                  │ ─────────── │  │ ─────────── │  │ ─────────  │
                  │ GPT-4o      │  │ Excel/Word  │  │ SLA Sweep  │
                  │ DALL-E 3    │  │ HTML/CSV    │  │ Notify     │
                  │ TTS-1       │  │ Export      │  │ Webhooks   │
                  │ ─────────── │  └──────┬──────┘  └─────┬──────┘
                  │Auto-Scalable│         │               │
                  │ (min: 1)    │         │               │
                  └──────┬──────┘         │               │
                         │                │               │
                         └────────────────┼───────────────┘
                                          │
                         ┌────────────────┼────────────────┐
                         │                │                │
                  ┌──────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
                  │    RDS      │  │  S3 Bucket  │  │  Celery    │
                  │ PostgreSQL  │  │  (Files)    │  │  Beat      │
                  │  17         │  │ ─────────── │  │ ─────────  │
                  │ ─────────── │  │ PDFs        │  │ Hourly SLA │
                  │ Multi-AZ    │  │ Images      │  │ Sweep      │
                  │ Auto Backup │  │ Audio       │  │ (Singleton)│
                  │ 150 max     │  │ Reports     │  │            │
                  │ connections │  │             │  │            │
                  └─────────────┘  └─────────────┘  └────────────┘

    ┌──────────────────────────────────────────────────────────────────────────┐
    │                         MONITORING (CloudWatch)                          │
    │  API 5xx rate │ Queue depth │ RDS connections │ Redis memory │ Task health│
    └──────────────────────────────────────────────────────────────────────────┘
```

**Key design decisions:**
- CloudFront serves **only** the frontend SPA from S3 — API traffic goes directly through ALB
- ElastiCache Redis sits between the API layer and workers as the central message broker
- API tasks and AI Worker tasks are independently auto-scalable (scaling metrics to be configured based on production load patterns)
- All compute runs in private subnets; only ALB and CloudFront are internet-facing
- Secrets Manager injects credentials at container startup — no secrets in environment variables or code

---

## Cost Breakdown

### Base Cost (Minimum Running Configuration)

| Service | Spec | Monthly Cost |
|---------|------|-------------|
| ECS Fargate — API (min 2 tasks) | 1 vCPU, 2 GB each | ~$60 |
| ECS Fargate — AI Worker (min 1 task) | 1 vCPU, 3 GB | ~$40 |
| ECS Fargate — Report Worker (1 task) | 0.5 vCPU, 2 GB | ~$22 |
| ECS Fargate — Default Worker (1 task) | 0.25 vCPU, 0.5 GB | ~$8 |
| ECS Fargate — Beat (1 task) | 0.25 vCPU, 0.5 GB | ~$8 |
| ElastiCache Redis | cache.t3.micro, Multi-AZ | ~$25 |
| RDS PostgreSQL | db.t3.medium, Multi-AZ, 20 GB | ~$60 |
| ALB | 1 load balancer | ~$20 |
| S3 | 100 GB storage | ~$3 |
| CloudFront | 100 GB transfer | ~$10 |
| NAT Gateway | 1 AZ | ~$32 |
| Secrets Manager | 5 secrets | ~$3 |
| CloudWatch | Logs + alarms | ~$10 |
| Route 53 + ACM | 1 hosted zone + SSL | ~$2 |
| Data transfer | ~100 GB/month | ~$9 |
| **Base Total** | | **~$312/month** |

### Scaling Cost (Additional Tasks When Active)

| Scaled Service | Per Additional Task | Trigger |
|---------------|-------------------|---------|
| API Task | +~$30/month per task | High request volume |
| AI Worker Task | +~$40/month per task | Queue depth > threshold |

Auto-scaling adds cost only while demand is high. Tasks scale back to minimum during low-traffic periods, keeping base cost stable.


---

## ECS Task Summary

| Task | Min Count | Max Count | vCPU | Memory | What it runs | Scaling |
|------|-----------|-----------|------|--------|-------------|---------|
| API | 2 | Configurable | 1 | 2 GB | `uvicorn src.main:app` | Auto-scale on request count / CPU |
| AI Worker | 1 | Configurable | 1 | 3 GB | `celery worker -Q ai_processing -c 2` | Auto-scale on queue depth |
| Report Worker | 1 | — | 0.5 | 2 GB | `celery worker -Q reports -c 2` | Fixed |
| Default Worker | 1 | — | 0.25 | 0.5 GB | `celery worker -Q default -c 4` | Fixed |
| Beat | 1 | — | 0.25 | 0.5 GB | `celery beat` | Fixed (singleton) |
| **Base Total** | **6 tasks** | | **4 vCPU** | **10 GB** | | |

**Scaling logic:**
- **API tasks** — auto-scale when sustained request load exceeds capacity of current tasks. Minimum 2 ensures high availability across AZs even at idle.
- **AI Worker tasks** — auto-scale when `ai_processing` queue depth exceeds a threshold (e.g., >5 pending tasks). Each additional worker doubles concurrent lesson processing. Scales back when queue drains.
- **Report, Default, Beat** — fixed at 1 task each. Report and default workloads are lightweight and infrequent. Beat must remain a singleton to prevent duplicate scheduled tasks.

Specific scaling metrics and thresholds will be configured after observing production load patterns.

---

## Implementation Roadmap

| Phase | Scope | Deliverables | Dependency |
|-------|-------|-------------|-----------|
| **1. Containerization** | Docker setup for local development | Backend Dockerfile, docker-compose.yml (API + 3 workers + Beat + Redis + Postgres), .dockerignore | None |
| **2. Infrastructure as Code** | Terraform modules for all AWS resources | VPC, ECS cluster, ALB, ElastiCache, RDS (import existing), S3 + CloudFront, ECR, Secrets Manager, CloudWatch, IAM roles | Phase 1 |
| **3. CI/CD Pipeline** | Automated build and deployment | Backend deploy script (build → ECR → ECS update), frontend deploy script (build → S3 → CloudFront invalidation), migration runner | Phase 2 |
| **4. Production Launch** | Go-live on AWS | DNS cutover, SSL certificates, secret rotation, smoke testing, monitoring validation | Phase 3 |

---

