# Physiotherapy Clinic Booking System

Next.js 16 + Drizzle ORM + Postgres (Neon) booking system for a physiotherapy
clinic, with phone/OTP login, service booking, a doctor dashboard, and
finance/stats reporting.

## What changed to make this production-ready

The uploaded project was a working prototype with one **critical**
vulnerability and several gaps expected before going live. Fixed:

1. **Forgeable auth tokens (critical).** The old "token" was just
   `base64(JSON.stringify({ userId, role }))` with no signature — anyone
   could hand-craft `{ "role": "doctor" }` and get full doctor access to
   every patient's data and the finance dashboard. Auth now uses signed
   HS256 JWTs (`src/lib/auth.ts`) verified against a secret (`JWT_SECRET`)
   that only the server knows.
2. **OTP abuse.** `/api/auth/send-otp` and `/api/auth/verify-otp` had no
   rate limiting, so an attacker could SMS-bomb any phone number (racking up
   your SMS bill) or brute-force a 6-digit OTP (1,000,000 combinations) within
   its 5-minute validity window. Both routes now enforce per-phone and per-IP
   limits, backed by Postgres so limits hold even across multiple server
   instances (`src/lib/rate-limit.ts`).
3. **OTP leakage in production.** If SMS delivery wasn't configured, the API
   always returned the OTP in the JSON response ("demo mode"), which would
   have let anyone log in as anyone by just requesting an OTP. This now only
   happens outside production, or in prod if you explicitly set
   `ALLOW_DEMO_OTP=true` (for staging/demos only).
4. **Booking race condition.** Two people booking the same slot at nearly
   the same time could both succeed (check-then-insert race). Booking now
   runs inside a `SERIALIZABLE` transaction with automatic retry on conflict.
5. **No input validation.** Route bodies were used directly from
   `req.json()`. All write endpoints now validate input with `zod` and
   reject malformed/out-of-range data with a 400.
6. **Missing security headers, `X-Powered-By` disabled, HSTS, frame/embed
   protection** — added in `next.config.ts`.
7. **Environment variables were unvalidated** — `src/lib/env.ts` validates
   everything on boot and fails fast with a clear message instead of a
   confusing runtime crash mid-request.
8. **No migrations** — the schema only existed as Drizzle table definitions
   with nothing tracking DB structure over time. Generated proper SQL
   migrations under `drizzle/`, plus useful indexes (`appointments` by date
   and user, `otp_codes` by phone) that the original schema was missing.
9. **Docker/production build** — added a multi-stage `Dockerfile` using
   Next's `standalone` output, `.dockerignore`, and `docker-compose.yml`.

## Getting started

```bash
npm install
cp .env.example .env   # then fill in real values (see below)
npm run db:migrate      # creates all tables in your database
npm run dev
```

### Environment variables (`.env`)

| Variable | Required | Notes |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string (Neon, RDS, etc.) |
| `JWT_SECRET` | yes | ≥32 random chars. Generate: `node -e "console.log(require('crypto').randomBytes(48).toString('base64'))"` |
| `FAST2SMS_API_KEY` | no | Enables real SMS delivery of OTPs |
| `ALLOW_DEMO_OTP` | no | `true` only for staging/demo — lets the API return the OTP in the response when SMS isn't sent |
| `NODE_ENV` | no | `development` \| `production` |

A `.env` has already been created for you with:
- your Neon `DATABASE_URL`
- a freshly generated, random `JWT_SECRET`
- `ALLOW_DEMO_OTP=true` (so login works locally without an SMS provider)

**Before deploying to real production:** set these as environment variables
in your hosting platform (Vercel/Docker host/etc.) rather than shipping the
`.env` file, set `NODE_ENV=production`, set `ALLOW_DEMO_OTP=false` (or leave
unset), and configure `FAST2SMS_API_KEY` so OTPs actually get delivered by
SMS instead of failing closed.

### Database setup

This sandbox's network can't reach your Neon host directly (Neon isn't on
its egress allowlist), so migrations couldn't be applied for you. Run this
yourself once you have `.env` in place and network access to Neon:

```bash
npm run db:migrate
```

Since your database already has `otp_codes`, `services`, `appointments`,
and `gallery_items` from before, the migration in `drizzle/0000_*.sql` uses
`CREATE TABLE IF NOT EXISTS` / `CREATE INDEX IF NOT EXISTS` and wraps the
foreign-key constraints in a `DO $$ ... EXCEPTION WHEN duplicate_object`
block, so it's safe to run even though some tables are already there — it
will only create what's missing (`users`, `rate_limit_events`, the new
indexes, and the FK constraints) and skip the rest.

**One important caveat:** since I can't connect to your database from here,
I can't confirm the *columns* on your existing `otp_codes` / `services` /
`appointments` / `gallery_items` tables match `src/db/schema.ts` exactly. If
they were created by an earlier version of this same schema they'll match.
If they diverge (different column names/types), the migration will succeed
but the app may error at query time on the mismatched table(s). If that
happens after you deploy, run `npx drizzle-kit introspect` against your
database and share the output (or the error) and I can reconcile the schema.

If you change `src/db/schema.ts` later, regenerate and re-run migrations:

```bash
npm run db:generate
npm run db:migrate
```

## Deployment

### Vercel (simplest)
Push to a Git repo, import into Vercel, set the environment variables above
in the project settings, then run `npm run db:migrate` once (locally, or via
a one-off script/CI job) pointed at the same `DATABASE_URL`.

### Docker
```bash
docker compose up --build
```
This builds the standalone Next.js server and runs it on port 3000. Run
`npm run db:migrate` separately (locally or via CI) against the same
database before starting the container for the first time.

## Security notes / things to revisit as this grows

- Rate limiting here is a simple Postgres-backed fixed window good for a
  single clinic's traffic. If you outgrow it, swap `src/lib/rate-limit.ts`
  for Redis/Upstash.
- `rate_limit_events` grows over time; prune it periodically (there's a
  `pruneOldRateLimitEvents()` helper — call it from a daily cron/scheduled
  function).
- Consider adding an audit log for doctor actions on the finance/appointment
  endpoints if this handles real patient billing data.
