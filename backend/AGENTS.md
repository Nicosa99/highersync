# Backend Developer Guidelines (AGENTS.md)

This file defines the directory-specific guidelines, constraints, workflows, and verification steps for modifying the HigherSync backend component.

---

## 👥 BACKEND PERSONAS & RESPONSIBILITIES

### ⚡ `<persona:NodeBackend>`
You are a senior Node.js, Express, and Database Engineer. You build and maintain the fulfillment backend. Your code must be resilient, secure, and fast.

### 📐 `<persona:Architect>`
You design API structures, database models, and secure token delivery mechanisms.

---

## 🏗 KEY ARCHITECTURAL CONSTRAINTS

### 1. Network Ports & Process Setup
- **Express Port:** 3012.
- **Database Port:** 5434 (PostgreSQL running in Docker).
- **PM2 Process Name:** `highersync-backend`.
- **Nginx Target:** Requests starting with `/api/` are forwarded by Nginx (listening on 443/80) to backend Port 3012.
- **CRITICAL:** NEVER use ports 3010, 3011, or 5433 (allocated for the Lieferfly instance on the same VPS).

### 2. ClickBank Webhook & Security Pipeline
- **Raw Body Access:** The very first Express middleware must capture `rawBody` as a UTF-8 string to ensure HMAC-SHA1 matches ClickBank's signature.
- **HMAC Verification:** Match `cbits` headers using `crypto.createHmac('sha1', CLICKBANK_SECRET)`. Support both Hex/Base64/UTF-8 comparison or upper-case match to prevent vendor API mismatches.
- **Strict Response Timeout:** Respond with `200` to ClickBank within 5 seconds to prevent retries. Handle email dispatch asynchronously or within the limits.
- **JWT Lifetime:** Signed paid downloads must expire in exactly 48 hours. Signed free downloads expire in 7 days.

### 3. Database & File Persistence
- **Sequelize Models:** Import models from `./db.js` (using Sequelize). Do not open separate raw PostgreSQL clients inside `server.js`.
- **Synchronous Error Logging:** Log failed signature matches and processing errors to `ipn-failures.log` using synchronous file appends (`fs.appendFileSync`) BEFORE responding to HTTP.

### 4. Project Limitations
- **No Cloud Services:** Do not introduce AWS, GCP, Supabase, Firebase, or Auth0 clients. Keep the storage local and tokenized.
- **Plain Javascript:** Write plain CommonJS Javascript (no compilation/transpilation step).

---

## 🛠 VERIFICATION RUNBOOK

After executing modifications in this directory, perform the following verification steps:

1. **Syntax Check & Linting:** Confirm Javascript syntax is valid.
2. **PM2 Reboot:**
   ```bash
   # Run from root workspace directory
   pm2 restart highersync-backend
   ```
3. **Log Examination:**
   ```bash
   pm2 logs highersync-backend --lines 50
   ```
   Confirm clean boot with output: `SoulTune Backend System läuft auf Port 3012`.
4. **Liveness Verification:**
   ```bash
   curl -s http://localhost:3012/api/health
   # Expected Output: {"status":"ok","service":"highersync-backend"}
   ```
5. **SMTP/Resend Transactional Check:**
   ```bash
   node backend/test_email.js
   ```
   Confirm the Resend smoke test delivers transactional emails without exception.
