# SULAH — AI-Powered Mediation Platform
**NLU Shimla — Intern Project 2026**

This document is for whoever inherits this codebase next. It explains what the project does, how the pieces fit together, what's solid, what's fragile, and where to look first.

---

## 1. What this project is

SULAH is a platform that lets two parties in a dispute (landlord/tenant, employment, business, neighbours, etc.) go through an AI-assisted mediation process with a human mediator supervising throughout. Nothing is fully automated — the AI drafts, extracts, and scores; the mediator reviews, edits, and publishes everything that a party actually sees.

**High-level flow:**
1. A mediator creates a case (or a party applies and a mediator accepts it) → both parties get one-time invitation links.
2. Both parties fill out an intake wizard describing the dispute ("submissions").
3. Once both submit, an AI pipeline ("Burst 1") runs: tone analysis, bias removal, conflict extraction, neutral summary, mediatability scoring.
4. Mediator reviews Burst 1 results, then sends an AI-generated questionnaire to both parties.
5. Once both answer, a second AI pipeline ("Burst 2") runs BATNA/WATNA (best/worst alternative to negotiated agreement) analysis.
6. Mediator drafts a settlement proposal (AI-assisted first draft), edits it, and publishes it.
7. Parties accept or reject. Rejections trigger an AI-generated revision (Sub-system H) for the next round (up to `max_rounds`, default 3, extendable).
8. Once both accept, the mediator finalises the case, both parties digitally confirm, and a settlement PDF is generated and made available for download.

---

## 2. Repo layout

```
/
├── ai/                      Python AI subsystems (no web framework — pure functions)
│   ├── schemas.py           SINGLE SOURCE OF TRUTH for all AI output shapes (Pydantic)
│   ├── subsystems/          subsystem_a.py .. subsystem_h.py (see section 4)
│   ├── pipeline_burst1.py   Orchestrates Burst 1 (F, E, A, B, G)
│   ├── pipeline_burst2.py   Orchestrates Burst 2 (D)
│   ├── proposal_draft.py    First-draft proposal generator (plain text, not JSON)
│   └── utils/ai_client.py   Shared LLM call wrapper (currently Groq, swappable to Claude)
│
├── nlu-backend/             FastAPI backend
│   ├── main.py              App entrypoint, CORS config
│   ├── app/
│   │   ├── api/v1/routes/   auth, cases, documents, invitations, questionnaires, proposals, settlement
│   │   ├── core/            security (JWT/bcrypt), database (Supabase client), state_machine.py, dependencies (auth guards)
│   │   ├── models/          Pydantic request/response models
│   │   └── services/pdf_generator.py   ReportLab settlement PDF builder
│   ├── tasks.py             ⚠️ ACTUAL Celery tasks used in production (root-level, NOT app/worker/tasks.py)
│   └── app/worker/celery_app.py   Older/parallel Celery setup — see warning in section 6
│
├── frontend/                React (Vite) SPA
│   └── src/
│       ├── pages/mediator/  Mediator-facing screens
│       ├── pages/party/     Party-facing screens
│       ├── api/ and services/  Two parallel axios client setups (see section 6)
│       └── routes/AppRoutes.jsx   All routing, role-gated
│
├── tests/                   Python test scripts (not pytest — run as `python -m tests.xxx`) + scenario JSON fixtures
└── docs/                    api-contract.md, state-machine.md, database.md — written specs (partially stale, see section 7)
```

---

## 3. Tech stack

| Layer | Choice |
|---|---|
| Backend | FastAPI (Python), Supabase (Postgres + Storage), Redis (Upstash) + Celery for background jobs, JWT auth (python-jose + bcrypt) |
| AI | Currently **Groq** (`openai/gpt-oss-120b` / `openai/gpt-oss-20b`) for cost reasons during dev. Designed to swap to Claude for production — see `ai/utils/ai_client.py`, the swap is a few lines (model names + client import) |
| Frontend | React 19 + Vite 8 + React Router 7, plain inline `style={}` objects (no CSS framework in most pages; Tailwind is configured but barely used), Recharts for dashboard charts |
| PDF generation | ReportLab (settlement PDF), with a DejaVu Sans font pulled from the `matplotlib` install specifically so the ₹ symbol renders |

---

## 4. The AI pipeline (the heart of the project)

All AI subsystems live in `ai/subsystems/` and share Pydantic schemas from `ai/schemas.py`. **Never bypass schemas.py** — every downstream piece (Celery tasks, routes, frontend types implicitly) depends on those exact field names.

| Sub-system | File | Input | Output | Runs in |
|---|---|---|---|---|
| A — Conflict Extraction | `subsystem_a.py` | raw/cleaned party statements | `ConflictExtraction` (dispute_type, claims, disputed/undisputed facts, confidence) | Burst 1 — **critical path**, pipeline stops if this fails |
| B — Neutral Summary | `subsystem_b.py` | `ConflictExtraction` | `NeutralSummary` | Burst 1 |
| C — Questionnaire | `subsystem_c.py` | `ConflictExtraction` | `QuestionnaireOutput` (8-10 targeted questions) | Triggered manually by mediator, between Burst 1 and Burst 2 |
| D — BATNA/WATNA | `subsystem_d.py` | `ConflictExtraction` + questionnaire answers | `BatnaWatnaOutput` (per-party strength labels; **numeric scores are internal only**, parties never see them) | Burst 2 |
| E — Bias Removal | `subsystem_e.py` | raw statement OR `NeutralSummary` | `BiasRemovalOutput` | Burst 1 (runs twice: once on raw text before A, once on the generated summary) |
| F — Tone Analysis | `subsystem_f.py` | raw statements | `ToneAnalysis` (mediator-only, never shown to parties) | Burst 1 |
| G — Mediatability Score | `subsystem_g.py` | `ConflictExtraction` | `MediatabilitySore` (yes, spelled "Sore" — intentional, do not rename, it's wired everywhere) — **score is deterministic Python math, not LLM output**; only the justification text comes from the LLM | Burst 1 |
| H — Proposal Revision | `subsystem_h.py` | rejected proposal text + rejection reasons + BATNA/WATNA | revised draft + changes summary | Triggered on any party rejection |

Pipeline order (`ai/pipeline_burst1.py`): **F on raw text → E on raw text (Party A & B separately) → A on E's cleaned output → B on A's output → G on A's output**, run in that sequence (not fully parallel despite the docs occasionally saying so — the actual code in `pipeline_burst1.py` and `tasks.py` runs sequentially).

**Known quirks to know about before touching this:**
- `ai/utils/ai_client.py` uses Groq's free tier. It has a `call_with_retry` wrapper (3 retries, feeds the previous error back to the model) used for structured JSON output, plus `call_large_text` / `call_large_json` for plain-text/dict output used by proposal drafting and revision.
- Every subsystem function returns either a validated Pydantic model **or** a `{"status": "failed", ...}` dict. **Always check `is_failed(result)` before touching fields** — this convention is used everywhere, don't break it.
- `BatnaWatnaOutput` has a `model_validator` that force-corrects `watna_score > batna_score` by averaging both — this is a safety net, not a bug.

---

## 5. State machine (this is the backbone of the backend)

`nlu-backend/app/core/state_machine.py` is the **single source of truth** for case status. The golden rule, enforced by convention (not by DB trigger): **no route, no Celery task ever writes `cases.status` directly** — everything goes through `transition(case_id, new_state, actor_id)`, which validates the transition against a `VALID_TRANSITIONS` set and writes an audit log row atomically-ish (two separate REST calls, not a real DB transaction — see the long comment in the file for why that's an accepted tradeoff for MVP).

Full state list and transition table: see `docs/state-machine.md` (kept mostly in sync, but double check against the `VALID_TRANSITIONS` set in code — code wins if they disagree).

Key states: `BOTH_INVITED → FIRST_PARTY_SUBMITTED → BOTH_SUBMITTED → BURST_1_PROCESSING → BURST_1_COMPLETE → QUESTIONNAIRE_ACTIVE → QUESTIONNAIRE_COMPLETE → BURST_2_PROCESSING → BURST_2_COMPLETE → PROPOSAL_DRAFT → PROPOSAL_PUBLISHED → (MEDIATION_COMPLETE | MEDIATION_IN_PROGRESS → loop back to PROPOSAL_DRAFT) → MEDIATION_COMPLETE`. Failures at either AI burst go to `PROCESSING_FAILED`, from which a mediator can retry.

If you ever get a 409 `INVALID_TRANSITION` and think it's wrong, **the fix is always to add the pair to `VALID_TRANSITIONS`**, never to call the DB directly to bypass it.

---

## 6. ⚠️ Things that are duplicated / inconsistent — read before you refactor

This codebase was built by multiple people in parallel across several weeks, and a few things never got fully reconciled. A new developer should know about these rather than accidentally "fixing" one and breaking the one actually in use.

- **Two Celery task files exist**: `nlu-backend/tasks.py` (root-level) is the one actually imported by routes (`from tasks import process_burst_1`, etc.) and used in production. `nlu-backend/app/worker/celery_app.py` is an earlier/parallel setup with its own `hello_task`, `process_burst_1`, etc. **Treat `tasks.py` as canonical.** The `app/worker/` version appears to be legacy — confirm before deleting, but don't build on it.
- **Two frontend API client setups**: `frontend/src/api/client.js` (raw axios, base URL only) and `frontend/src/services/api.js` (axios with `/api/v1` baked into the base URL, plus a chunk of dev-only mock/localStorage-override logic for demoing settlement PDFs — see the `generateJSMediatorPDF` function and the interceptor overrides in that file). Most newer pages import from `services/api.js`. **Before wiring a new page, check which one sibling pages use** — mixing them causes double `/api/v1/api/v1/` prefix bugs (this has happened before, see comments in `frontend/src/api/cases.js`).
- **`frontend/src/services/api.js` contains demo/mock-mode logic** (checks `localStorage` for `case_status_${caseId}` overrides, generates a client-side jsPDF settlement doc as a fallback). This looks like leftover demo scaffolding from a presentation. It should probably be removed before a real production deploy, but confirm with the team first — some pages may rely on it still working offline.
- **`ai/schemas.py`'s `MediatabilitySore` class name has a typo** ("Sore" not "Score"). This is intentional/frozen — it's referenced by name across the Celery pipeline, routes, and possibly frontend expectations. Don't "fix" the typo without a full search-and-replace across all three layers.
- **Groq vs Claude**: the whole AI layer currently runs on Groq's free tier for cost reasons during development. Before any real deployment, swap `LARGE_MODEL`/`SMALL_MODEL` in `ai/utils/ai_client.py` to Claude model strings and swap the `Groq` client for `Anthropic` (the comment block at the top of that file has the intended swap already sketched out).
- **`docs/*.md` files lag behind code** in places (e.g. `docs/api-contract.md`'s error-code names, `docs/state-machine.md`'s Week 2 renames section). Treat these as *helpful context*, not ground truth — always check the actual route/model code.

---

## 7. Environment variables you'll need

Backend (`nlu-backend/.env`):
```
SUPABASE_URL=...
SUPABASE_KEY=...          # service role key for Celery tasks (not anon key)
JWT_SECRET=...            # min 32 chars
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24
REDIS_URL=rediss://...    # Upstash, note double-s scheme for TLS
DATABASE_URL=postgresql://...
```
AI (`ai/` reads from the same or a root `.env` via `load_dotenv()`):
```
GROQ_API_KEY=...
ANTHROPIC_API_KEY=...     # for when you swap to Claude
```
Frontend (`frontend/.env` or `Env.env`):
```
VITE_API_URL=http://localhost:8000
VITE_SUPABASE_URL=...
VITE_SUPABASE_ANON_KEY=...
```

---

## 8. Running it locally

**Backend:**
```
cd nlu-backend
python -m venv venv && venv\Scripts\activate   # or source venv/bin/activate on mac/linux
pip install -r requirements.txt
copy .env.example .env   # then fill in real values
uvicorn main:app --reload
```

**Celery worker** (needed for AI pipelines to actually run — without it, cases will sit stuck in `*_PROCESSING` forever):
```
cd nlu-backend
celery -A tasks worker --loglevel=info --pool=solo
```
`--pool=solo` is required on Windows (no `prefork` support there); keep it unless you've confirmed a different deployment target.

**Frontend:**
```
cd frontend
npm install
npm run dev
```

**AI subsystem tests** (from repo root, not pytest-based):
```
python -m tests.test_everything          # runs everything, verbose
python -m tests.test_scenario S-01       # runs one scenario end-to-end through all subsystems
python -m tests.test_subsystem_a         # tests just conflict extraction across all 8 scenarios
```
Scenario fixtures live in `tests/scenarios/S-01.json` .. `S-12.json` and cover: landlord/tenant, freelance invoice, wrongful dismissal, defective product, business partnership dissolution, property boundary, construction delay, vague/ambiguous (edge case), medical negligence, co-founder equity, debt recovery, and a double-booking refund case.

---

## 9. Database

Full SQL and RLS setup is in `docs/database.md` (it's written as a running changelog across Weeks 1–4, apply sections in order if setting up fresh). Key tables: `users`, `cases`, `case_invitations`, `submissions`, `documents`, `ai_analysis`, `questionnaires`, `questionnaire_responses`, `proposals`, `proposal_responses`, `settlement_confirmations`, `mediation_reports`, `audit_logs` (insert-only, enforced via RLS — never add UPDATE/DELETE policies to this table).

`audit_logs` currently has two different foreign-key patterns floating around (`case_id` for real cases, `application_id` for pre-acceptance application requests, because `application_requests` isn't in the `cases` table yet at that point) — see the long comments in `nlu-backend/app/api/v1/routes/cases.py` around `_write_audit_log_safe` for why.

---

## 10. Suggested first tasks for a new developer

1. Run the local setup end-to-end (backend + Celery + frontend) and walk through one full case lifecycle using scenario S-01 data, to build a mental model.
2. Decide the fate of the duplicate Celery/API-client files (section 6) — pick one, delete/deprecate the other, update imports.
3. Do the Groq → Claude swap in `ai/utils/ai_client.py` before any real usage, since Groq's free tier has rate limits that already show up as retry logic scattered through the questionnaire-send endpoint.
4. Reconcile `docs/*.md` against actual code, or generate fresh docs from code — the written specs are useful context but not reliable as of this handoff.
5. Review `frontend/src/services/api.js` for the mock/demo logic and decide whether it ships to production.

---

*This README was generated as a handoff document summarizing the existing codebase (backend, frontend, and AI subsystems) as of the current state of the repository. It is not a substitute for reading the actual code, especially `ai/schemas.py` and `app/core/state_machine.py`, which are the two files everything else depends on.*﻿
