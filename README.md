# KeelStack .cursorrules

AI agent rules for the [KeelStack Engine](https://keelstack.me) —
a production-grade Node.js + TypeScript backend with strict 8-layer architecture.

These rules exist because AI agents (Cursor, Claude, Copilot) write better,
safer code when they know exactly what they can and cannot touch.

Without rules, AI will:
- Instantiate Stripe clients wherever it feels like
- Read process.env directly instead of going through a config layer
- Bypass policy guards in modules
- Skip idempotency on mutating routes
- Call OpenAI directly instead of through a cost-controlled client

These rules prevent all of that.

---

## How to use

Copy `.cursorrules` into the root of your KeelStack Engine project:

```bash
curl -o .cursorrules \
  https://raw.githubusercontent.com/KeelStack-me/keelstack-cursorrules/main/.cursorrules
```

Or clone and copy manually.

---

## What it enforces

### Layer write permissions

| Layer | AI Write? | Why |
|---|---|---|
| 01-Core | ❌ NO | Security-critical. Errors, middleware, guards. Human review only. |
| 02-Common | ✅ YES | DTOs, types, utilities. Low risk. |
| 03-Policies | ❌ NO | Business rules. Wrong policy = revenue leak or data loss. |
| 04-Modules | ✅ YES | Feature modules. Auth/billing changes need human review. |
| 05-Infra | ❌ NO | DB schema + gateways. Schema changes require migrations. |
| 06-Background | ✅ YES | Adding job types is safe. WorkerPool changes need human review. |
| 07-AI | ❌ NO | LLMClient controls costs. AI must not modify its own constraints. |
| 08-Web | ✅ YES | Routes are safe to add. Must use the full middleware chain. |

### Hard rules (short version)

- Never modify `src/01-Core` without explicit human instruction
- Never bypass `src/03-Policies` checks
- Never read `process.env` directly — use `RuntimeConfig.ts`
- Never instantiate Stripe, Redis, or DB clients outside `src/05-Infra`
- Never call LLM APIs directly — always use the `llmClient` singleton
- Never add a POST/PUT/PATCH route without considering idempotency
- Never process webhooks without `webhookDeduplicationGuard`

### Failure safety checklist

Before any PR, AI is instructed to verify:

- [ ] Mutating routes use `idempotencyMiddleware`
- [ ] Webhook routes use `webhookDeduplicationGuard` — never `isProcessed()` + `markProcessed()` (confirmed race condition)
- [ ] Critical jobs use `RetryableJobRunner`, not raw `QueueWorker`
- [ ] All LLM calls go through `llmClient` with a `feature` field
- [ ] No new `process.env` reads outside `RuntimeConfig.ts`
- [ ] No new DB / Redis / Stripe instantiation outside `05-Infra`
- [ ] New routes have `@openapi` annotations

---

## Why these rules exist (the honest version)

I built KeelStack Engine after repeatedly rebuilding the same backend
foundation — auth, Stripe, background jobs, AI cost controls — across
different projects. Every time, the hardest bugs were in production:

- Duplicate Stripe charges from double-click submits
- Webhooks double-processing on Stripe retries
- Background jobs vanishing silently on worker crash
- One user burning the entire OpenAI budget in an afternoon

These `.cursorrules` encode what I learned. They tell the AI to stay
inside safe boundaries so you can move fast without breaking production.

---

## What these rules are NOT

**These are architectural guardrails, not a security boundary.**

Real security lives in the code:
- Argon2id password hashing (OWASP 2023 parameters)
- Timing-safe comparisons
- Idempotency via atomic `SET NX` (Redis in prod, in-memory in dev)
- Webhook signature verification
- 597 unit tests, 93.13% coverage, CI-enforced

Always review AI-generated code before merging.
Never trust `.cursorrules` alone to prevent a security issue.

---

## Full engine

These rules ship bundled with [KeelStack Engine](https://keelstack.me):

- Production Node.js + TypeScript backend
- Auth, Stripe billing, background jobs, AI cost controls — wired and tested
- 597 unit tests · 93.13% coverage · 0 known vulnerabilities
- One-time purchase · $29 early access only

---

## License

MIT — copy, adapt, use freely.

If you use these rules in your own project, you don't need to credit KeelStack.
But if they save you from a production incident, I'd love to hear about it.
