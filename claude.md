# CLAUDE.md — HigherSync

**Project**: E-commerce fulfillment backend for `.soultune` binaural audio protocols. Self-hosted on Contabo VPS. ClickBank IPN webhooks, JWT-signed download links, Resend transactional email. Privacy-first: no accounts, no DB, no cloud SDKs.

---

## Hard Rules (do not violate without explicit instruction)

- **Ports**: backend `3012`, frontend `3013`, db `5434`. NEVER use `3010`/`3011`/`5433` — those belong to the Lieferfly project on the same VPS.
- **No cloud SDKs**: no Vercel, Supabase, Firebase, Auth0, AWS SDK, GCP SDK. This is a self-hosted Express project.
- **No Next.js / NestJS / Fastify patterns**. Plain Express only.
- **No TypeScript** in backend. Plain JS.
- **No PostgreSQL queries** in `server.js`. The DB is dormant. Products live on disk (`produkte/`).
- **No user accounts, sessions, or cookies**. Token-based downloads only.
- **No secrets in source or commits**. Read from `.env`.
- **No async logging libraries** (Winston, Pino) unless explicitly requested. Synchronous file logging only.
- **Never modify the HMAC verification block** in the ClickBank webhook without explicit user request.

## Anti-Patterns (refuse and flag)

If you catch yourself reaching for any of these, stop and ask:
- Adding `pg`, `mongoose`, `prisma`, or any DB driver
- Wrapping the webhook in `express.json()` BEFORE the rawBody capture
- Moving secrets into committed config
- Introducing a build step (webpack, vite, esbuild) on the backend
- Replacing JWT with sessions
- Adding TypeScript transpilation to backend

---

## Stack

- Node.js + Express (backend `:3012`)
- Static HTML/CSS/JS via Express (frontend `:3013`)
- PM2 process manager — `ecosystem.config.js`
- Nginx reverse proxy + Let's Encrypt
- Resend (transactional email)
- JWT — `48h` paid downloads, `7d` free leads
- HMAC-SHA1 — ClickBank IPN verification

## Directory Map

```
backend/server.js              Webhook, JWT issuance, download endpoints
backend/test_email.js          Resend smoke test
backend/ipn-failures.log       HMAC mismatch / parse failures (synchronous append)
frontend/public/               Static pages (index, shop, product, affiliates)
frontend/public/products.json  Product catalog consumed by frontend
produkte/                      .soultune files (paid + free)
produkte/{slug}/manifest.json  Per-product audio specs
docs/                          Deployment guides, runbooks
ecosystem.config.js            PM2 process definitions
.env                           Secrets (gitignored)
```

## Endpoint Contract

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/api/webhooks/clickbank` | HMAC-SHA1 via `cbits` header | Verify IPN, issue 48h JWT, send email, 200 within 5s |
| POST | `/api/optin` | None | Free lead capture, issue 7d JWT for deep-sleep |
| GET  | `/api/download/:token` | JWT 48h | Stream paid `.soultune` |
| GET  | `/api/download-free/:token` | JWT 7d | Stream free `.soultune` (deep-sleep) |
| GET  | `/api/health` | None | Liveness probe |

## ClickBank IPN Flow (canonical sequence)

1. Receive POST. Capture `rawBody` BEFORE any body parsing middleware runs.
2. Verify `cbits` header: `HMAC-SHA1(rawBody, CLICKBANK_SECRET)`. On mismatch, append to `ipn-failures.log` and return `401`.
3. Parse `receipt_id` and `email` from body.
4. Sign JWT: `{ receipt_id, email, exp: now + 48h }` with `JWT_SECRET`.
5. Send email via Resend with download link `https://highersync.com/api/download/{token}`.
6. Return `200` within 5s. ClickBank retries on timeout — slow handlers cause duplicate fulfillment.

Logging must be synchronous and complete BEFORE the response is sent.

---

## Pre-Change Checklist

Before editing `server.js`:
- Confirm change does not silently touch HMAC verification.
- Confirm `rawBody` capture remains the first middleware.
- Confirm port `3012` is unchanged.
- Confirm no new dependency added without user approval.

Before editing the webhook handler specifically:
- Preserve the synchronous log path (`ipn-failures.log`).
- Preserve the 5s response budget — no awaiting slow upstream calls.
- Preserve JWT expiry (`48h`). Changing this invalidates live download links.

## Post-Change Verification

After any backend change:
1. `pm2 restart highersync-backend`
2. `pm2 logs highersync-backend --lines 30` — confirm clean boot, no exceptions.
3. `curl -s https://highersync.com/api/health` — expect `200`.
4. If the webhook was touched: replay the most recent failed IPN from `ipn-failures.log` (if present) against a local instance before redeploying.

## When to Ask the User Before Acting

Stop and ask before:
- Touching HMAC verification logic in any way
- Changing JWT structure or expiry (breaks live tokens)
- Adding any backend dependency (`npm install` in `backend/`)
- Modifying `manifest.json` schema (cross-cuts frontend + mobile SoulTune app)
- Introducing a new env var (requires VPS deploy step)
- Anything that implies migrating to a database
- Removing or renaming an existing endpoint

---

## Development Commands

```bash
# Start (production)
pm2 start ecosystem.config.js

# Logs
pm2 logs highersync-backend --lines 100
pm2 logs highersync-frontend --lines 100

# Restart after code change
pm2 restart highersync-backend highersync-frontend

# Nginx reload (after vhost edit)
sudo nginx -t && sudo systemctl reload nginx

# Smoke tests
curl https://highersync.com/api/health
node backend/test_email.js

# IPN debug
tail -f backend/ipn-failures.log
```

## Failure → Diagnostic

| Symptom | First check |
|---------|-------------|
| Webhook returns 401 | `cbits` header missing or HMAC mismatch → verify `CLICKBANK_SECRET`, inspect `ipn-failures.log` |
| Email not delivered | `RESEND_API_KEY` valid? Run `node backend/test_email.js` |
| Download returns 403 | JWT expired or tampered → check expiry math (48h paid, 7d free) and `JWT_SECRET` rotation |
| 502 from Nginx | PM2 process down → `pm2 status` then `pm2 restart highersync-backend` |
| Static asset 404 | Express static mount drift → confirm `frontend/public/` is served by `highersync-frontend` |
| ClickBank shows IPN delivered but no email | Webhook responded too slow OR Resend silent failure → cross-check `ipn-failures.log` and Resend dashboard |

## Adding a New Product

1. Copy `.soultune` file to `produkte/{slug}.soultune`.
2. Create `produkte/{slug}/manifest.json` matching existing schema (frequency, duration, channels).
3. Add entry to `frontend/public/products.json`.
4. If the product needs custom routing (different JWT lifetime, alternate file path), add a route in `server.js` mirroring the `/api/download/:token` pattern. Do NOT generalize the existing handler unless asked.

---

## Code Conventions

- Express middleware chains stay flat — no deep router composition unless complexity demands it.
- Inline request validation. No schema libraries pre-installed (`zod`, `joi`, etc.). Use `express-validator` only if already in `package.json`.
- Comments only where the *why* isn't obvious. No JSDoc unless asked.
- Errors: log to file, then respond. Never swallow silently.
- All logging synchronous and file-based. No log shipping, no structured logging libraries.

## Environment Variables

```
CLICKBANK_SECRET   HMAC-SHA1 secret (ClickBank vendor settings)
JWT_SECRET         Token signing secret (rotating invalidates ALL live tokens)
RESEND_API_KEY     Resend transactional email key
PORT               Backend port — 3012, do not change
```

## File References

- Nginx vhost: `/etc/nginx/sites-available/highersync.com`
- PM2 config: `./ecosystem.config.js`
- IPN failure log: `./backend/ipn-failures.log`
- Paid product: `produkte/Quantum-Wealth.soultune`
- Free product: `produkte/deep-sleep.soultune`