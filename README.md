# Sulah — AI-Powered Mediation Platform
### National Law University Shimla | Legal Innovation and Incubation Cell (LIIC)
### Built under India's Mediation Act 2023

**Live app:** [nlu-mediation-platform.vercel.app](https://nlu-mediation-platform.vercel.app/)

---

## Table of Contents

1. [What This Is](#1-what-this-is)
2. [System Architecture](#2-system-architecture)
3. [Folder Structure](#3-folder-structure)
4. [Tech Stack](#4-tech-stack)
5. [AI Pipeline](#5-ai-pipeline)
6. [Case Flow and State Machine](#6-case-flow-and-state-machine)
7. [Database Schema](#7-database-schema)
8. [Local Development Setup](#8-local-development-setup)
9. [Deployment](#9-deployment)
10. [Environment Variables](#10-environment-variables)
11. [Known Limitations and Future Work](#11-known-limitations-and-future-work)
12. [Original Intern Team](#12-original-intern-team)

---

## 1. What This Is

Sulah is a full-stack web application that digitises the formal mediation process under India's Mediation Act 2023.

It allows citizens to file dispute applications online, receive AI-generated conflict analysis, and reach legally documented settlements through a structured mediator-supervised workflow.

**Core principle:**

> AI advises and drafts. The mediator decides and publishes. Parties make final acceptance decisions. AI never makes a legally consequential decision alone.

### User Roles

| Role | Description |
|---|---|
| `mediator` | Manages cases, reviews AI analysis, sends questionnaires, creates and publishes proposals, finalises settlements |
| `party_user` | Shared account role for both the requesting party and the against party |

Both disputing parties register with the same `party_user` role. Their case-specific identity — `requesting_party` or `against_party` — is not stored on the user record. It is determined dynamically per case via the `case_invitations` table.

Use `GET /api/v1/cases/{case_id}/my-role` to resolve which party a logged-in user is in a given case.

---

## 2. System Architecture

```
React Frontend (Vercel)
        │
        │ HTTPS + JWT
        ▼
FastAPI Backend (Render — Web Service)
        │                    │
        │                    │ .delay()
        ▼                    ▼
  Supabase              Redis (Upstash)
  PostgreSQL                 │
  (13 tables)                ▼
                    Celery Worker (Render — Web Service)
                             │
                             ▼
                    Groq LLM API
                    (8 AI sub-systems)
```

**Important deployment note:** Render does not offer a free tier for Background Workers. The Celery worker is deployed as a Web Service on Render via `worker_web.py`, which exposes a `/` health endpoint. UptimeRobot pings it every 5–10 minutes to prevent Render's free tier from spinning the service down after 15 minutes of inactivity.

---

## 3. Folder Structure

```
nlu-mediation-platform/
├── ai/                              # AI pipeline (Python)
│   ├── subsystems/
│   │   ├── subsystem_a.py           # Conflict extraction (large model)
│   │   ├── subsystem_b.py           # Neutral summary (large model)
│   │   ├── subsystem_c.py           # Questionnaire generation (small model)
│   │   ├── subsystem_d.py           # BATNA/WATNA analysis (large model)
│   │   ├── subsystem_e.py           # Bias removal (small model)
│   │   ├── subsystem_f.py           # Tone analysis (small model)
│   │   ├── subsystem_g.py           # Mediatability score (small model)
│   │   └── subsystem_h.py           # Proposal revision (large model)
│   ├── utils/
│   │   └── ai_client.py             # Shared LLM client with retry logic
│   ├── schemas.py                   # All Pydantic models for AI output
│   ├── pipeline_burst1.py           # Canonical Burst 1 reference implementation
│   ├── pipeline_burst2.py           # Burst 2 reference implementation
│   └── proposal_draft.py            # Initial proposal generation
│
├── frontend/                        # React + Vite + TailwindCSS
│   ├── src/
│   │   ├── pages/
│   │   │   ├── auth/                # Login, register, invitation accept
│   │   │   ├── mediator/            # Mediator UI screens
│   │   │   └── party/               # Party UI screens
│   │   ├── routes/                  # AppRoutes, ProtectedRoute, RoleBasedRoute
│   │   ├── api/                     # Axios API modules
│   │   ├── services/                # Auth and domain services
│   │   └── components/              # Shared UI components
│   ├── index.html
│   └── vite.config.js
│
├── nlu-backend/                     # FastAPI backend
│   ├── app/
│   │   ├── api/v1/
│   │   │   ├── router.py            # Route aggregator
│   │   │   └── routes/              # auth, cases, invitations, documents,
│   │   │                            # questionnaires, proposals, settlement
│   │   ├── core/
│   │   │   ├── state_machine.py     # Case state transitions (golden rule)
│   │   │   ├── database.py          # Supabase client
│   │   │   ├── config.py            # Settings from .env
│   │   │   ├── security.py          # JWT helpers
│   │   │   └── dependencies.py      # FastAPI auth dependencies
│   │   ├── models/                  # Pydantic request/response models
│   │   ├── services/
│   │   │   └── pdf_generator.py     # ReportLab settlement PDF
│   │   └── worker/
│   │       └── celery_app.py        # Alternate/legacy Celery app (dev/smoke tests)
│   ├── tasks.py                     # Primary Celery task definitions (canonical — used in production)
│   ├── worker_web.py                # Celery + FastAPI wrapper for Render
│   ├── main.py                      # FastAPI entry point (top-level, not inside app/)
│   ├── requirements.txt
│   ├── .env.example                 # Backend environment template
│   └── frontend.env.example         # Frontend environment template
│
├── tests/                           # Test scenarios S-01 to S-12
│   ├── scenarios/
│   └── test_*.py
│
├── docs/                            # Architecture and API docs
│   ├── api-contract.md
│   ├── database.md
│   └── state-machine.md
│
└── README.md
```

> `main.py` lives at the top level of `nlu-backend/`, not inside `app/`. The correct entrypoint command is `uvicorn main:app`, not `uvicorn app.main:app`.

---

## 4. Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 8, TailwindCSS 3, React Router 7, Axios |
| Backend | FastAPI, Python 3.11, Uvicorn |
| Database | PostgreSQL via Supabase |
| AI Models | Groq — `openai/gpt-oss-120b` (large), `openai/gpt-oss-20b` (small) |
| Background Jobs | Celery 5 + Redis (Upstash) |
| PDF Generation | ReportLab |
| Authentication | JWT (HS256) |
| Storage | Supabase Storage (`case-documents` bucket) |
| Deployment | Render (backend + Celery worker), Vercel (frontend) |

---

## 5. AI Pipeline

The platform has an **8-sub-system AI pipeline** split across two asynchronous Celery bursts.

### Burst 1 — Triggered when both parties submit statements

Canonical order (Final Flow — **do not reorder**):

```
Step 1: Sub-system F  — Tone analysis on RAW statements
Step 2: Sub-system E  — Bias removal on RAW statements → cleaned text
Step 3: Sub-system A  — Conflict extraction on CLEANED text  ← CRITICAL
Step 4: Sub-system B  — Neutral summary from conflict JSON
Step 5: Sub-system G  — Mediatability score (Python scoring + LLM text)
```

**Why this order matters:**

- **F before E:** Tone analysis must capture true emotional signal from raw text. Once E cleans the statements, that signal is lost.
- **E before A (critical):** Bias removal must run before conflict extraction. If emotional or loaded language reaches A, extracted claims are biased — and everything downstream (B, G, C, D) builds on A's output.
- **A before B and G:** Neutral summary and mediatability both consume A's structured JSON output.

**Sub-system A is the only critical step.** If A fails, the pipeline stops and the case transitions to `PROCESSING_FAILED`. All other sub-system failures degrade gracefully with warning badges.

The authoritative reference implementation is `ai/pipeline_burst1.py`. Celery invokes this pipeline via `process_burst_1` in `nlu-backend/tasks.py`.

### Burst 2 — Triggered when both parties complete the questionnaire

```
Sub-system D — BATNA/WATNA analysis
Input:  ConflictExtraction (Burst 1) + both parties' questionnaire answers
Output: Negotiation position per party, settlement zone
```

### Proposal Generation and Revision

```
proposal_draft.py  — generates initial draft (mediator action)
Sub-system H       — generates revised draft after any party rejection
                      (mediator notes are fed here only — never into Burst 1 or 2)
```

### Sub-System Reference

| Sub-system | File | Model | Input | Critical? |
|---|---|---|---|---|
| A | `subsystem_a.py` | large | Cleaned statements | **Yes** |
| B | `subsystem_b.py` | large | ConflictExtraction | No |
| C | `subsystem_c.py` | small | ConflictExtraction | No |
| D | `subsystem_d.py` | large | ConflictExtraction + questionnaire answers | No |
| E | `subsystem_e.py` | small | Raw statements | No |
| F | `subsystem_f.py` | small | Raw statements | No |
| G | `subsystem_g.py` | small | ConflictExtraction | No |
| H | `subsystem_h.py` | large | Rejected proposal + reasons + BATNA/WATNA | No |

**Model mapping:** large model = `openai/gpt-oss-120b`, small model = `openai/gpt-oss-20b` (both via Groq, defined in `ai/utils/ai_client.py`).

### Key AI Engineering Decisions

**Mediatability score is Python, not LLM:**

LLM scores are non-deterministic — the same input can produce different numbers on different runs, which mediators can't trust. The score is calculated deterministically using weighted Python factors (monetary value, jurisdiction clarity, extraction confidence, undisputed facts count, dispute type suitability). The LLM writes only the justification paragraph; if that text generation fails, a fallback string is used and the score itself is never at risk.

**BATNA invariant enforced in Pydantic, not the prompt:**

BATNA score must always be >= WATNA score (mathematically required — best alternative cannot be worse than worst alternative). Prompts can be ignored by LLMs; Python validators cannot. A `model_validator` on `PartyNegotiationPosition` in `ai/schemas.py` enforces this on every LLM response:

```python
if self.watna_score > self.batna_score:
    avg = (self.batna_score + self.watna_score) // 2
    self.batna_score = avg
    self.watna_score = avg
```

If the model returns BATNA=4, WATNA=7, both are corrected to (5, 5) — averaging is safer than swapping, because neither original value is trustworthy.

**Mediator notes only reach Sub-system H:**

Burst 1 and Burst 2 must stay unbiased — based only on party statements and questionnaire answers. Feeding mediator notes into earlier analysis would skew results toward mediator bias rather than reflecting the parties' own input. Sub-system H is the only place mediator notes are used, during proposal revision, where the mediator is explicitly and appropriately guiding the outcome.

**Retry logic:**

Every LLM call retries up to 3 times via `ai/utils/ai_client.py`. On failure, the error is fed back to the model for self-correction. After 3 failures, non-critical sub-systems degrade gracefully.

---

## 6. Case Flow and State Machine

### Two Case Creation Paths — Mediator Always Creates the Case

Under the Mediation Act 2023, the mediator is a neutral appointee — not an initiator. The mediator formally creates every case in both paths, even when a party files the initial application. `cases.created_by` is always the mediator's `user_id` — this is intentional, not a bug.

**Path 1 — Party initiated:**

```
Party files application (APPLICATION_PENDING)
  → Mediator reviews application
  → Mediator accepts
  → Mediator formally creates case + sends invitations to both parties
```

**Path 2 — Mediator initiated:**

```
Mediator creates case directly
  → Generates invitation links
  → Shares with both parties
```

Both paths merge at `BOTH_INVITED`. From that point forward, Path 1 and Path 2 are identical.

### State Machine — 16 States

The state machine in `nlu-backend/app/core/state_machine.py` is the **only** path to changing case status. No route handler, Celery task, or helper function may set `case.status` directly. All changes go through `transition()`. Invalid transitions return HTTP 409 Conflict. Every transition is written to `audit_logs` (insert-only, never deleted).

**Application request states (Path 1 only — `application_requests` table)**

| State | Description |
|---|---|
| `APPLICATION_PENDING` | Party filed request, waiting for mediator |
| `APPLICATION_REJECTED` | Mediator rejected the request |
| `WITHDRAWN` | Party withdrew before mediator acted |

**Case states (both paths — `cases` table)**

| State | Description |
|---|---|
| `BOTH_INVITED` | Case created, both invitation links generated |
| `FIRST_PARTY_SUBMITTED` | One party submitted intake, waiting for other |
| `BOTH_SUBMITTED` | Both submitted — Burst 1 triggers automatically |
| `BURST_1_PROCESSING` | AI Burst 1 pipeline running |
| `BURST_1_COMPLETE` | AI analysis ready for mediator review |
| `QUESTIONNAIRE_ACTIVE` | Mediator sent questionnaire to both parties |
| `QUESTIONNAIRE_COMPLETE` | Both parties answered — Burst 2 triggers |
| `BURST_2_PROCESSING` | BATNA/WATNA AI pipeline running |
| `BURST_2_COMPLETE` | BATNA/WATNA results ready |
| `PROPOSAL_DRAFT` | Mediator created proposal, not visible to parties |
| `PROPOSAL_PUBLISHED` | Proposal published, parties can respond |
| `MEDIATION_IN_PROGRESS` | At least one party rejected, revision in progress |
| `MEDIATION_COMPLETE` | Both parties accepted a proposal |

**Failure and recovery states (also on `cases` table)**

| State | Description |
|---|---|
| `PROCESSING_FAILED` | AI pipeline failed — mediator can retry |
| `MEDIATION_FAILED` | Max negotiation rounds exhausted with no agreement |

### Valid Transitions

```
APPLICATION_PENDING  → APPLICATION_REJECTED
APPLICATION_PENDING  → WITHDRAWN
APPLICATION_PENDING  → BOTH_INVITED          (mediator accepts application)

BOTH_INVITED         → FIRST_PARTY_SUBMITTED
BOTH_INVITED         → BOTH_SUBMITTED         (simultaneous submission edge case)
FIRST_PARTY_SUBMITTED → BOTH_SUBMITTED

BOTH_SUBMITTED       → BURST_1_PROCESSING
BURST_1_PROCESSING   → BURST_1_COMPLETE
BURST_1_PROCESSING   → PROCESSING_FAILED

BURST_1_COMPLETE     → QUESTIONNAIRE_ACTIVE
QUESTIONNAIRE_ACTIVE → QUESTIONNAIRE_COMPLETE
QUESTIONNAIRE_COMPLETE → BURST_2_PROCESSING
BURST_2_PROCESSING   → BURST_2_COMPLETE
BURST_2_PROCESSING   → PROCESSING_FAILED

BURST_2_COMPLETE     → PROPOSAL_DRAFT
PROPOSAL_DRAFT       → PROPOSAL_PUBLISHED
PROPOSAL_PUBLISHED   → MEDIATION_COMPLETE    (both parties accept)
PROPOSAL_PUBLISHED   → MEDIATION_IN_PROGRESS (any party rejects)
MEDIATION_IN_PROGRESS → PROPOSAL_DRAFT       (mediator creates revised proposal)
MEDIATION_IN_PROGRESS → MEDIATION_COMPLETE   (both accept revised proposal)
MEDIATION_IN_PROGRESS → MEDIATION_FAILED     (max rounds exhausted)
MEDIATION_COMPLETE   → MEDIATION_COMPLETE     (mediator finalise — audit log only)

PROCESSING_FAILED    → BURST_1_PROCESSING     (mediator retry — Burst 1)
PROCESSING_FAILED    → BURST_2_PROCESSING     (mediator retry — Burst 2)
```

---

## 7. Database Schema

13 tables in Supabase PostgreSQL.

| Table | Purpose |
|---|---|
| `users` | Mediators and party users (`mediator` or `party_user` role) |
| `cases` | Core case record — `created_by` is always the mediator |
| `application_requests` | Path 1 party applications |
| `case_invitations` | Invitation tokens per party (`requesting_party` / `against_party`) |
| `submissions` | Party dispute statements (6-step intake wizard) |
| `ai_analysis` | AI sub-system outputs (JSONB), both bursts |
| `questionnaires` | AI-generated questions |
| `questionnaire_responses` | Both parties' answers |
| `proposals` | Settlement proposal drafts and revisions |
| `proposal_responses` | Accept/reject decisions per party |
| `settlement_confirmations` | Typed name + signature image |
| `mediation_reports` | PDF storage path and metadata |
| `audit_logs` | Immutable event log (insert-only — no UPDATE or DELETE) |

AI output schemas evolve frequently. JSONB columns let the schema live in Python (Pydantic models) rather than requiring an `ALTER TABLE` every time a field changes.

**Party isolation:** Parties must never see each other's data. Enforced at the API layer in FastAPI routes — not only via RLS — and verified against `case_invitations` on every party request. A party cannot see the other party's questionnaire answers. Parties only see published proposals, never drafts. BATNA/WATNA numeric scores are internal to the mediator; parties see strength labels only.

**`audit_logs` is insert-only by design:** legal mediation requires tamper-proof records. There is no UPDATE or DELETE policy on this table, and none should ever be added — past events must remain unmodifiable even with database access.

**BATNA/WATNA column:** The `ai_analysis` table has a `batna_watna` JSONB column added after initial creation. Burst 2 saves to both `result` and `batna_watna`; `generate_proposal_revision` tries `batna_watna` first and falls back to `result`. If setting up a fresh database:

```sql
ALTER TABLE ai_analysis
ADD COLUMN IF NOT EXISTS batna_watna JSONB;
```

**Dispute types** (defined in `ai/schemas.py`):

```python
class DisputeType(str, Enum):
    LANDLORD_TENANT     = "landlord_tenant"
    EMPLOYMENT          = "employment"
    COMMERCIAL_CONTRACT = "commercial_contract"
    NEIGHBOUR_DISPUTE   = "neighbour_dispute"
    FAMILY_BUSINESS     = "family_business"
    CONSTRUCTION        = "construction"
    CONSUMER            = "consumer"
    DEBT_RECOVERY       = "debt_recovery"
    OTHER               = "other"
```

**Classification precedence** (when a dispute could fit multiple categories, in order): `family_business` → `landlord_tenant` → `construction` → `commercial_contract` → `employment` → `debt_recovery` → `neighbour_dispute` → `consumer` → `other`. Full boundary-case rules are documented in the prompt inside `ai/subsystems/subsystem_a.py`.

Full schema setup and RLS policies are in [`docs/database.md`](docs/database.md).

---

## 8. Local Development Setup

### Prerequisites

- Python 3.11+
- Node.js 18+
- Git
- A Supabase project (free tier works)
- A Groq API key (free tier works)
- An Upstash Redis database (free tier works)

### Backend Setup

```bash
# Clone repo
git clone https://github.com/nlu-liic-shimla/nlu-mediation-platform.git
cd nlu-mediation-platform

# Create virtual environment (from repo root)
python -m venv venv
venv\Scripts\activate           # Windows
# source venv/bin/activate      # Mac/Linux

# Install dependencies
cd nlu-backend
pip install -r requirements.txt

# Set up environment variables
copy .env.example .env          # Windows
# cp .env.example .env          # Mac/Linux
# Edit .env with your actual values (see Section 10)

# Run FastAPI backend (from nlu-backend/ — main.py is top-level, not inside app/)
uvicorn main:app --reload --port 8000
```

API docs: `http://localhost:8000/docs`
Health check: `http://localhost:8000/api/v1/health`

### Celery Worker Setup (separate terminal)

```bash
cd nlu-backend
# Activate the same venv as above, then:
celery -A tasks.celery_app worker --pool=solo --loglevel=info
```

`--pool=solo` is required on Windows. On Linux/Mac you can use `--pool=prefork` for better performance.

> Note: `nlu-backend/tasks.py` (root level) is the canonical Celery task file used by the running app. `app/worker/celery_app.py` is an older/parallel setup kept around from earlier dev/smoke testing — don't build new features on it.

### Frontend Setup (separate terminal)

```bash
cd frontend
npm install

# Copy env template from backend folder
copy ..\nlu-backend\frontend.env.example .env    # Windows
# cp ../nlu-backend/frontend.env.example .env    # Mac/Linux
# Set VITE_API_URL=http://localhost:8000

npm run dev
# Runs on http://localhost:5173
```

### Running Tests

```bash
# From repo root, with venv activated
python -m pytest tests/test_subsystem_a.py -v
python -m pytest tests/test_burst1_pipeline.py -v
python -m pytest tests/test_scenario.py -v
```

Test scenarios **S-01 through S-12** in `tests/scenarios/` cover all 9 dispute types plus edge cases: landlord/tenant, freelance invoice, wrongful dismissal, defective product, business partnership dissolution, property boundary, construction delay, vague/ambiguous dispute (edge case), medical negligence, co-founder equity, debt recovery, and a double-booking refund dispute.

---

## 9. Deployment

### Current Setup

| Service | Platform | Notes |
|---|---|---|
| Frontend | Vercel | https://nlu-mediation-platform.vercel.app |
| Backend API | Render (Web Service) | FastAPI via Uvicorn |
| Celery Worker | Render (Web Service) | Via `worker_web.py` |
| Database | Supabase | PostgreSQL + Storage |
| Redis | Upstash | Celery broker and result backend |

### Celery Worker — Important Note

Render's free tier does not support Background Workers. The Celery worker runs as a Web Service using `worker_web.py`, which starts the Celery worker in a background thread and exposes a `/` health endpoint.

UptimeRobot pings the worker URL every 5–10 minutes to prevent Render's free tier from spinning down due to inactivity.

If you upgrade to a paid Render plan: switch to a proper Background Worker service, remove the health endpoint dependency from the Celery service, and remove UptimeRobot for the worker URL (keep it for the main backend if that's still on a free tier).

### Render — Backend Web Service Settings

```
Root Directory:  nlu-backend
Build Command:   pip install -r requirements.txt
Start Command:   uvicorn main:app --host 0.0.0.0 --port $PORT
```

### Render — Celery Web Service Settings

```
Root Directory:  nlu-backend
Build Command:   pip install -r requirements.txt
Start Command:   python worker_web.py
```

### Vercel — Frontend Settings

```
Root Directory:  frontend
Framework:       Vite
Build Command:   npm run build
Output Dir:      dist
```

Set `VITE_API_URL` to your Render backend URL in the Vercel environment variables.

### Checking Logs

```
Backend logs:  Render Dashboard → backend service → Logs
Celery logs:   Render Dashboard → celery service → Logs
               Look for: [Burst 1] and [Burst 2] log lines
DB queries:    Supabase Dashboard → SQL Editor
```

**Checking if Celery is actually processing:** look for `[Burst 1] Starting for case {id}` followed by `[Burst 1] Pipeline COMPLETE for case {id}` in the Celery logs. If you see "Starting" but never "COMPLETE", the task is stuck or failed silently — check for exceptions above it in the log.

---

## 10. Environment Variables

### Backend (`nlu-backend/.env`)

Copy `nlu-backend/.env.example` to `nlu-backend/.env`.

```bash
# Supabase
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-supabase-anon-key

# Database (optional — direct Postgres connection)
DATABASE_URL=postgresql://postgres:password@host:6543/postgres?pgbouncer=true

# Redis (Upstash)
REDIS_URL=rediss://:your-password@your-host.upstash.io:6380
CELERY_BROKER_URL=rediss://:your-password@your-host.upstash.io:6380

# Groq AI (required — used by all AI sub-systems)
GROQ_API_KEY=gsk_your_groq_api_key

# JWT
JWT_SECRET=your-generated-jwt-secret-min-32-chars
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24
```

Generate a JWT secret:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

> **Note:** `nlu-backend/.env.example` also lists `ANTHROPIC_API_KEY` for an optional Claude fallback/swap. The running codebase currently uses Groq via `GROQ_API_KEY` — see `ai/utils/ai_client.py` for the swap-over comment block if migrating to Claude for production.

### Frontend (`frontend/.env`)

Copy `nlu-backend/frontend.env.example` to `frontend/.env`.

```bash
VITE_API_URL=http://localhost:8000
VITE_SUPABASE_URL=https://your-project-id.supabase.co
VITE_SUPABASE_ANON_KEY=your-supabase-anon-key
```

**Never commit real `.env` files to GitHub.** They are excluded via `.gitignore`.

---

## 11. Known Limitations and Future Work

### Current Limitations

| Area | Limitation |
|---|---|
| Language | English only. Hindi statements produce poor extraction results. |
| Documents | Uploaded documents are visible to the mediator via signed URLs but are not processed by any AI sub-system — the AI only reads party statements. |
| Notifications | No email notification system. Parties must check the dashboard. |
| Signature | Settlement uses typed name + uploaded signature image — not legally equivalent to Aadhaar eSign. |
| Video | No live session capability. |
| Legal precedent | AI analysis is based on party statements alone, not Indian case law. |
| Cost monitoring | No per-case token tracking or API cost alerts. |
| Free tier | Render free tier spins down after 15 minutes of inactivity. UptimeRobot is used as a workaround, adding cold-start latency. |

### Planned Improvements (Priority Order)

1. **RAG with Indian case law** — add pgvector to Supabase, seed 150+ Supreme Court / High Court judgements, embed and index by dispute type, retrieve top-3 relevant precedents per case, inject as context into Sub-systems A and D. Highest-impact item — reduces hallucination in legal analysis.
2. **Hindi language support** — detect language of party statement, translate to English before the AI pipeline, optionally display analysis back in Hindi.
3. **Email notifications** — SendGrid integration for invitation sent, proposal published, application accepted/rejected.
4. **Aadhaar eSign integration** — replace typed name + image signature with NSDL/CDSL API for legally stronger settlement agreements.
5. **Per-sub-system temperature tuning** — currently all sub-systems use `temperature=0.1`; classification tasks (A, C, G) could use `0.0` for consistency, generation tasks (B, H) could use `0.2` for more natural language.
6. **Evaluation framework** — ground truth annotations for all test scenarios, automated accuracy measurement per deployment, before/after tracking on prompt changes.
7. **Video mediation sessions** — WebRTC or Jitsi integration, with session recording and transcript storage.

---


## Things That Look Like Bugs But Are Not

- **`cases.created_by = mediator` even in party-initiated flow** — Required by Mediation Act 2023. The mediator formally creates every case.
- **Both parties have `role = party_user`** — Case-specific role (`requesting_party` / `against_party`) is determined via `case_invitations` and `GET /cases/{id}/my-role`.
- **Mediatability score never changes on re-runs** — Score is deterministic Python, not LLM output.
- **Mediator notes not in Burst 1 or Burst 2** — Notes are fed to Sub-system H only, during proposal revision.
- **Sub-system E runs on raw text before Sub-system A** — Intentional pipeline order. Do not reorder.
- **Celery deployed as Web Service with health endpoint** — Render free tier limitation.
- **`audit_logs` has no update or delete endpoint** — Insert-only for legal tamper-proof requirement.

---

Sulah — AI-Powered Mediation Platform
National Law University Shimla | LIIC | 2026
