# Changelog

All rule changes are documented here.
Format: [version] — date — what changed and why.

---

## [1.1.1] — 2026-03-26

### Added

- `releaseKey()` on handler failure rule — idempotencyMiddleware and
  webhookDeduplicationGuard must now call `releaseKey(storeKey)` inside
  a catch block if the handler fails after claiming the key. Prevents
  blocked retries and silent data loss on partial failures.

- `ADDING A NEW FEATURE MODULE` section (Section 5) — exact file
  structure, service rules, router rules, DTO rules, and policy rules
  for adding a new backend module safely. AI now has a canonical pattern
  to follow instead of inventing its own structure.

- `EXTENDING THE BACKEND` section (Section 14) — step-by-step patterns
  for the five most common extension scenarios:
  - Adding a new billing plan or pricing tier
  - Adding a new LLM-powered feature
  - Adding a new background job type
  - Adding a new external integration (e.g. SendGrid, Slack)
  - Adding multi-tenancy to a new module

- `FRONTEND INTEGRATION` section (Section 13) — covers:
  - Hard boundary: frontend never imports backend src/ modules
  - Environment variables (NEXT_PUBLIC_ prefix rules, never expose secrets)
  - Authentication: Bearer token pattern, accessToken in memory,
    refreshToken in httpOnly cookie only
  - Idempotency keys: crypto.randomUUID() generation pattern per action
  - Async jobs: 202 + poll pattern with TanStack Query example
  - LLM budget display: GET /api/v1/llm/budget polling pattern
  - CORS: .env-only config, never CORS_ORIGIN=* in production
  - Framework-agnostic: Next.js, React SPA, Vue, SvelteKit, Flutter
    all supported with same header patterns

- `OBSERVABILITY AND DEPLOYMENT` section (Section 15) — structured
  logging rules, Sentry/OpenTelemetry hook guidance, Docker + deployment
  targets (Railway, Fly.io, Render, DigitalOcean, own VPS).

- `ADDING A NEW API ENDPOINT` full checklist (Section 12) — 7-step
  process from DTO definition to E2E test, ensuring no step is skipped.

- `DATABASE RULES` section (Section 8) — upsert preference, transaction
  guidance for multi-step operations, in-memory mode transparency rule.

- `SECURITY RULES` section (Section 9) — dependency pinning, npm audit
  requirement, CSRF guidance, parameterized queries only, egress filtering,
  token storage rules.

- `CANONICAL REFERENCE FILES` section (Section 18) — AI now has a
  single lookup table of authoritative reference files for every major
  pattern in the codebase.

- `// @vibe-safe: HUMAN-CROSS-REVIEW` comment pattern — required on
  any change touching a human-only layer (01-Core, 03-Policies,
  05-Infra, 06-Background retry logic, 07-AI).

- `// @vibe-safe: NO-IS-PROCESSED` comment pattern — required when
  refactoring old isProcessed() + markProcessed() code to tryClaimKey().

- `// @vibe-safe: REVIEW-NON-HUMAN` comment pattern — required when
  weakened retry/dead-letter logic is found in background workers.

- Pre-commit checklist now split into two sections:
  Backend checklist (16 items) and Frontend integration checklist
  (10 items) — both must pass before any commit.

### Changed

- `06-Background` layer write permissions now explicitly clarifies:
  RetryableJobRunner.ts and WorkerPool.ts are ❌ human-review-only,
  even though the layer is otherwise AI-writable.

- `LLM / AI FEATURE RULES` now requires explicit default model
  declaration in new code: `model: '4o-mini'` or `model: 'haiku'`.
  Upgrade requires a comment explaining why complex reasoning is needed.

- `isProcessed() + markProcessed()` rule strengthened — now includes
  an explicit refactor instruction and required @vibe-safe comment
  when the old pattern is found in existing code.

- `MINIMAL CHANGE PRINCIPLE` now explicitly includes a post-change
  review checklist: race conditions, transactional safety, idempotency,
  layer boundary violations, missing tests.

- `WHO YOU ARE` section (Section 0) expanded to include frontend
  integration responsibilities and "never invent infrastructure" rule.

### Why these changes

v1.1.1 addresses three gaps discovered during real-world engine use:

1. The releaseKey() gap — idempotency and webhook dedup middlewares
   could block retries silently on handler failure. Now fixed and
   encoded as a rule so AI never omits it in new middleware code.

2. The extension gap — users extending the backend had no canonical
   pattern for new modules, jobs, integrations, or multi-tenancy.
   Sections 5, 12, and 14 close this with exact step-by-step guides.

3. The frontend gap — users connecting any frontend (Next.js, React,
   Vue, Flutter) had no guidance in the rules file. Section 13 closes
   this with auth, idempotency, 202+poll, CORS, and budget patterns.

---

## [1.1.0] — 2026-03-21

### Added
- `IDEMPOTENCY AND WEBHOOK DEDUP` section with explicit rule:
  never use `isProcessed()` + `markProcessed()` together —
  confirmed race condition under concurrent Stripe retries.
- `BACKGROUND JOBS` section clarifying when to use `RetryableJobRunner`
  vs raw `QueueWorker` (critical vs fire-and-forget).
- `AI FEATURE RULES` section requiring all LLM calls to go through
  `llmClient` singleton with a `feature` field for cost attribution.
- `FAILURE SAFETY CHECKLIST` — AI must verify before submitting
  any PR touching mutating routes, webhooks, jobs, or LLM calls.
- Layer import rules explicitly documented to prevent circular
  dependencies and reverse-direction imports.

### Changed
- Clarified that `04-Modules` legitimately imports from `05-Infra`
  and `06-Background` — layer order is NOT strictly sequential.
- `LLM_PROVIDER=stub` documented as the required local dev default.

### Why these changes
v1.1 added the Failure Safety layer (idempotency middleware, webhook
deduplication guard, retry-safe job runner) and the Centralized LLM
Client. These rules encode the constraints that make those features
work correctly when AI is writing feature code on top of them.

---

## [1.0.0] — 2026-02-10

### Initial release

- Layer write permissions table (01–08)
- Hard rules: no direct process.env, no Stripe/Redis/DB outside Infra,
  no schema changes without migrations
- Coding rules: TypeScript strict mode, conventional commits,
  HTTP status code conventions (200/201/202)
- OpenAPI annotation requirement for all new routes
