# KeelStack .cursorrules

AI agent rules for the [KeelStack Engine](https://keelstack.me) —
a production-grade Node.js + TypeScript backend with strict 8-layer
architecture, failure safety patterns, and full-stack extension support.

These rules exist because AI agents (Cursor, Claude, Copilot) write
better, safer code when they know exactly what they can and cannot touch.

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
