# KeelStack .cursorrules

AI agent rules for the [KeelStack Engine](https://keelstack.me) — a production-grade Node.js + TypeScript backend with strict 8-layer architecture, failure safety patterns, and full-stack extension support.

These rules exist because AI agents (Cursor, Claude, Copilot) write better, safer code when they know exactly what they can and cannot touch.

Without rules, AI will:
- Instantiate Stripe clients wherever it feels like
- Read process.env directly instead of going through a config layer
- Bypass policy guards in modules
- Skip idempotency on mutating routes
- Call OpenAI directly instead of through a cost-controlled client
- Wire a frontend to the backend incorrectly (missing auth headers,
  no idempotency keys, wrong job-polling pattern)

These rules prevent all of that.

---

## How to use

Copy `.cursorrules` into the root of your KeelStack Engine project:

```bash
curl -o .cursorrules \
  https://raw.githubusercontent.com/KeelStack-me/keelstack-cursorrules/main/.cursorrules
```

Or clone and copy manually:

```bash
git clone https://github.com/KeelStack-me/keelstack-cursorrules.git
cp keelstack-cursorrules/.cursorrules your-project/.cursorrules
```

Cursor auto-loads `.cursorrules` from the project root on next open.

---

## What it enforces

### Layer write permissions

| Layer         | AI Write? | Why                                                              |
|---------------|-----------|------------------------------------------------------------------|
| 01-Core       | ❌ NO     | Security-critical. Middleware, guards, errors. Human only.       |
| 02-Common     | ✅ YES    | DTOs, types, utilities. Low risk.                                |
| 03-Policies   | ❌ NO     | Business rules. Wrong policy = revenue leak or data loss.        |
| 04-Modules    | ✅ YES    | Feature modules. Auth/billing changes need human review.         |
| 05-Infra      | ❌ NO     | DB schema + gateways. Schema changes require migrations.         |
| 06-Background | ✅ YES    | New job types safe. RetryableJobRunner = human review only.      |
| 07-AI         | ❌ NO     | LLMClient controls costs. AI must not modify its own limits.     |
| 08-Web        | ✅ YES    | Routes safe to add. Must use full middleware chain.              |

Any change touching a ❌ layer requires a
`// @vibe-safe: HUMAN-CROSS-REVIEW` comment in the file and PR.

---

### Hard rules (short version)

- Never modify `src/01-Core` without explicit human instruction
- Never bypass `src/03-Policies` checks
- Never read `process.env` directly — use `RuntimeConfig.ts`
- Never instantiate Stripe, Redis, or DB clients outside `src/05-Infra`
- Never call LLM APIs directly — always use the `llmClient` singleton
- Never add a POST/PUT/PATCH route without considering idempotency
- Never process webhooks without `webhookDeduplicationGuard`
- Never use `isProcessed()` + `markProcessed()` — race condition confirmed
- Never delete `// @vibe-safe` comments
- Never commit secrets, API keys, or credentials to source

---

### Idempotency and webhook dedup

Two mechanisms — use the correct one per context:

```ts
// Webhook routes
router.post('/webhooks/:provider',
  webhookDeduplicationGuard(store, 'stripe'), handler);

// Mutating API routes
router.post('/resource',
  idempotencyMiddleware({ store, namespace: 'billing.sub' }), handler);
```

Both use `tryClaimKey()` (atomic SET NX). If the handler fails after claiming the key, `releaseKey()` must be called before throwing — this prevents blocked retries and silent data loss.

---

### Background jobs

```ts
// Critical paths — use RetryableJobRunner
// Fire-and-forget only — use raw QueueWorker

// Handler contract
return { ok: true };                          // success
return { ok: false, errorMessage: 'reason' }; // retryable failure
throw new NonRetryableJobError('bad input');  // must not retry
```

All job handlers must be idempotent — repeated execution must produce no duplicate side effects.

---

### LLM cost control

```ts
// All LLM calls via llmClient singleton — never directly
await llmClient.complete({
  model: '4o-mini',          // cheapest model by default
  feature: 'report_gen',     // required for cost attribution
  messages: [...],
});
```

Set `LLM_PROVIDER=stub` in `.env` for local dev — no API key needed.

---

### Frontend integration

The rules cover full frontend wiring for any framework
(Next.js, React SPA, Vue, SvelteKit, Flutter):

**Auth:**
```ts
// Login → { accessToken, refreshToken }
// accessToken → memory only (never localStorage)
// refreshToken → httpOnly + Secure cookie
// Attach: Authorization: Bearer <accessToken>
```

**Idempotency keys (per user action):**
```ts
const key = `${tenantId}-${feature}-${crypto.randomUUID()}`;
// 'Idempotency-Key': key  on every POST/PUT/PATCH
```

**Async jobs (202 + poll):**
```ts
// POST /api/v1/tasks → 202 { jobId, pollUrl }
// GET {pollUrl} → { status: 'queued'|'processing'|'done'|'failed' }
// Poll every 1.5s until terminal state. Cap at 60 attempts.
```

**LLM budget display:**
```ts
// GET /api/v1/llm/budget → { tokensUsedThisHour, budgetPerHour }
// Show as tokens or % — never raw API cost
```

Reference implementation:
[keelstack-ui-starter](https://github.com/KeelStack-me/keelstack-ui-starter)
(Next.js 14 · TypeScript · TanStack Query · Tailwind · Axios)

---

### Pre-commit checklist (backend)

- [ ] Mutating routes use `idempotencyMiddleware`
- [ ] `releaseKey()` called on handler failure after key claim
- [ ] Webhook routes use `webhookDeduplicationGuard` — never `isProcessed()` + `markProcessed()`
- [ ] Critical jobs use `RetryableJobRunner`, not `QueueWorker`
- [ ] All LLM calls via `llmClient` with `feature` field + default model set
- [ ] No `process.env` reads outside `RuntimeConfig.ts`
- [ ] No DB / Redis / Stripe instantiation outside `05-Infra`
- [ ] `schema.ts` changes have matching migration
- [ ] New routes have `@openapi` annotations
- [ ] No reverse-direction layer imports
- [ ] All unit + E2E tests pass
- [ ] No secrets committed to source

### Pre-commit checklist (frontend integration)

- [ ] No backend `src/` files imported into frontend
- [ ] API base URL from env var, not hardcoded
- [ ] `accessToken` in memory, `refreshToken` in httpOnly cookie
- [ ] `Idempotency-Key` on all POST/PUT/PATCH requests
- [ ] 202 responses trigger poll loop with max retry cap
- [ ] `failed` job state handled with retry option in UI
- [ ] `CORS_ORIGIN` set in `.env`, not hardcoded in code
- [ ] No secrets in `NEXT_PUBLIC_` env vars

---

## Why these rules exist (the honest version)

I built KeelStack Engine after repeatedly rebuilding the same backend foundation — auth, Stripe, background jobs, AI cost controls — across different projects. Every time, the hardest bugs were in production:

- Duplicate Stripe charges from double-click submits
- Webhooks double-processing on Stripe retries
- Background jobs vanishing silently on worker crash
- One user burning the entire OpenAI budget in an afternoon

These `.cursorrules` encode what I learned. They tell the AI to stay inside safe boundaries so you can move fast without breaking production.

---

## What these rules are NOT

**These are architectural guardrails, not a security boundary.**

Real security lives in the code:
- Argon2id password hashing (OWASP 2023 parameters)
- Timing-safe comparisons
- Idempotency via atomic `SET NX` (Redis in prod, in-memory in dev)
- Webhook signature verification
- 597 unit tests · 93.13% coverage · CI-enforced at 90% threshold

Always review AI-generated code before merging.
Never trust `.cursorrules` alone to prevent a security issue.

---

## Full engine

These rules ship bundled with [KeelStack Engine](https://keelstack.me):

- Production Node.js + TypeScript + Express backend
- Auth, Stripe billing, background jobs, LLM cost controls
  — all wired, tested, and failure-safe
- 597 unit tests · 93.13% coverage · 0 known vulnerabilities
- 15 AI prompts for Cursor, Claude, and Copilot
- One-time purchase · $29 early builder price

---

## License

MIT — copy, adapt, use freely.

If you use these rules in your own project, you don't need to credit KeelStack. But if they save you from a production incident, I'd love to hear about it.


Both files are now **fully aligned with v1.1.1 `.cursorrules`** — every new section (frontend, extension patterns, release key fix, security, observability, deployment) is reflected accurately in the changelog with clear "why" reasoning, and the README now serves as a **complete, scannable reference** for both backend and frontend users.
