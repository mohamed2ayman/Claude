# SIGN + CENVOX Platform — Comprehensive Build Status & Team Knowledge Report

> **Report date:** 2026-04-25
> **Branch:** `claude/sad-gauss-75017e` (worktree)
> **Scope:** Full monorepo audit — runtime + source — performed by Claude Opus 4.7
> **Audience:** Engineering team, technical leadership, onboarding new contributors

---

## Executive Summary

The SIGN + CENVOX platform is a construction-contract lifecycle management system with three layers: a NestJS 10 backend, a FastAPI/Celery AI backend (Anthropic Claude + OpenAI embeddings + pgvector), and two React 18 + Vite frontends (`apps/sign` user/admin app and `apps/cenvox` public landing). All seven Docker services run cleanly together (Postgres 15+pgvector, Redis 7, NestJS, FastAPI, Celery worker, SIGN frontend, CENVOX frontend) and the health endpoints respond. The backend exposes **33 controllers / ~210 endpoints** spanning authentication, MFA, contracts, parties, clauses, comments, versions, approvals, organizations, projects, knowledge assets, obligations, notifications, payments, audit logs, dashboards, and more. The AI backend exposes **11 FastAPI routes** that fan out to **9 Celery tasks** (risk, summarize, diff, obligations, conflict, chat, research, OCR text extract, clause extract) all using the async-job pattern (`POST → {job_id} → poll`). The data model is built on **33 TypeORM entities** with **21 migrations** applied through `init-db.sql` plus migration files. Realtime uses a single Socket.IO gateway (`/collaboration` namespace) with per-contract rooms.

The user-facing experience is uneven. The SIGN client portal is broad: **21 `/app/*` routes** covering dashboard, projects, contracts, contract editor, knowledge assets, obligations, notifications, organization, members, billing, plans, MFA setup, etc. The Admin portal has **16 routes** covering organizations, users, plans, billing, knowledge curation, audit, and observability — but `/admin/account-settings` is still the placeholder `AdminComingSoonPage`. The Contractor portal is essentially a stub: a single dashboard page with hardcoded zeros and "no contracts" messaging — there is no contractor inbox, no signing flow, no document review surface. The CENVOX landing renders 6 product cards but only **SIGN** is `available: true`; the other 5 (Cenvox Hub, BidSign, ProcurePro, ContractorOS, FieldSign) are coming-soon placeholders with hardcoded content in a single 1041-line `App.tsx`.

Integrations are wired but not all production-validated. **Paymob** is fully implemented end-to-end (auth → order → payment_keys → HMAC SHA-512 webhook → subscription activation + transaction record) with a dev fallback that returns mock keys when `PAYMOB_API_KEY` is unset. **DocuSign** has a JWT Grant service and an unauthenticated webhook endpoint that always 200s — the webhook handler does not yet update contract state. **AWS S3 + SES** clients exist for production storage/email, with local-disk + nodemailer/SMTP fallbacks for dev. **Two Bull queues** are active: `email-queue` (transactional email send) and `obligation-reminders` (cron-style scan that flips `OVERDUE` and notifies creators) — no other background processing exists yet. The collaboration gateway broadcasts clause/comment/status/risk events but there is no presence persistence and no operational-transform conflict resolution.

The biggest cross-cutting risks are: (1) **no automated tests** anywhere in the repo (no `*.spec.ts`, no `pytest`, no Vitest), (2) **no `CLAUDE.md`** at the repo root or in any sub-app — every new AI session re-discovers the architecture from cold, (3) **no `lessons.md`** — past mistakes are not captured, (4) **a wrong default API URL in `apps/sign/src/services/api/axios.ts`** (`http://localhost:3001/api` instead of `http://localhost:3000/api/v1`) which is masked by `VITE_API_URL` in normal use but will bite anyone running without a `.env`, (5) **seed-script role drift** — `youssef141162@gmail.com` is documented in onboarding as `OWNER_ADMIN` but the seed creates him as `SYSTEM_ADMIN`, (6) **the contractor experience does not exist** beyond a stub page, despite being a core persona, and (7) **the DocuSign webhook is a no-op** — envelope status changes never reach the contract. Estimated overall completion is **~63%**: backend ~80%, AI backend ~75%, SIGN client portal ~70%, Admin portal ~75%, Contractor portal ~10%, CENVOX landing ~30% (1 of 6 products live), tests/docs/CI 0%.

---

## Overall Completion Estimate

| Layer | Completion | Notes |
|---|---|---|
| **Backend (NestJS)** | ~80% | 32 modules, 33 controllers, ~210 endpoints, 21 migrations, 2 Bull processors. Missing: tests, distributed locks, refresh-token rotation hardening, DocuSign webhook handler. |
| **AI Backend (FastAPI + Celery)** | ~75% | 11 routes, 9 Celery tasks, embeddings + pgvector ingest/search. Missing: tests, retry/backoff policies, dead-letter handling, observability beyond Celery logs. |
| **SIGN Client Portal (`/app/*`)** | ~70% | 21 routes, i18n (en/ar) + RTL, full contract editor, billing + plans. Missing: contract diff UI polish, obligation kanban, end-to-end signing flow, client-side error boundaries on all pages. |
| **SIGN Admin Portal (`/admin/*`)** | ~75% | 16 routes including organizations, users, plans, billing, knowledge curation, audit, observability. Missing: `account-settings` (still ComingSoon), real charting on dashboards, role-management UI. |
| **SIGN Contractor Portal (`/contractor/*`)** | ~10% | Only a stub dashboard with hardcoded zeros. No inbox, no signing, no document review. |
| **CENVOX Landing (`apps/cenvox`)** | ~30% | 1 of 6 product cards is live (SIGN). 1041-line single-file landing, hardcoded content, hardcoded `localhost:5173` link. |
| **Database & Migrations** | ~85% | 33 entities, 21 migrations, pgvector + uuid-ossp installed via `init-db.sql`. Missing: data retention policies, partitioning for audit/notification tables. |
| **Realtime (Socket.IO)** | ~60% | One `/collaboration` gateway, JWT auth, per-contract rooms, broadcasts but no OT, no presence persistence, no per-clause locks. |
| **Integrations (Paymob, DocuSign, S3, SES)** | ~65% | Paymob 100% (incl. webhook). DocuSign service yes, webhook is no-op. S3/SES code paths exist; storage currently uses local disk. |
| **Tests** | 0% | No backend specs, no AI tests, no frontend tests. |
| **CI/CD** | 0% | No `.github/workflows/`, no pipeline definitions found. |
| **Documentation** | ~15% | Has `README.md`, Swagger at `/api/docs`, this report; no `CLAUDE.md`, no `lessons.md`, no architecture diagrams, no runbook. |
| **Overall** | **~63%** | Functional core, weak edges. Production-ready for internal pilots, not for external GA. |

---

# Part A — Architecture & Inventory

## A.1 SIGN User Dashboard (`apps/sign` client portal `/app/*`)

**Tech:** React 18, Vite 5, Redux Toolkit + React Query, react-router-dom v6, MUI + Tailwind utility classes, i18next (en/ar) with automatic RTL.

**Routes (21):**

1. `/app` → DashboardPage (overview cards, recent contracts, quick actions)
2. `/app/projects` → ProjectsPage (list + create)
3. `/app/projects/:id` → ProjectDetailPage
4. `/app/contracts` → ContractsListPage (filterable by status / type)
5. `/app/contracts/new` → CreateContractPage (35+ FIDIC/NEC types or ADHOC)
6. `/app/contracts/:id` → ContractDetailPage (reads + actions)
7. `/app/contracts/:id/edit` → ContractEditorPage (clauses, parties, comments, versions)
8. `/app/contracts/:id/risk` → ContractRiskPage (AI risk analysis output)
9. `/app/contracts/:id/obligations` → ContractObligationsPage
10. `/app/knowledge-assets` → KnowledgeAssetsPage (org library)
11. `/app/knowledge-assets/:id` → KnowledgeAssetDetailPage
12. `/app/obligations` → ObligationsPage (cross-contract list)
13. `/app/notifications` → NotificationsPage
14. `/app/organization` → OrganizationPage (settings)
15. `/app/organization/members` → MembersPage (invites, roles)
16. `/app/billing` → BillingPage (current plan, transactions)
17. `/app/plans` → PlansPage (upgrade/downgrade, Paymob iframe)
18. `/app/profile` → ProfilePage
19. `/app/security` → SecurityPage (password, sessions)
20. `/app/security/mfa` → MfaSetupPage (TOTP enroll, recovery codes)
21. `/app/audit` → user-scoped activity feed

**State:**
- Auth slice (JWT access + refresh, role, MFA flags)
- Org slice (active org, subscription)
- Notifications slice (unread counter polled + WS push)
- React Query for all list/detail fetches with stale-time tuned per resource

**i18n:** `apps/sign/src/i18n/index.ts` configures i18next + LanguageDetector, two JSON locales (`en`, `ar`), and on `languageChanged` flips `document.documentElement.dir` and `lang` so the entire app re-flows for RTL Arabic.

**Realtime:** ContractEditorPage joins `contract:{id}` room on mount; subscribes to `clause:updated`, `comment:added`, `status:changed`, `risk:updated`, `user:joined`, `user:left`.

**Known gaps:**
- No global error boundary on `App.tsx`; per-page boundaries inconsistent.
- Obligation kanban / calendar is list-only.
- Diff viewer renders text-only diffs; no side-by-side rich-text view.
- ContractEditorPage version compare is wired to API but UI is minimal.

## A.2 SIGN Admin Portal (`/admin/*`)

**Layout:** All admin routes are wrapped in `AdminLayout` (sidebar + topbar). Role-gated to `SYSTEM_ADMIN` and `OPERATIONS`.

**Routes (16):**

1. `/admin` → AdminDashboardPage (KPIs)
2. `/admin/organizations` → OrganizationsListPage
3. `/admin/organizations/:id` → OrganizationDetailPage
4. `/admin/users` → UsersListPage
5. `/admin/users/:id` → UserDetailPage (impersonate, suspend, reset MFA)
6. `/admin/plans` → PlansAdminPage (CRUD subscription plans)
7. `/admin/plans/new` → CreatePlanPage
8. `/admin/plans/:id` → EditPlanPage
9. `/admin/billing` → BillingAdminPage (PaymentTransaction list, refunds)
10. `/admin/knowledge` → KnowledgeCurationPage (org-cross asset review)
11. `/admin/knowledge/:id` → KnowledgeAssetReviewPage
12. `/admin/audit` → AuditLogPage (filter by user/action/resource)
13. `/admin/observability` → ObservabilityPage (queue depth, errors)
14. `/admin/notifications` → NotificationsAdminPage (system broadcasts)
15. `/admin/account-settings` → **`AdminComingSoonPage`** (placeholder)
16. `/admin/portal-select` → portal chooser (when admin also has a tenant context)

**Known gaps:**
- `account-settings` is an empty placeholder.
- Observability page reads from a stubbed endpoint; no real metrics integration (Prometheus/Grafana not wired).
- No role-management UI; roles must be changed via DB or seeds.

## A.3 CENVOX Landing (`apps/cenvox`)

**Single file:** `apps/cenvox/src/App.tsx` — 1041 lines. Sections (11): Hero, Trust badges, Product Suite (6 cards), Why Cenvox, How It Works, Built For, Security, Integrations, Testimonials, Pricing CTA, Footer.

**Product cards (6):**
| Card | Status | Link |
|---|---|---|
| **SIGN** | `available: true` | hardcoded `http://localhost:5173` |
| Cenvox Hub | coming soon | — |
| BidSign | coming soon | — |
| ProcurePro | coming soon | — |
| ContractorOS | coming soon | — |
| FieldSign | coming soon | — |

**UX:** custom cursor follower, IntersectionObserver-driven reveal animations.

**Known gaps:**
- All copy hardcoded; no CMS, no i18n.
- `SIGN_URL` is hardcoded `localhost` — must be replaced with env-driven URL before deploy.
- No SEO meta, no OG tags, no sitemap.

## A.4 Backend API (NestJS 10)

**Entry:** `backend/src/main.ts` sets global prefix `api/v1`, helmet, CORS (FRONTEND_URL + 5180/5174 in dev), `ValidationPipe({ whitelist: true, transform: true })`, Swagger UI at `/api/docs`.

**App module:** 32 feature modules + Health + TypeOrmModule.forRootAsync (`autoLoadEntities: true`) + BullModule.forRootAsync (Redis URL).

**Modules:** auth, users, organizations, projects, contracts, parties, clauses, comments, versions, approvals, knowledge-assets, obligations, notifications, ai, embeddings, subscriptions (incl. payments), invitations, sessions, audit, dashboards, storage, docusign, collaboration (WS gateway), mfa, recovery-codes, email, payments, observability, admin, health, plus shared `database/entities`.

**Controllers (33) / Endpoints (~210):** `AuthController` 16 endpoints (login, register, refresh, logout, MFA setup/verify/enable/disable, recovery codes, password reset). `ContractsController` 27 endpoints (CRUD + parties + clauses + versions + comments + responses + approvers + request-approval + review-approval + pending-approvals/mine). `OrganizationsController`, `ProjectsController`, `KnowledgeAssetsController`, `ObligationsController`, `NotificationsController`, `SubscriptionsController` (plans CRUD + create-payment-intention + activate + Paymob webhook + transactions list), `AdminController`, `DashboardsController`, `AuditController`, `UsersController`, `InvitationsController`, `MfaController`, `DocusignWebhookController`, etc.

**Auth:**
- `JwtAccessGuard` (passport-jwt) on all protected routes.
- `RolesGuard` enforces `@Roles(...)` decorators against `UserRole` enum.
- 15-min access token + 7-day refresh token (separate secrets).
- MFA via `otplib` TOTP + email OTP fallback + bcrypt-hashed recovery codes.

**Validation:** all DTOs use `class-validator` and the global `ValidationPipe` whitelists/transforms.

**Known gaps:**
- No tests (`*.spec.ts` absent).
- Refresh tokens are not rotated on use; revocation list is in-memory.
- No rate limiting on auth endpoints (no `@nestjs/throttler` config).
- Some controllers swallow upstream errors (e.g., `subscriptions.service.ts:362` `try { ... } catch {}`).

## A.5 AI Backend (FastAPI + Celery)

**Entry:** `ai-backend/main.py` mounts `agents` + `embeddings` routers; `/health` endpoint.

**Routes (11):** `POST /agents/risk-analysis`, `/summarize`, `/diff`, `/extract-obligations`, `/chat`, `/research`, `/extract-text`, `/extract-clauses`, `/detect-conflicts`; `GET /agents/jobs/{job_id}`; `POST /embeddings/ingest`, `POST /embeddings/search`.

**Celery tasks (9):** `run_risk_analysis`, `run_summarize`, `run_diff_analysis`, `run_extract_obligations`, `run_conflict_detection`, `run_chat`, `run_research`, `run_extract_text` (30-min limit for Arabic OCR), `run_extract_clauses` (40-min limit). Worker `max_tasks_per_child` low (memory budget ~500KB before restart) — heavy OCR uses fresh process each time to avoid leaks.

**Storage of results:** Redis backend; status read by `GET /agents/jobs/{job_id}` returning `{state, result, error}`.

**Embeddings:** OpenAI `text-embedding-3-small` → pgvector column on `KnowledgeAsset` / chunked `KnowledgeAssetChunk`. Cosine search via `<=>` operator.

**LLM:** Anthropic Claude (model id from env). Prompts are domain-specific per agent.

**Known gaps:**
- No retries/backoff policy on Celery tasks beyond defaults; a transient OpenAI 429 can fail a job permanently.
- No dead-letter queue.
- Job results expire with Redis TTL; no archival.

## A.6 Database Entities (33)

User, Organization, Membership, Invitation, Session, Project, Contract, ContractVersion, ContractParty, Clause, ClauseComment, ClauseResponse, ContractApprover, ApprovalRequest, ContractAuditLog, KnowledgeAsset, KnowledgeAssetChunk, KnowledgeAssetReview, Obligation, ObligationReminder, Notification, NotificationPreference, SubscriptionPlan, OrganizationSubscription, PaymentTransaction, AuditLog, MfaSecret, RecoveryCode, EmailVerificationToken, PasswordResetToken, AiJob, RiskAnalysis, RoleChangeLog.

**Migrations (21):** present in `backend/src/database/migrations/`. Bootstrapped by `docker/init-db.sql` which creates extensions (`pgvector`, `uuid-ossp`) and enum types (`user_role`, `contract_status`, `risk_level`, `asset_type`, `asset_review_status`, `subscription_status`, `obligation_status`, `notification_type`).

**ContractStatus state machine (12):** DRAFT → PENDING_APPROVAL → APPROVED → PENDING_TENDERING → SENT_TO_CONTRACTOR → CONTRACTOR_REVIEWING → PENDING_FINAL_APPROVAL → CHANGES_REQUESTED → RISK_ESCALATION_PENDING → ACTIVE → COMPLETED → TERMINATED. Transitions enforced inside `ContractsService` rather than as a DB constraint.

**Permission model:** 9 `UserRole` values × 4 `PermissionLevel` values (VIEWER, COMMENTER, EDITOR, APPROVER) × 20+ `JobTitle` values. `JOB_TITLE_DEFAULT_PERMISSION` map provides safe defaults.

**Seeded users (idempotent):** `youssef141162@gmail.com / Youssef@1997` and `admin@sign.com / Admin@Sign2026` — **both** seeded as `UserRole.SYSTEM_ADMIN`. Onboarding docs that say "youssef = OWNER_ADMIN" are wrong.

## A.7 Bull Jobs / Background Processing

Only **2 processors** exist:

1. **`email-queue` → `EmailQueueProcessor`** ([email-queue.processor.ts](backend/src/modules/notifications/email-queue.processor.ts)): handles `send-email` jobs, calls `EmailService.sendGenericEmail(to, subject, html)`, throws on failure to let Bull retry per queue config.
2. **`obligation-reminders` → `ObligationReminderProcessor`** ([obligation-reminder.processor.ts](backend/src/modules/obligations/obligation-reminder.processor.ts)): handles `check-reminders`, scans `PENDING`/`IN_PROGRESS` obligations with a `due_date`, flips `OVERDUE` if past due, sends notification to contract creator when `daysUntilDue <= reminder_days_before`.

**Missing:** no AI-job dispatcher queue (AI work goes directly to Celery), no PDF render queue, no S3 cleanup queue, no DocuSign envelope sync queue.

## A.8 Integrations

| Integration | Status | File / module | Notes |
|---|---|---|---|
| **Paymob** (payments, EG) | ✅ Full | [subscriptions.service.ts](backend/src/modules/subscriptions/subscriptions.service.ts) | Auth → Order → Payment Key flow; HMAC SHA-512 webhook verification ([:281](backend/src/modules/subscriptions/subscriptions.service.ts:281)); subscription activation + `PaymentTransaction` insert on success/refund/failure. Dev fallback returns mock key. |
| **DocuSign** (e-sign) | ⚠️ Partial | `backend/src/modules/docusign/` | JWT Grant auth via `docusign-esign` SDK; `POST /docusign/webhook` is unauthenticated and returns 200 unconditionally — does **not** update contract state. |
| **AWS S3** (object storage) | ⚠️ Wired, unused at runtime | `@aws-sdk/client-s3` imported | `StorageService` currently uses local-disk (`uploads/` volume) regardless. |
| **AWS SES** (prod email) | ✅ Code paths exist | `EmailService` | Production uses SES; dev uses nodemailer SMTP via env. |
| **OpenAI** (embeddings) | ✅ Active | `ai-backend/app/services/embeddings.py` | `text-embedding-3-small` |
| **Anthropic Claude** (LLM) | ✅ Active | `ai-backend/app/agents/*.py` | All AI agents |
| **Redis** | ✅ Active | docker-compose | Bull broker + Celery broker + Celery result backend |
| **PostgreSQL + pgvector** | ✅ Active | `pgvector/pgvector:pg15` | Vector search on knowledge assets |
| **Socket.IO** | ✅ Active | `collaboration.gateway.ts` | JWT-authenticated, namespace `/collaboration`, per-contract rooms |
| **i18next** | ✅ Active | SIGN frontend | en/ar with RTL flip |

## A.9 Feature Connectivity Map

```
[ apps/sign client UI ]
   │  axios (JWT, refresh interceptor)
   ▼
[ NestJS api/v1/* ]──┬─► PostgreSQL (TypeORM, 33 entities)
                    ├─► Redis ──► Bull (email-queue, obligation-reminders)
                    ├─► Socket.IO `/collaboration` (broadcasts to client UIs)
                    ├─► Paymob HTTPS (auth + order + payment_keys)  ◄── webhook ── Paymob
                    ├─► DocuSign HTTPS (JWT Grant)                  ◄── webhook ── DocuSign (no-op)
                    ├─► AWS SES (prod email) | nodemailer (dev)
                    └─► FastAPI ai-backend
                            │
                            ├─► Celery worker (9 tasks)
                            │     ├─► Anthropic Claude
                            │     └─► OpenAI embeddings ──► pgvector cols
                            └─► Redis (job state + result backend)

[ apps/cenvox landing ] ── hardcoded link to apps/sign at localhost:5173
```

**End-to-end AI flow (e.g. risk analysis):**
1. Client calls `POST /api/v1/contracts/:id/risk-analysis`.
2. NestJS `ContractsController` → `AiService.triggerRiskAnalysis(contractText)` → `POST {AI_BACKEND_URL}/agents/risk-analysis`.
3. FastAPI router enqueues `run_risk_analysis.delay(...)`, returns `{job_id, status: "queued"}`.
4. Celery worker runs the agent (Claude call), writes result to Redis backend.
5. Client polls `GET /api/v1/ai/jobs/:job_id` → NestJS proxies to FastAPI `GET /agents/jobs/{job_id}` → returns `{state, result}`.
6. On success, frontend updates risk panel; if `risk_level` changes, gateway also broadcasts `risk:updated`.

---

# Part B — The 6 Golden Rules for Working in This Repo

These rules exist because the codebase is **wide and weakly-tested**: changes that look local can break a queue processor three modules away, and a fresh AI session has no idea any of this exists.

## Rule 1 — Plan Before Coding

**Why:** This monorepo crosses three runtimes (Node, Python, browser) and six product surfaces (SIGN client, SIGN admin, SIGN contractor, CENVOX, AI backend, NestJS API). A typical "small" change touches 3–7 files. Without a plan, you will edit two and forget five.

**How to apply:** Before any change >1 file, write a short plan: (1) **Goal** in one sentence, (2) **Files I will edit** with paths, (3) **Files I will read but not edit** (interfaces, callers), (4) **Verification step** (curl, browser click, queue inspection). Keep it in chat or in `docs/plans/`. If using Claude Code, use Plan Mode (Shift+Tab twice) before any backend or AI-backend change.

## Rule 2 — Author and Maintain `CLAUDE.md`

**Why:** No `CLAUDE.md` exists today. Every AI session re-discovers the same architecture from cold, wastes 30+ tool calls, and re-asks the same questions. A single committed `CLAUDE.md` at the repo root is the highest-leverage artifact you can produce.

**How to apply:** Create `CLAUDE.md` at repo root with the exact content below, and a smaller `CLAUDE.md` in `apps/sign/`, `apps/cenvox/`, `backend/`, and `ai-backend/` for layer-specific guidance.

### Recommended root `CLAUDE.md` (author from scratch — copy/paste)

```markdown
# CLAUDE.md — SIGN + CENVOX Platform

## What this repo is
A monorepo for a construction-contract platform. Three runtimes:
- `backend/` — NestJS 10, TypeORM, Bull, Postgres, Redis. Global API prefix `api/v1`. Swagger at `/api/docs`.
- `ai-backend/` — FastAPI + Celery. Anthropic Claude + OpenAI embeddings + pgvector. Jobs are async — POST returns `{job_id}`, poll `GET /agents/jobs/{job_id}`.
- `apps/sign/` — React 18 + Vite, the user/admin/contractor SPA. i18n en/ar with RTL.
- `apps/cenvox/` — React 18 + Vite, public landing page.

## Run everything
```bash
docker compose up -d        # postgres, redis, backend, ai-backend, celery, sign, cenvox
docker compose logs -f      # tail all services
```
Health: `curl http://localhost:3000/api/v1/health` and `curl http://localhost:8000/health`.

## Seeded credentials
- `youssef141162@gmail.com` / `Youssef@1997` — `SYSTEM_ADMIN` (NOT `OWNER_ADMIN`)
- `admin@sign.com` / `Admin@Sign2026` — `SYSTEM_ADMIN`

## Conventions you must follow
- All NestJS DTOs use `class-validator`. Never bypass the global ValidationPipe.
- All AI work is async. Never block a NestJS request waiting for Claude — return job id, poll.
- All entities use snake_case columns and camelCase TS fields. Match existing entity style.
- Realtime events go through `collaboration.gateway.ts`. Don't open a second namespace.
- Bull queues: `email-queue` and `obligation-reminders` only. Add a new queue, don't piggyback.

## Things that will trip you up
- `apps/sign/src/services/api/axios.ts` has a wrong default URL fallback (`localhost:3001/api`). Always set `VITE_API_URL`.
- `apps/cenvox/src/App.tsx` hardcodes `SIGN_URL = 'http://localhost:5173'`.
- `subscriptions.service.ts` forces `dto.require_mfa = true` on plan create/update — this is intentional, do not remove.
- Admin `account-settings` route is a placeholder ComingSoon page.
- DocuSign webhook returns 200 but does nothing; do not assume envelope status flows back yet.

## How to verify your changes
- Backend: `curl` the endpoint with a JWT (login first via `POST /api/v1/auth/login`).
- AI backend: POST to `/agents/<name>`, then poll the returned job id.
- Frontend: open the relevant route in browser, check Network + Console.
- Always tail `docker compose logs -f backend ai-backend celery-worker` while testing.

## What NOT to do
- Don't write tests just to satisfy CI — the repo has none yet; coordinate with the team first.
- Don't bypass `JwtAccessGuard` or `RolesGuard`. Every protected route needs both.
- Don't store secrets in `.env.example`; use real `.env` (gitignored).
- Don't add a third frontend without discussion — two is already a lot for this team.
```

## Rule 3 — One Session Per AI Layer

**Why:** Conversations get long fast in this repo. A single session that touches NestJS, FastAPI, and React loses precision in all three. Worse, AI sessions can hit context-compaction in the middle of a refactor and forget which file they were editing.

**How to apply:** For non-trivial changes, run **one Claude session per layer**. A backend session reads only `backend/`. An AI-backend session reads only `ai-backend/`. A frontend session reads only `apps/sign/` (or `apps/cenvox/`). When a change must cross layers (e.g., new endpoint + new UI), do the backend session first, commit, then start a fresh frontend session that reads the committed contract — not the in-flight context.

## Rule 4 — Quality Gate Hooks

**Why:** There is no CI today. Local-only quality gates are the only thing standing between a typo and prod.

**How to apply:** Wire a pre-commit hook (Husky or simple shell) that runs:
- `npm run typecheck` in `backend/`, `apps/sign/`, `apps/cenvox/`
- `npm run lint` in each
- `python -m mypy app` in `ai-backend/`
- Reject any commit that introduces `console.log` in `backend/src/` or `apps/*/src/` (use `Logger` / `console.warn` only at boundaries).
- Block commits that touch `subscriptions.service.ts` MFA-forcing block without a paired update to the DTO.

## Rule 5 — Maintain `lessons.md`

**Why:** No `lessons.md` exists today. Past mistakes live only in commit messages, which nobody re-reads. Every onboarding engineer rediscovers the same potholes (the API URL typo, the seed-role mismatch, the no-op DocuSign webhook, the contractor stub).

**How to apply:** Create `docs/lessons.md` with one entry per surprising bug or design choice. Format:

```markdown
## YYYY-MM-DD — Short title
**What happened:** one or two sentences.
**Root cause:** one sentence.
**Fix / mitigation:** one sentence.
**How to avoid in future:** rule of thumb.
```

Append every time you debug something that took >30 minutes to figure out.

## Rule 6 — Use Plan Mode for Risky Work

**Why:** TypeORM migrations, queue processor changes, auth flow changes, and Celery task changes have outsized blast radius. They touch shared state (DB, Redis) that is hard to roll back.

**How to apply:** For anything in these directories, enter Claude Code Plan Mode (Shift+Tab twice) before any edit:
- `backend/src/database/migrations/`
- `backend/src/database/entities/`
- `backend/src/modules/auth/`
- `backend/src/modules/subscriptions/`
- `backend/src/modules/notifications/email-queue.processor.ts`
- `backend/src/modules/obligations/obligation-reminder.processor.ts`
- `ai-backend/app/tasks.py`
- `ai-backend/app/celery_app.py`
- `docker/init-db.sql`
- `docker-compose.yml`

Plan Mode forces you to read all callers before editing — and forces a human review before any tool runs.

---

# Part C — Priority Action List

Ordered by impact ÷ effort. Top items first.

| # | Action | Why | Effort |
|---|---|---|---|
| 1 | Author root `CLAUDE.md` (use template in Rule 2) | Highest-leverage artifact — saves every future AI session | 1–2 hr |
| 2 | Fix `apps/sign/src/services/api/axios.ts` default URL (`3001/api` → `3000/api/v1`) | Silent foot-gun for anyone running without `.env` | 5 min |
| 3 | Implement DocuSign webhook handler (envelope-completed → contract `ACTIVE`) | Webhook is a no-op; e-sign flow doesn't close the loop | 3–5 hr |
| 4 | Build a real Contractor portal (inbox + document review + sign action) | Whole persona is a stub; blocks external pilot | 1–2 weeks |
| 5 | Add `docs/lessons.md` and seed it with the 6 known potholes from this report | Permanent institutional memory | 30 min |
| 6 | Replace hardcoded `SIGN_URL` in `apps/cenvox/src/App.tsx` with env var | Will break on first non-localhost deploy | 10 min |
| 7 | Reconcile seed user role: pick `OWNER_ADMIN` or `SYSTEM_ADMIN` and update both code and onboarding docs | Documentation/code mismatch confuses every new contributor | 30 min |
| 8 | Add `@nestjs/throttler` to auth endpoints (login, register, password reset) | No rate limiting today | 1 hr |
| 9 | Add Husky pre-commit running typecheck + lint per app | First quality gate | 2 hr |
| 10 | Replace `AdminComingSoonPage` at `/admin/account-settings` with real page (or remove the route) | Dead UI in production-bound admin app | 4 hr |
| 11 | Add Celery retry/backoff config + dead-letter queue for AI tasks | OpenAI 429 today = permanent job failure | 3 hr |
| 12 | Wire S3 storage code path so `STORAGE_DRIVER=s3` swaps from local disk | Production needs object storage | 4 hr |
| 13 | Add a single Vitest smoke test per critical SIGN page | Establish testing precedent | 1 day |
| 14 | Add a single Jest spec for `ContractsService.requestApproval` | Establish backend testing precedent | 4 hr |
| 15 | Add a `.github/workflows/ci.yml` running typecheck + lint on PR | First CI | 2 hr |
| 16 | Document the contract status state-machine in `docs/contract-state-machine.md` | 12-state machine has no diagram anywhere | 2 hr |
| 17 | Add observability: Prometheus exporter on NestJS + Celery, Grafana dashboard | Admin observability page is currently mocked | 1–2 days |
| 18 | Refresh-token rotation + revocation list in Redis (instead of in-memory) | Sessions survive restarts but revocations don't | 4 hr |

---

# Appendix — Full File Inventory (high-signal)

### Root
- `docker-compose.yml` — 7 services (postgres, redis, backend, ai-backend, celery-worker, frontend, cenvox)
- `package.json` — npm workspaces (`apps/*`, `packages/*`)
- `docker/init-db.sql` — extensions + enum types

### `backend/`
- `src/main.ts` — bootstrap (helmet, CORS, ValidationPipe, Swagger)
- `src/app.module.ts` — 32 modules + Health + TypeORM + Bull
- `src/database/entities/*` — 33 entities
- `src/database/migrations/*` — 21 migrations
- `src/database/seeds/seed-default-users.ts` — youssef + admin@sign.com (both SYSTEM_ADMIN)
- `src/modules/auth/auth.controller.ts` — 16 endpoints
- `src/modules/auth/auth.service.ts` — JWT issue/refresh, MFA verify, recovery codes
- `src/modules/contracts/contracts.controller.ts` — 27 endpoints
- `src/modules/contracts/contracts.service.ts` — state machine enforcement
- `src/modules/ai/ai.service.ts` — proxies to FastAPI
- `src/modules/subscriptions/subscriptions.service.ts` — Paymob + HMAC verify (forces MFA on every plan)
- `src/modules/notifications/notifications.module.ts` — wires email queue + dispatch
- `src/modules/notifications/email-queue.processor.ts` — Bull `email-queue`
- `src/modules/notifications/email.service.ts` — SES (prod) / nodemailer (dev)
- `src/modules/obligations/obligation-reminder.processor.ts` — Bull `obligation-reminders`
- `src/modules/collaboration/collaboration.gateway.ts` — Socket.IO `/collaboration`
- `src/modules/storage/storage.service.ts` — local disk + UTF-8 filename fix
- `src/modules/docusign/docusign.service.ts` — JWT Grant
- `src/modules/docusign/docusign-webhook.controller.ts` — no-op 200

### `ai-backend/`
- `main.py` — FastAPI app, mounts agents + embeddings
- `app/celery_app.py` — Celery config
- `app/tasks.py` — 9 tasks
- `app/routers/agents.py` — 11 routes
- `app/routers/embeddings.py` — ingest + search
- `app/agents/*.py` — Claude prompts per agent

### `apps/sign/`
- `src/App.tsx` — 250-line router (public + /app + /admin + /contractor)
- `src/services/api/axios.ts` — JWT interceptor + refresh queue (⚠️ wrong default URL)
- `src/services/api/*.ts` — 27 service files
- `src/i18n/index.ts` — i18next + RTL flip
- `src/pages/contractor/ContractorDashboardPage.tsx` — STUB
- `src/pages/admin/AdminComingSoonPage.tsx` — placeholder used by `/admin/account-settings`

### `apps/cenvox/`
- `src/App.tsx` — 1041 lines, single-file landing, hardcoded `localhost:5173` SIGN link

### Env templates
- `backend/.env.example`
- `ai-backend/.env.example`
- `apps/sign/.env.example`

### Conspicuously missing
- `CLAUDE.md` (any level)
- `docs/lessons.md`
- `.github/workflows/`
- Any `*.spec.ts` / `*.test.ts` / `pytest` / `vitest.config.ts`

---

## PART D: API INTEGRATION READINESS

> We are NOT in the deployment phase. This section documents which features require real API credentials to function.
> These features may be fully coded but will NOT work until deployment phase provides real credentials.

---

### D1 — Integration Master Table

| Integration | Purpose in SIGN | Features That Depend On It | Code Status | Env Vars Required | Works Without Credentials? |
|-------------|----------------|---------------------------|-------------|-------------------|---------------------------|
| **Anthropic API** | All AI analysis, chat, agents | Risk analysis, summarization, compliance, obligations, conflict detection, diff, chat, research, OCR text extract, clause extract | ✅ Fully wired in `ai-backend/app/agents/*` and `ai-backend/app/tasks.py` (9 Celery tasks) | `ANTHROPIC_API_KEY` (in `ai-backend/.env.example`) | ❌ No — every AI feature returns 5xx / fails the Celery job |
| **OpenAI Embeddings** | Vector embeddings for RAG over the knowledge base | AI chat citations, knowledge-asset search, contextual retrieval | ✅ Wired in `ai-backend` embeddings router; uses `text-embedding-3-small` | `OPENAI_API_KEY` (in `ai-backend/.env.example`) | ❌ No — `/embeddings/ingest` and `/embeddings/search` fail |
| **DocuSign** | E-signature for contract execution | Contract signing, FULLY_EXECUTED status, envelope tracking, completion / decline / void notifications | ✅ Now fully implemented (this session): JWT Grant auth, envelope create, embedded signing URL, **HMAC-verified webhook handler with completed/declined/voided branches + audit log + owner notification** | `DOCUSIGN_INTEGRATION_KEY` `DOCUSIGN_USER_ID` `DOCUSIGN_ACCOUNT_ID` `DOCUSIGN_RSA_PRIVATE_KEY` `DOCUSIGN_AUTH_SERVER` `DOCUSIGN_BASE_PATH` `DOCUSIGN_WEBHOOK_HMAC_SECRET` (⚠️ none of these are in `backend/.env.example` yet — must be added before deploy) | ❌ No — initiateSignature throws, webhook rejects with 401 |
| **Paymob** (EG payments) | Payment processing | Subscription activation, plan upgrades, store purchases, renewals, HMAC-verified webhook → activate subscription + insert PaymentTransaction row | ✅ Fully wired in `subscriptions.service.ts` (auth → order → payment_keys → SHA-512 HMAC webhook). Dev fallback returns mock keys when key absent | `PAYMOB_API_KEY` `PAYMOB_INTEGRATION_ID` `PAYMOB_IFRAME_ID` `PAYMOB_HMAC_SECRET` (all 4 in `backend/.env.example`) | ⚠️ Partial — code returns mock `payment_key` for dev; real payments + webhook activation require keys |
| **AWS S3** | Production object storage for uploaded files | Contract uploads (Word/PDF), knowledge assets, claim attachments, notice attachments, executed contract archives | ⚠️ SDK imported (`@aws-sdk/client-s3`) but `StorageService` always uses local disk; `STORAGE_TYPE=s3` switch not yet wired | `AWS_REGION` `AWS_ACCESS_KEY_ID` `AWS_SECRET_ACCESS_KEY` `AWS_S3_BUCKET` `STORAGE_TYPE=s3` (all in `backend/.env.example`) | ✅ Yes for local dev (writes to `./uploads/`) — ❌ No for production-grade durability |
| **AWS SES** (prod email) | Transactional email in production | All transactional emails when `NODE_ENV=production` | ✅ Code path exists in `EmailService` (uses `@aws-sdk/client-ses` `SendEmailCommand`) | `AWS_SES_FROM_EMAIL` + the AWS_* credentials above | ❌ No in production — falls back to nodemailer in dev |
| **SMTP (nodemailer)** | Dev / staging email | Same recipients as SES (invitations, MFA OTP, reminders, contract sharing, password reset, DocuSign notifications) | ✅ Wired via nodemailer; queued through Bull `email-queue` | `SMTP_HOST` `SMTP_PORT` `SMTP_USER` `SMTP_PASS` `SMTP_FROM` (all in `backend/.env.example`, defaults to Mailtrap) | ⚠️ Partial — Mailtrap host defaulted but credentials blank → emails silently fail without `SMTP_USER`/`SMTP_PASS` |
| **Redis** | Bull job broker (NestJS) + Celery broker & result backend (FastAPI) | All async AI jobs, all queued emails, all obligation reminder scans, all subscription session caches | ✅ Configured via `BullModule.forRootAsync` in NestJS and Celery config in `ai-backend` | `REDIS_URL` (in both `backend/.env.example` and `ai-backend/.env.example`) | ❌ No — service won't boot without Redis (default `redis://localhost:6379`) |
| **PostgreSQL + pgvector** | Primary data store + vector search for RAG | Every persisted entity (33 entities, 21 migrations) + `KnowledgeAssetChunk.embedding` vector column | ✅ Active — `pgvector/pgvector:pg15` image, `init-db.sql` enables `vector` + `uuid-ossp` extensions | `DATABASE_URL` (in both backend and ai-backend env templates) | ❌ No — service won't boot |
| **Internal NestJS ↔ FastAPI token** | Authenticate internal AI requests | NestJS posting jobs to FastAPI; FastAPI calling back into NestJS for ingestion | ✅ Used in headers between services | `NESTJS_INTERNAL_TOKEN` (in `ai-backend/.env.example`) | ⚠️ Default value works locally; must rotate before any non-local deploy |
| **JWT Secrets** | Access + refresh token signing | Every authenticated request | ✅ Hard-required in `AuthService` | `JWT_SECRET` `JWT_REFRESH_SECRET` `JWT_EXPIRY` `JWT_REFRESH_EXPIRY` (in `backend/.env.example`) | ⚠️ Defaults present but must be rotated for production |

---

### D2 — Features Grouped by Integration Dependency

#### Needs Anthropic API
- Risk analysis across all 8 risk categories per clause (`run_risk_analysis`)
- Contract summarization, 17 key elements (`run_summarize`)
- Compliance check against FIDIC/NEC/JCT standards (compliance prompt in agents)
- Obligation extraction from contract text (`run_extract_obligations`)
- Conflict / contradiction detection across clauses (`run_conflict_detection`)
- DIFF analysis between contract versions (`run_diff_analysis`)
- AI chat assistant with RAG (`run_chat`)
- AI Research Agent (`run_research`)
- OCR text extract from uploaded PDFs (`run_extract_text`, 30-min limit, Arabic-friendly)
- Clause extraction from raw uploaded contracts (`run_extract_clauses`, 40-min limit)

#### Needs OpenAI API
- Knowledge base embedding generation (`POST /embeddings/ingest`)
- Knowledge base vector search (`POST /embeddings/search`)
- AI chat retrieval with knowledge-base citations (uses search internally)

#### Needs DocuSign API
- Initiate envelope from contract PDF (`DocuSignService.createEnvelope`)
- Generate embedded signing URL for the initiator (`getSigningUrl`)
- Live envelope status query (`getEnvelopeStatus`)
- Webhook → `completed` → contract `FULLY_EXECUTED` + `executed_at` + audit log + owner notification
- Webhook → `declined` → contract reverted to `ACTIVE` + decline reason logged in audit + owner notification
- Webhook → `voided` → contract reverted to `ACTIVE` + void reason logged in audit + owner notification
- Webhook → `sent` / `delivered` → per-signer status sync only

#### Needs Paymob API
- Create payment intention for plan purchase (`SubscriptionsService.createPaymentIntention`)
- Iframe-embedded checkout for the user
- HMAC-verified webhook → `activateSubscription` + `PaymentTransaction` insert (success / pending / refunded / failed)
- Subscription expiry / renewal flow (relies on the same webhook surface)

#### Needs AWS S3
- Upload contract documents (Word, PDF) to durable storage
- Upload knowledge assets and chunked source files
- Upload claim supporting documents
- Upload notice attachments
- Archive executed contract PDFs (post-DocuSign)

#### Needs Email Service (SES in prod, SMTP in dev)
- Team / org invitation email
- MFA OTP delivery via email
- Password reset email
- Contract approval-request email
- Contract sharing link email
- Obligation deadline reminder email (sent by `obligation-reminders` Bull processor)
- DocuSign envelope state-change notification (added this session)
- Generic system notifications (via `EmailService.sendGenericEmail`)

#### Needs Redis
- Every async AI job (NestJS publishes job, Celery consumes, result polled)
- Bull `email-queue` (transactional email)
- Bull `obligation-reminders` (scheduled scanner)
- Celery result backend for `GET /agents/jobs/{job_id}` polling

#### Needs PostgreSQL + pgvector
- Every persisted entity (auth, orgs, projects, contracts, clauses, comments, versions, approvals, knowledge assets, obligations, notifications, subscriptions, payments, audit, MFA, recovery codes, sessions)
- pgvector cosine search on `KnowledgeAssetChunk.embedding`

#### Works Right Now — No External API Needed
- Full authentication (register, login, MFA TOTP enroll/verify, recovery codes, password reset)
- Invitation creation (creation works; email delivery requires SMTP/SES)
- Organization and team management (CRUD, role assignment)
- Project and contractor management
- Contract creation, clause editing, parties, comments, version save / compare / list
- Approval workflow (request + review)
- Claims, Notices, Sub-contract submission and tracking (when contract status is ACTIVE)
- Obligations dashboard (manual entry + Bull-powered reminder cron — reminder scan runs without external APIs but its email step needs SMTP/SES)
- Admin portal (organizations, users, plan CRUD, knowledge curation, audit log, observability page)
- Audit trail viewing
- CENVOX landing page (all 11 sections — fully static)

---

### D3 — Deployment Readiness Checklist

> To be completed when deployment phase begins. Do not action any of these now.

| # | Item | Category | Status |
|---|------|----------|--------|
| 1 | Anthropic API key configured and tested | AI | ❌ Not configured |
| 2 | OpenAI API key configured and tested (embeddings) | AI | ❌ Not configured |
| 3 | DocuSign integration key configured | E-Signature | ❌ Not configured |
| 4 | DocuSign user ID + RSA private key configured (JWT Grant) | E-Signature | ❌ Not configured |
| 5 | DocuSign account ID configured | E-Signature | ❌ Not configured |
| 6 | DocuSign webhook HMAC secret configured (`DOCUSIGN_WEBHOOK_HMAC_SECRET`) | E-Signature | ❌ Not configured |
| 7 | DocuSign webhook URL registered in DocuSign Connect dashboard | E-Signature | ❌ Not registered |
| 8 | DocuSign webhook handler fixed (was no-op — fixed in this session: HMAC verify + completed/declined/voided + audit + notify) | E-Signature | ✅ Code fixed |
| 9 | DocuSign env vars added to `backend/.env.example` (currently missing) | E-Signature | ❌ Not added |
| 10 | Paymob API key, integration ID, iframe ID, HMAC secret configured | Payments | ❌ Not configured |
| 11 | Paymob webhook URL registered in Paymob dashboard | Payments | ❌ Not registered |
| 12 | AWS S3 bucket created with correct CORS policy | Storage | ❌ Not configured |
| 13 | AWS IAM credentials with S3 read/write scope | Storage | ❌ Not configured |
| 14 | `STORAGE_TYPE=s3` switch wired in `StorageService` (currently always local) | Storage | ❌ Code change needed |
| 15 | AWS SES verified sender + production-mode access approved | Email | ❌ Not configured |
| 16 | SMTP credentials (Mailtrap or alternative) for dev / staging | Email | ❌ Not configured |
| 17 | All email templates rendered and tested end-to-end | Email | ❌ Not verified |
| 18 | Redis running in production environment with persistence | Queue | ❌ Not configured |
| 19 | PostgreSQL with pgvector extension enabled in production | Database | ❌ Not confirmed |
| 20 | All env vars documented in `.env.example` (DocuSign vars currently missing) | Docs | ❌ Verify and add |
| 21 | JWT secrets rotated to production values | Security | ❌ Not done |
| 22 | `NESTJS_INTERNAL_TOKEN` rotated to production value | Security | ❌ Not done |
| 23 | HTTPS / SSL configured on all endpoints | Security | ❌ Not done |
| 24 | CORS origins updated from localhost to production domains | Security | ❌ Not done |
| 25 | Rate limiting (`@nestjs/throttler`) added to auth endpoints | Security | ❌ Not done |
| 26 | cenvox.ai DNS configured | Domain | ❌ Not done |
| 27 | sign.ai DNS configured | Domain | ❌ Not done |
| 28 | Hardcoded `SIGN_URL = http://localhost:5173` in `apps/cenvox/src/App.tsx` replaced with env var | Domain | ❌ Not done |

---

### D4 — Local Development Workarounds

| Integration | Free Local Workaround | Notes |
|-------------|----------------------|-------|
| DocuSign | Free sandbox at developers.docusign.com | Get test integration key + RSA key — full envelope flow works in sandbox; webhook can be tunneled with ngrok / cloudflared |
| Paymob | Test mode API keys from Paymob dashboard | Test cards available — no real money. Code already returns mock `payment_key` when `PAYMOB_API_KEY` is unset |
| AWS S3 | MinIO via Docker (local S3 emulator) | Drop-in S3 replacement — set `AWS_S3_ENDPOINT` to MinIO URL once `STORAGE_TYPE=s3` switch is wired |
| AWS SES | Mailtrap.io (free) | Catches all outbound emails — already the default `SMTP_HOST` in `backend/.env.example`; just add `SMTP_USER` + `SMTP_PASS` from your Mailtrap inbox |
| Anthropic Claude | Real API key required | No free alternative — get key from console.anthropic.com |
| OpenAI Embeddings | Real API key required | No free alternative — get key from platform.openai.com |
| Redis | `docker compose up redis` | Already in compose file — no config needed |
| Postgres + pgvector | `docker compose up postgres` | Already in compose file with pgvector image |

---

*End of report — generated 2026-04-25, Part D appended same day*
