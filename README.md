# video-translation-dubbing

> **ONE REEL. EVERY AUDIENCE.**  
> AI-powered regional video localization platform for short-form video creators. Dub 9:16 vertical Reels into 12 Indian languages while preserving the creator's natural voice, cadence, and vocal identity.

[![Live Website](https://img.shields.io/badge/Live-gowider.in-FF441F?style=for-the-badge&logo=google-chrome&logoColor=white)](https://gowider.in)
[![Next.js 15](https://img.shields.io/badge/Next.js_15-000000?style=for-the-badge&logo=next.js&logoColor=white)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript_5-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![BullMQ & Redis](https://img.shields.io/badge/BullMQ_&_Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://bullmq.io/)
[![Cloudflare R2](https://img.shields.io/badge/Cloudflare_R2-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)](https://www.cloudflare.com/developer-platform/r2/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![CI/CD Pipeline](https://github.com/mahe-gi/gowider/actions/workflows/ci-cd.yml/badge.svg)](https://github.com/mahe-gi/gowider/actions/workflows/ci-cd.yml)

---

## Table of Contents

- [1. System Architecture](#1-system-architecture)
- [2. Key Features](#2-key-features)
- [3. Technology Stack](#3-technology-stack)
- [4. Supported Languages](#4-supported-languages)
- [5. Core Engine & State Machine](#5-core-engine--state-machine)
- [6. Local Development Setup](#6-local-development-setup)
- [7. Environment Variables](#7-environment-variables)
- [8. Production Deployment Guide](#8-production-deployment-guide)
- [9. Automated CI/CD Pipeline](#9-automated-cicd-pipeline)
- [10. Testing & Quality Verification](#10-testing--quality-verification)
- [11. Security & Compliance](#11-security--compliance)

---

## 1. System Architecture

GoWider is architected around a **decoupled, event-driven, and disposable VPS topology**:
- **Next.js Web Server (`gowider-web`)**: Handles client rendering, Google OAuth authentication, pre-signed upload URL generation, and checkout flows.
- **Worker Daemon (`gowider-worker`)**: Orchestrates AI dubbing jobs, asynchronous polling, video/subtitle archiving, and payment webhook reconciliation.
- **Redis Queue (`gowider-redis`)**: High-performance job broker managing BullMQ delayed polling and exponential backoff retry queues.
- **Managed PostgreSQL**: Authoritative source of truth for user profiles, project metadata, generation runs, and credit ledger.
- **Cloudflare R2**: High-throughput S3-compatible private object storage for source and dubbed media.

```text
┌────────────────────────────────────────────────────────┐
│                   Creator Browser                      │
│  - Google OAuth Session (Auth.js v5)                   │
│  - Direct-to-R2 Presigned Chunked Upload               │
│  - Studio (Interactive 9:16 Canvas & Language Chips)   │
│  - Multi-Language Tabbed Player & Subtitle Viewer      │
│  - Razorpay Checkout Modal                             │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│                   Next.js 15 App                       │
│                                                        │
│  API Endpoints:                                        │
│  - POST /api/uploads/presign (Direct Storage PUT)      │
│  - POST /api/uploads/complete (Fail-closed metadata)   │
│  - POST /api/projects        (Create draft project)    │
│  - POST /api/projects/:id/configure (Save language cfg)│
│  - POST /api/projects/:id/generate (Reserve & Enqueue) │
│  - GET  /api/projects/:id    (Authoritative status)    │
│  - POST /api/payments/order  (Create Razorpay order)   │
│  - POST /api/payments/verify (HMAC SHA-256 validation) │
│  - POST /api/webhooks/razorpay (Deduplicated Webhooks) │
└───────────────┬────────────────────────┬───────────────┘
                │                        │
                ▼                        ▼
       ┌────────────────┐       ┌─────────────────┐
       │   PostgreSQL   │       │ Cloudflare R2   │
       │ (State Truth)  │       │ Object Storage  │
       │ - users        │       │                 │
       │ - projects     │       │ - sources/...   │
       │ - gen_runs     │       │ - outputs/...   │
       │ - outputs      │       └────────┬────────┘
       │ - wallets      │                │
       │ - transactions │                │ Stream to Provider
       └────────┬───────┘                ▼
                │ Enqueue job   ┌───────────────────┐
                ▼               │ Sarvam AI Dubbing │
       ┌────────────────┐       │                   │
       │     BullMQ     │       │ - Speech-to-Text  │
       │        ↓       │       │ - Translation LLM │
       │     Redis 7    │       │ - Voice Synthesis │
       └────────┬───────┘       └───────────────────┘
                │ Dequeue jobs           ▲
                ▼                        │ Orchestration
       ┌─────────────────────────────────┴──┐
       │         GoWider Worker             │
       │  - workers/generation-worker.ts    │
       │  - workers/payment-worker.ts       │
       │  - workers/maintenance-worker.ts   │
       └────────────────────────────────────┘
```

---

## 2. Key Features

- **Voice-Preserving Neural Dubbing**: Clones the original creator's voice texture, tone, and pacing into regional target languages.
- **Direct-to-Storage Upload Pipeline**: Client-side video uploads stream directly to Cloudflare R2 via presigned PUT URLs, bypassing Next.js API server memory limits.
- **Non-Blocking Delayed Polling**: Background workers schedule delayed checks in Redis rather than holding synchronous connections during AI processing.
- **Atomic Credit Ledger**: Double-entry ledger prevents double-spending by atomically reserving credits at dispatch and releasing credits upon failure.
- **Zero-Loss Payment Reconciliation**: Razorpay orders are verified via HMAC SHA-256 client callbacks and backed by idempotent, deduplicated server webhooks.
- **Production-Hardened Security**: Strict Content Security Policy (CSP), cryptographically signed downloads, and automated orphan media cleanup.

---

## 3. Technology Stack

| Layer | Technology | Purpose |
| :--- | :--- | :--- |
| **Frontend** | Next.js 15, React 19, Tailwind CSS | App Router, Server Actions, Lucide Icons, Responsive 9:16 Canvas |
| **Backend API** | Next.js Server Components & Route Handlers | RESTful APIs with Zod schema validation & fail-closed error handling |
| **Database** | PostgreSQL + Drizzle ORM | Normalized relational schema with strict indexes and foreign keys |
| **Queue & Cache** | Redis 7 + BullMQ | Asynchronous job queues, concurrency management, and worker scheduling |
| **Worker Engine** | TypeScript Daemons (`workers/index.ts`) | Standalone orchestration service running on dedicated worker processes |
| **AI Provider** | Sarvam AI | Regional Indian speech recognition, translation LLMs, and TTS dubbing |
| **Media Storage** | Cloudflare R2 (S3-compatible) | S3 SDK (`@aws-sdk/client-s3`), presigned uploads (600s), signed downloads (900s) |
| **Authentication**| NextAuth.js (Auth.js v5) | Google OAuth 2.0 with JWT sessions and user auto-provisioning |
| **Payments** | Razorpay Gateway | INR checkout packs, HMAC-SHA256 signature verification, webhooks |
| **DevOps & Server**| Docker Compose, Caddy 2, AWS EC2 | Multi-stage Docker builds, auto-SSL reverse proxy, systemd integration |

---

## 4. Supported Languages

GoWider supports localization across **12 Indic languages**:

| Code | Language | Script Native |
| :--- | :--- | :--- |
| `en-IN` | English (India) | English |
| `hi-IN` | Hindi | हिन्दी |
| `te-IN` | Telugu | తెలుగు |
| `ta-IN` | Tamil | தமிழ் |
| `kn-IN` | Kannada | ಕನ್ನಡ |
| `ml-IN` | Malayalam | മലയാളം |
| `bn-IN` | Bengali | বাংলা |
| `mr-IN` | Marathi | मराठी |
| `gu-IN` | Gujarati | ગુજરાતી |
| `pa-IN` | Punjabi | ਪੰਜਾਬੀ |
| `od-IN` | Odia | ଓଡ଼ିଆ |
| `as-IN` | Assamese | অসমীয়া |

---

## 5. Core Engine & State Machine

Localization jobs progress through a state machine that isolates external AI provider latency:

```text
[ User clicks "Generate" ]
             │
             ▼
[ POST /api/projects/:id/generate ]
  - Verify credit balance >= required cost
  - Atomically reserve credits in PostgreSQL
  - Enqueue job: generation:start
             │
             ▼
[ generation:start ]
  - Initialize Sarvam AI dubbing job
  - Stream source video from R2 to Sarvam
  - Store external provider job ID
  - Enqueue delayed job: generation:poll-live (15s delay)
             │
             ▼ (Worker thread is freed immediately)
[ generation:poll-live ]
  - Poll Sarvam job status
  - If processing ──► Re-enqueue generation:poll-live (15s delay)
  - If failed ──────► Release reserved credits & mark run "failed"
  - If completed ───► Enqueue generation:poll-export (5s delay)
             │
             ▼
[ generation:poll-export ]
  - Verify all language audio/video exports are ready
  - Download outputs and archive MP4 & SRT to private R2 storage
  - Settle wallet (charge only successful language tracks)
  - Update project status to "completed"
```

---

## 6. Local Development Setup

### Prerequisites
- Node.js $\ge$ 20.0.0
- Docker Desktop or Docker Engine
- npm $\ge$ 10.0.0

### Step 1: Clone Repository
```bash
git clone https://github.com/mahe-gi/gowider.git
cd gowider
```

### Step 2: Install Dependencies
```bash
npm install
```

### Step 3: Start Local Database & Redis
```bash
docker compose up -d postgres redis
```

### Step 4: Configure Environment File
Create a `.env.local` file with local development credentials:
```bash
cp .env.example .env.local
```

### Step 5: Push Database Schema
```bash
npm run db:push
```

### Step 6: Start Web Server & Worker Daemon
Open two terminal windows:

**Terminal 1 (Next.js Web Server):**
```bash
npm run dev
```

**Terminal 2 (Background Worker Daemon):**
```bash
npm run worker:dev
```

Visit [http://localhost:3000](http://localhost:3000) in your browser.

---

## 7. Environment Variables

| Variable | Required | Purpose |
| :--- | :---: | :--- |
| `DATABASE_URL` | **Yes** | PostgreSQL connection string (`sslmode=require` for Neon/RDS) |
| `REDIS_URL` | **Yes** | Redis connection URI (e.g. `redis://redis:6379`) |
| `AUTH_SECRET` | **Yes** | 32+ character random secret for NextAuth session encryption |
| `GOOGLE_CLIENT_ID` | **Yes** | Google Cloud Console OAuth 2.0 Web Client ID |
| `GOOGLE_CLIENT_SECRET` | **Yes** | Google Cloud Console OAuth 2.0 Client Secret |
| `STORAGE_DRIVER` | **Yes** | `r2` in production, `local` for offline development |
| `R2_ACCOUNT_ID` | When `r2` | Cloudflare Account ID |
| `R2_ACCESS_KEY_ID` | When `r2` | Cloudflare R2 Access Key ID |
| `R2_SECRET_ACCESS_KEY`| When `r2` | Cloudflare R2 Secret Access Key |
| `R2_BUCKET_NAME` | When `r2` | Cloudflare R2 Bucket Name (e.g. `gowider`) |
| `SARVAM_API_KEY` | **Yes** | Sarvam AI API Key for speech recognition and TTS dubbing |
| `RAZORPAY_KEY_ID` | **Yes** | Razorpay Key ID (`rzp_test_...` or `rzp_live_...`) |
| `RAZORPAY_KEY_SECRET` | **Yes** | Razorpay Key Secret for HMAC verification |
| `RAZORPAY_WEBHOOK_SECRET` | **Yes** | Razorpay Webhook Secret for callback validation |
| `NEXT_PUBLIC_APP_URL` | No | Base application URL (default: `https://gowider.in`) |

---

## 8. Production Deployment Guide

GoWider is deployed using **Docker Compose** on an AWS EC2 instance (`t3.small` / Amazon Linux 2023 or Ubuntu) with **Caddy** as the reverse proxy for automatic SSL management.

### Deployment Commands on Server:

```bash
# 1. Clone repository
git clone https://github.com/mahe-gi/gowider.git /home/ec2-user/gowider
cd /home/ec2-user/gowider

# 2. Configure production .env file (with chmod 600)
nano .env

# 3. Apply PostgreSQL schema migrations & columns
npm run db:sync

# 4. Build and start production containers
docker compose -f docker-compose.prod.yml up -d --build

# 5. Check container health
docker compose -f docker-compose.prod.yml ps
```

### Caddy Reverse Proxy Configuration (`/etc/caddy/Caddyfile`):
```caddy
gowider.in, www.gowider.in {
    reverse_proxy 127.0.0.1:3000
}
```

---

---

## 9. Automated CI/CD Pipeline

GoWider uses **GitHub Actions** for continuous integration and continuous deployment (`.github/workflows/ci-cd.yml`).

```text
[ Git Push / Pull Request ]
            │
            ▼
┌────────────────────────────────────────────────────────┐
│  Stage 1: Continuous Integration (CI)                  │
│  - TypeScript Typecheck (tsc --noEmit)                 │
│  - ESLint Static Code Analysis (next lint)             │
│  - Unit Test Suite Execution (vitest run tests/unit)   │
│  - Next.js Production Build Validation (next build)    │
└───────────────────────────┬────────────────────────────┘
                            │
                     Passed on main?
                            │
               Yes ─────────┴───────── No (Halt)
               │
               ▼
┌────────────────────────────────────────────────────────┐
│  Stage 2: Continuous Deployment (CD via SSH)           │
│  - Connect to AWS EC2 via SSH (appleboy/ssh-action)    │
│  - Pull latest commits (git fetch && git reset --hard) │
│  - Layer-cached Docker image builds                    │
│  - Run database migrations (npm run db:migrate)        │
│  - Hot-swap containers with zero-downtime transition   │
│  - Prune dangling Docker images (docker image prune)   │
│  - Healthcheck verification (/api/health HTTP 200 OK)  │
└────────────────────────────────────────────────────────┘
```

### Required GitHub Repository Secrets

Configure these secrets under **Repository Settings $\rightarrow$ Secrets and variables $\rightarrow$ Actions**:

| Secret Name | Value | Purpose |
| :--- | :--- | :--- |
| `EC2_HOST` | `52.66.50.152` | Public IP or DNS of the AWS EC2 instance |
| `EC2_USER` | `ec2-user` | SSH user on Amazon Linux |
| `EC2_SSH_KEY` | *(Contents of `gowider.pem`)* | Private SSH key for automated server access |
| `EC2_DEPLOY_PATH` | `/home/ec2-user/gowider` | *(Optional)* Target directory on the EC2 server |

---

## 10. Testing & Quality Verification

```bash
# Run unit test suite (13 test files / 52 tests)
npm run test:unit

# Run full test suite
npm test

# Static code linting
npm run lint

# TypeScript type safety check
npm run typecheck

# Next.js production build compilation
npm run build

# Seed dev credits for local testing
npm run dev:seed-credits
```

---

## 11. Security & Compliance

- **Voice Rights Consent**: Explicit legal warranty checkbox recorded with timestamps prior to localization.
- **Content Security Policy (CSP)**: Strict headers restrict script execution, framing, and connect origins to approved hosts (`accounts.google.com`, `checkout.razorpay.com`, `*.r2.cloudflarestorage.com`).
- **Cryptographic Signatures**: Razorpay webhooks and payment callbacks are validated using `crypto.createHmac("sha256", secret)`.
- **Ephemeral Access**: Video downloads use time-limited presigned URLs (900 seconds) preventing unauthorized hotlinking.

---

## License

Copyright © 2026 GoWider. All rights reserved.
