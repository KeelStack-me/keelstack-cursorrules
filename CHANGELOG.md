# Changelog

All rule changes are documented here.
Format: [version] — date — what changed and why.

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
- `FAILURE SAFETY CHECKLIST` — AI must verify this before submitting
  any PR that touches mutating routes, webhooks, jobs, or LLM calls.
- Layer import rules now explicitly documented (which layers may import
  from which) to prevent circular dependencies and reverse-direction imports.

### Changed
- Clarified that `04-Modules` legitimately imports from `05-Infra` and
  `06-Background` — the layer order is NOT strictly sequential.
- `LLM_PROVIDER=stub` now documented as the required local dev default
  so AI doesn't generate code that assumes a real API key is present.

### Why these changes
v1.1 added the Failure Safety layer (idempotency middleware, webhook
deduplication guard, retry-safe job runner) and the Centralized LLM Client.
These rules encode the constraints that make those features work correctly
when AI is writing feature code on top of them.

---

## [1.0.0] — 2026-02-10

### Initial release

- Layer write permissions table (01–08)
- Hard rules: no direct process.env, no Stripe/Redis/DB outside Infra,
  no schema changes without migrations
- Coding rules: TypeScript strict mode, conventional commits,
  HTTP status code conventions (200/201/202)
- OpenAPI annotation requirement for all new routes
