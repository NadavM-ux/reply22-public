# Security

## Reporting a vulnerability

Email the address in `apps/web/src/lib/brand.ts` (`BRAND.privacyEmail`). Please don't open a public issue.

## Controls in place

| Area | Control | Where |
|---|---|---|
| Passwords | Hashed by Better Auth; 10–128 chars; rejected if found in breach corpus (k-anonymity) | `common/auth/auth.factory.ts` |
| Accounts | Email verification required; reset links single-use, 1 h expiry, revoke all sessions | `auth.factory.ts` |
| 2FA | TOTP + 10 backup codes; attempts rate-limited | `auth.factory.ts`, `auth-rate-limits.ts` |
| Brute force | Per-IP limits in Redis (sign-in 5/min, reset 5/h, 2FA 5/min); API 120/min; webhooks 1000/min | `auth-rate-limits.ts`, `app.module.ts` |
| Client IP | Taken as the right-most address not in `TRUSTED_PROXIES`, so `X-Forwarded-For` can't be spoofed | `main.ts`, `auth.factory.ts` |
| Sessions | httpOnly, SameSite=Lax, Secure in production; 7-day expiry; listable and revocable | Better Auth |
| CSRF | SameSite cookies + `SameOriginGuard` rejects cross-origin writes; form bodies only accepted on Meta callbacks; Next server actions check Origin | `same-origin.guard.ts`, `main.ts` |
| Tenancy | `WorkspaceGuard` checks membership on every workspace route; repositories filter by `workspaceId` | `workspace.guard.ts`, `packages/db` |
| Input | Zod schemas on every body; UUID pipes on params; size limits on bodies (1 MB JSON, 16 KB forms) | `@app/contracts`, `main.ts` |
| Webhooks | HMAC-SHA256 verified with constant-time compare over the raw body; Meta callbacks verify `signed_request` | `packages/integrations/src/meta` |
| OAuth | Signed, expiring `state` bound to user + workspace; an account can't be moved between workspaces | `packages/security/src/signed-state.ts`, social-account repository |
| Secrets at rest | Platform tokens AES-256-GCM with a versioned keyring (rotate by prepending a key) | `packages/security/src/token-cipher.ts` |
| Logging | No headers or query strings in request logs; tokens never logged | `common/logging/logger.config.ts` |
| Headers | Helmet on the API; CSP, HSTS, frame-ancestors none, nosniff on the web app | `main.ts`, `apps/web/next.config.ts` |
| Uploads | Presigned PUT bound to content type and exact length; storage keys never contain user input | `create-upload.use-case.ts` |
| Abuse caps | 10 owned workspaces per user, 100 automations per account, 60-day scheduling horizon | `packages/domain/src/shared/limits.ts` |
| Data minimisation | Daily purge of old webhook payloads, run history, commenters, and abandoned uploads (rows and files) | worker `maintenance/` |
| Dependencies | `pnpm audit --prod` in CI; vulnerable transitive packages pinned; direct dependencies use releases ≥ 14 days old; peers never leak between apps (`autoInstallPeers: false`) | CI, `pnpm-workspace.yaml` |
| Containers | Distroless base pinned by digest; non-root; no shell or package manager; only built code + production dependencies; `.env` excluded from builds; run read-only with all capabilities dropped | `Dockerfile`, `.dockerignore`, `docker-compose.yml` |
| Migrations | Applied by a one-off release step under a Postgres advisory lock, so parallel deploys can't migrate twice | `packages/db/src/migrate.ts` |

## Production checklist

- [ ] `NODE_ENV=production` (secure cookies; missing integration secrets fail boot).
- [ ] `TRUSTED_PROXIES` lists your load balancer / platform proxy ranges (and Cloudflare's, if proxied), and the API is not reachable except through them.
- [ ] Containers run read-only, non-root, with all capabilities dropped; migrations run as a release step before rollout.
- [ ] Unique random `BETTER_AUTH_SECRET`, `OAUTH_STATE_SECRET` (≥ 32 chars) and `TOKEN_ENCRYPTION_KEYS`, stored in the host's secret manager.
- [ ] `RESEND_API_KEY` set and sending domain verified (SPF/DKIM/DMARC).
- [ ] R2 bucket: private, CORS limited to the web origin, and a lifecycle rule aborting incomplete multipart uploads after 1 day (abandoned single uploads are deleted by the worker).
- [ ] Postgres: TLS, point-in-time recovery, least-privilege app role. Redis: TLS, password, `noeviction`.
- [ ] Meta App Dashboard: webhook, deauthorize and data-deletion URLs set ([docs/railway.md](docs/railway.md)).

## Known trade-offs

- The web CSP allows `'unsafe-inline'` scripts because nonce-based CSP would force every page (including the static landing
  page) to render dynamically. Revisit if the app starts rendering user-generated HTML.
- All workspace members currently have the same permissions. Role checks are needed before team invites ship.
- Row-Level Security is not enabled yet; tenant isolation relies on guards and repository filters (see ADR 0001).
