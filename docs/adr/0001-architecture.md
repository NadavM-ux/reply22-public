# ADR 0001: Monorepo, clean architecture, Postgres + queues

**Status:** accepted · 2026-11-04

## Context
Comment automations are webhook driven and bursty. Meta expects a fast `200`, retries on failure,
caps automated DMs per account, and a Private Reply may be sent only once per comment.
Publishing is a slow multi-step workflow (upload → platform processing → publish).

## Decisions
1. **TypeScript monorepo** (pnpm + Turborepo). Shared Zod contracts mean one source of truth for validation.
2. **Clean architecture.** `@app/domain` is pure (no framework imports, enforced by ESLint). Use cases depend on
   ports (interfaces / abstract classes). Drizzle repositories and API clients are adapters.
3. **NestJS** for the API and the worker (DI, modules, guards). **Next.js** for web only.
4. **Webhooks only acknowledge.** Verify the signature → insert into `webhook_events` (unique key) → enqueue → `200`.
   All Graph API calls happen in the worker.
5. **Exactly-once effects.** `automation_runs (automation_id, trigger_ref)` is unique, so retries never DM twice.
6. **Rate limiting** is a Redis token bucket per account. Rate-limited jobs are delayed, not failed.
7. **Tokens** are AES-256-GCM encrypted with a versioned keyring (rotation without downtime).
8. **Token lifecycle** is owned by one service (`AccountCredentials`). A scheduled sweep renews tokens inside each
   platform's window, publishing refreshes just-in-time, and any credential rejection flips the account to
   `reauth_required`. Transient failures leave the account active so the next sweep retries.
9. **PostgreSQL** with UUIDv7 keys, `timestamptz`, FK cascades, and indexes matched to hot queries.
   Automation actions are a JSONB column on the aggregate (always read/written together, validated twice).

## Deferred (tracked)
- Postgres **Row-Level Security** as a second tenant wall (currently every repository query is workspace-scoped).
- **Monthly partitioning** of `webhook_events` and `automation_runs` once volume justifies it.
- Transactional outbox (currently: compensation delete if enqueueing fails after the dedupe insert).
