# Running Reply22 on Railway

Everything runs in one Railway project: three services built from this repo's `Dockerfile`,
plus Railway's Postgres and Redis. Visitors only reach `web`. It serves the site and forwards
`/api/*` to `api` over Railway's private network, so the API never needs a public address.

| Service  | Built as         | Public address | Notes                                       |
| -------- | ---------------- | -------------- | ------------------------------------------- |
| web      | `SERVICE=web`    | reply22.com    | Next.js; proxies `/api/*` to api            |
| api      | `SERVICE=api`    | none           | runs database migrations before each deploy |
| worker   | `SERVICE=worker` | none           | comment replies, publishing, retention jobs |
| Postgres | Railway database | none           | turn on backups if your plan has them       |
| Redis    | Railway database | none           | must keep `maxmemory-policy noeviction`     |

The examples use `reply22.com`. If the domain is different, change it everywhere below.

## 1. Create the project

1. **New Project → Deploy from GitHub repo → `reply22`.** Rename the service it creates to `api`.
2. In the same project, **+ Create → GitHub Repo → `reply22`** twice more, naming them `worker` and `web`.
3. **+ Create → Database → PostgreSQL**, then again for **Redis**.

Railway builds all three app services from the root `Dockerfile`. Each one picks its image with
the `SERVICE` variable (section 3): the Dockerfile declares `ARG SERVICE`, so Railway passes the
variable into the build.

## 2. Service settings

**api**

- Pre-deploy command: `/nodejs/bin/node node_modules/@app/db/dist/migrate.js`
  The image has no shell and Railway runs the command as is, so Node is called by its full path.
  Migrations take a database lock, so two deploys can't run them at once.
- Healthcheck path: `/api/health/ready`
- Restart policy: On failure
- Networking: no public domain.

**worker**: restart on failure, no public domain, no healthcheck.

**web**: healthcheck path `/`, restart on failure, custom domain `reply22.com` on port `3000`
(section 4).

## 3. Variables

Never paste secret values into chat, issues or commits. Paste them only into Railway.

**Shared variables** (Project Settings → Shared Variables, then share each one with `api` and
`worker`):

| Variable                | Value                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------- |
| `NODE_ENV`              | `production`                                                                          |
| `TOKEN_ENCRYPTION_KEYS` | secret: generated (below). Must be identical in api and worker                        |
| `META_APP_ID`           | Meta app → Instagram → API setup with Instagram business login → **Instagram app ID** |
| `META_APP_SECRET`       | same page → **Instagram app secret**                                                  |
| `META_REDIRECT_URI`     | `https://reply22.com/api/connect/instagram/callback`                                  |
| `TIKTOK_CLIENT_KEY`     | TikTok app → client key                                                               |
| `TIKTOK_CLIENT_SECRET`  | TikTok app → client secret                                                            |
| `TIKTOK_REDIRECT_URI`   | `https://reply22.com/api/connect/tiktok/callback`                                     |
| `S3_ENDPOINT`           | `https://<account-id>.r2.cloudflarestorage.com`                                       |
| `S3_REGION`             | `auto`                                                                                |
| `S3_BUCKET`             | `reply22-media`                                                                       |
| `S3_ACCESS_KEY_ID`      | R2 API token → access key ID                                                          |
| `S3_SECRET_ACCESS_KEY`  | R2 API token → secret access key                                                      |

**api**

| Variable                                    | Value                                                                                |
| ------------------------------------------- | ------------------------------------------------------------------------------------ |
| `SERVICE`                                   | `api`                                                                                |
| `PORT`                                      | `4000`                                                                               |
| `DATABASE_URL`                              | `${{Postgres.DATABASE_URL}}`                                                         |
| `REDIS_URL`                                 | `${{Redis.REDIS_URL}}`                                                               |
| `WEB_ORIGIN`                                | `https://reply22.com`                                                                |
| `BETTER_AUTH_URL`                           | `https://reply22.com`                                                                |
| `BETTER_AUTH_SECRET`                        | secret: generated                                                                    |
| `OAUTH_STATE_SECRET`                        | secret: generated                                                                    |
| `META_WEBHOOK_VERIFY_TOKEN`                 | secret: generated (also typed into Meta's webhook setup)                             |
| `TRUSTED_PROXIES`                           | `127.0.0.1/8,::1/128,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,100.64.0.0/10,fc00::/7` |
| `EMAIL_FROM`                                | `Reply22 <hello@reply22.com>`                                                        |
| `RESEND_API_KEY`                            | Resend → API key                                                                     |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | optional; Google sign-in turns on when both are set                                  |

`TRUSTED_PROXIES` lists private address ranges only. The API has no public address, so the only
private sender is the `web` service, which passes on the visitor's address set by Railway's edge.
If you later put Cloudflare's proxy (orange cloud) in front, also add
[Cloudflare's IP ranges](https://www.cloudflare.com/ips/).

**worker**: `SERVICE=worker`, `DATABASE_URL=${{Postgres.DATABASE_URL}}`,
`REDIS_URL=${{Redis.REDIS_URL}}`, plus the shared variables.

**web**

| Variable  | Value                                         |
| --------- | --------------------------------------------- |
| `SERVICE` | `web`                                         |
| `PORT`    | `3000`                                        |
| `API_URL` | `http://${{api.RAILWAY_PRIVATE_DOMAIN}}:4000` |

`API_URL` is read when the web image is built, so changing it needs a redeploy of `web`.

**Generating the secrets.** Run this in your own terminal and paste each line into Railway:

```bash
echo "BETTER_AUTH_SECRET=$(openssl rand -base64 48)"; echo "OAUTH_STATE_SECRET=$(openssl rand -base64 48)"; echo "TOKEN_ENCRYPTION_KEYS=v1:$(openssl rand -base64 32)"; echo "META_WEBHOOK_VERIFY_TOKEN=$(openssl rand -hex 24)"
```

Keep `TOKEN_ENCRYPTION_KEYS` safe: it decrypts the stored Instagram and TikTok tokens. To rotate
it later, put the new key first (`v2:…,v1:…`), so the old tokens still decrypt.

## 4. Domain and DNS (Cloudflare)

Railway can only serve a bare domain (no `www`) through DNS providers that flatten CNAME
records, such as Cloudflare. Move the domain's nameservers to Cloudflare first.

1. Railway → `web` → Settings → Networking → **+ Custom Domain** → `reply22.com`, port `3000`.
   It shows a CNAME and a TXT record. Add **both** in Cloudflare, with the cloud **grey**
   (DNS only). Railway issues the HTTPS certificate once they resolve.
2. `www`: in Cloudflare add an `AAAA` record `www` → `100::` with the cloud **orange**, then
   Rules → Redirect Rules → template **Redirect from WWW to Root** (301, keep the query string).
   Sign-in only accepts `reply22.com`, so `www` must redirect rather than serve the site.
3. Later, to turn on Cloudflare's proxy for `reply22.com` as well: set SSL/TLS to **Full** (not
   Full (strict), which Railway doesn't support) and extend `TRUSTED_PROXIES` as above.

## 5. Outside services

**Cloudflare R2** (videos). Create a private bucket `reply22-media` and an API token with
Object Read & Write on that bucket only. In the bucket's settings, add this CORS policy
(browsers upload straight to R2 with a presigned link, one PUT per file, whose type and size
are signed into the link):

```json
[
  {
    "AllowedOrigins": ["https://reply22.com"],
    "AllowedMethods": ["PUT"],
    "AllowedHeaders": ["content-type"],
    "MaxAgeSeconds": 3600
  }
]
```

No lifecycle rule is needed: the app never starts multipart uploads, and the worker deletes
uploads that were never confirmed after a day.

The bucket stays private: Instagram fetches videos through short-lived presigned links, and
TikTok gets the file uploaded to it in chunks, so no public or verified media domain is needed.

**Inboxes.** Cloudflare → Email Routing: forward `privacy@`, `support@` and `hello@reply22.com`
to the inbox you read. The privacy page names `privacy@reply22.com`, and Meta and TikTok need a
contact address on the domain.

**Resend** (sending email). Add the domain `reply22.com`, add the DNS records Resend shows to
Cloudflare (grey cloud), and add a DMARC record: `TXT _dmarc` → `v=DMARC1; p=none;`. Resend's
records sit on a `send` subdomain, so they don't clash with Email Routing.

**Meta app dashboard**

| Setting                   | Value                                                      |
| ------------------------- | ---------------------------------------------------------- |
| OAuth redirect URI        | `https://reply22.com/api/connect/instagram/callback`       |
| Webhook callback URL      | `https://reply22.com/api/webhooks/meta`                    |
| Webhook verify token      | the `META_WEBHOOK_VERIFY_TOKEN` value                      |
| Webhook fields            | `comments`                                                 |
| Deauthorize callback URL  | `https://reply22.com/api/webhooks/meta/deauthorize`        |
| Data deletion request URL | `https://reply22.com/api/webhooks/meta/data-deletion`      |
| Privacy policy / Terms    | `https://reply22.com/privacy`, `https://reply22.com/terms` |

**TikTok app**: redirect URI `https://reply22.com/api/connect/tiktok/callback`; scopes
`user.info.basic` and `video.publish`; website, terms and privacy URLs as above.

**Google sign-in** (optional): redirect URI `https://reply22.com/api/auth/callback/google`.

## 6. First deploy checklist

- [ ] `api` deploy log shows `migrations applied`, and the healthcheck passes.
- [ ] `https://reply22.com` loads with a valid certificate; `www.reply22.com` redirects to it.
- [ ] Sign up: the confirmation email arrives from `hello@reply22.com`.
- [ ] Connect your own Instagram account (it needs a role on the Meta app until App Review).
- [ ] Comment a keyword on your post: the reply and DM arrive, and the run shows in the dashboard.
- [ ] Upload a short video and publish to Instagram and TikTok (TikTok posts stay private
      until the Direct Post audit passes).
- [ ] Redis: `CONFIG GET maxmemory-policy` returns `noeviction`.
- [ ] Postgres: backups are on (Postgres service → Backups), or a nightly copy is set up.

## Releasing

Pushing to `main` rebuilds and redeploys the services. The `api` pre-deploy step migrates the
database before the new API starts, but `worker` and `web` don't wait for it. So each migration
must also work with the previous release's code: add a column before code uses it, and drop
one only a release after the code stops using it.
