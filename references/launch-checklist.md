# Launch checklist (the gate)

Run this before launch and before any change that touches production infrastructure. Check each item against the repo, the dashboards or a command, not against memory. Report only failures, as a numbered list, each with the fix. Groups A, B, C, L and R block launch outright; the rest can be accepted as a known gap only if the owner says so in writing in PLAN.md.

## A. Tenancy and data isolation (blocking)

1. Every tenant owned table has `tenant_id not null` with a foreign key.
2. RLS enabled and forced on every table in the app schema (`select relname from pg_class where relrowsecurity = false and relnamespace = 'public'::regnamespace and relkind = 'r'` returns nothing).
3. Policies exist for select, insert, update and delete on every table.
4. The API and worker connect as `app_api` and `app_worker`, never as `postgres` or the service role.
5. Tenant context is set with `set_config(..., true)` inside the transaction, from the membership lookup, never from request input.
6. `rls.test.ts` passes with two tenants across every table.
7. Child tables use composite `(tenant_id, parent_id)` foreign keys.
8. Cross tenant reads return 404.
9. On Supabase, grants on the app schema revoked from `anon` and `authenticated`.

## B. Authentication and authorization (blocking)

1. JWT verification checks signature, `exp`, `iss`, `aud`, and rejects unexpected algorithms.
2. Email verification required before API access.
3. Password policy and leaked password protection on in Supabase.
4. Sign in with Apple present on iOS if any third party login exists.
5. `requireRole` on every mutating route; the permissions matrix in PLAN.md matches the code.
6. `/admin/*` behind Cloudflare Access and the `platform_admins` check.
7. Logout everywhere invalidates refresh tokens.
8. New device login email goes out.

## C. Secrets (blocking)

1. `.env` ignored, `.env.example` complete and current.
2. Production, staging and local use different Supabase projects, Stripe keys, R2 buckets and provider accounts.
3. CI grep for `service_role`, `sk_live`, `rk_live`, `AKIA`, `BEGIN PRIVATE KEY` in web and mobile bundles passes.
4. Only `EXPO_PUBLIC_*` values that are safe to be public exist in the mobile app.
5. Database network restrictions on; database password differs from any dashboard password.
6. Every secret has a rotation section in RUNBOOK.md.
7. Third party keys are restricted or scoped (Stripe restricted key, R2 token scoped to the bucket, Cloudflare token scoped to purge).

## D. Network and Cloudflare

1. API host proxied, Full (Strict) TLS, Always Use HTTPS, HSTS on.
2. Origin accepts only Cloudflare: a Tunnel with no public domain on the API (the only option on Railway and most PaaS), or an IP allowlist of Cloudflare ranges on AWS.
3. WAF managed rules on; rate limiting rules A and B configured; Bot Fight Mode on.
4. Turnstile on signup, login, reset and public forms, verified server side.
5. Cache rule bypasses on `Authorization` and `Cookie`.
6. Access policy on `/admin*`.
7. Web host DNS only while on Vercel, with Vercel Firewall bot protection on; proxied through Cloudflare once the frontend is a container behind your own gateway.
8. Security headers present on API and web (check with `curl -I`).
9. CORS allowlist matches the real web origins and nothing else.

## E. API hardening

1. zod schemas with `.strict()` on every route for params, query and body.
2. Pagination capped at 100; keyset cursors on every list.
3. Body limit 1 MB; uploads go direct to R2.
4. No `sql.raw` with user input (grep).
5. Error responses carry a request id and no stack or SQL.
6. SSRF guard on any server side fetch of user URLs.
7. `Idempotency-Key` required on creating POSTs and honored.
8. Sort and filter fields come from allowlists.

## F. Rate limiting and abuse

1. Tiers from `rate-limiting.md` implemented with Redis plus memory insurance; auth routes fail closed.
2. 429 responses carry `Retry-After` and the `RateLimit-*` headers.
3. Concurrency caps on exports, imports and search.
4. Plan multipliers read from the `plans` table and cached.
5. Tenant kill switch (`status = 'suspended'`) enforced in auth middleware.
6. Signup velocity and disposable email checks on.
7. Honeypot on public forms.
8. Alerts on 429 share, Turnstile failures, signup spikes and webhook signature failures.

## G. Caching

1. Caching map in PLAN.md lists every cache with key, TTL and invalidation.
2. Keys are tenant scoped and versioned; no `KEYS` or `SCAN` in request code (grep).
3. Stale while revalidate and single flight in the shared helper; no ad hoc caching elsewhere.
4. Two Redis instances: queue with `noeviction`, cache with `allkeys-lru`.
5. ETags on list and detail routes.
6. Public pages on ISR with on demand revalidation; authenticated pages `no-store`.
7. R2 objects written with immutable cache headers under hashed keys.
8. Cache hit ratio metric visible on a dashboard.

## H. Database

1. Pooled connection string (a transaction mode pooler) in the API and worker; direct only for migrations. `prepare: false`.
2. Pool sizes: API 10, worker 5, total under half the pooler max.
3. Per role `statement_timeout`, `idle_in_transaction_session_timeout`, `lock_timeout` set.
4. Every foreign key indexed; every tenant list query has a matching composite index (`explain` on the top 5 routes).
5. Append only tables partitioned by month with a rotation job.
6. Migrations are files, reviewed, applied to staging first; no dashboard edits.
7. `pg_stat_statements` on; top 10 reviewed once before launch.
8. Autovacuum settings tuned on the biggest hot table.
9. `updated_at` triggers on every table.

## I. Jobs

1. Worker runs as its own service with graceful shutdown.
2. Every job has a deterministic id and is idempotent.
3. Backoff with jitter, 8 attempts, failed set retained 7 days.
4. Bull Board reachable behind Access.
5. Cron set (aggregates, partitions, retention, billing reconcile, backups, health report) in UTC with a leader lock.
6. Alerts on queue age and failed count.
7. Per tenant in flight cap.
8. Webhook route stores raw events and returns within 2 seconds (measured).

## J. Storage and uploads

1. Presigned PUT with content type and size caps; `files` row tracks status.
2. Worker validates magic bytes, strips EXIF, rejects SVG, scans documents if accepted.
3. Private files served through short lived signed URLs after a permission check.
4. Tenant prefixed keys; orphan cleanup job runs nightly.
5. Bucket versioning on for uploads; lifecycle expiry on exports and backups.
6. CORS on the bucket limited to the real origins and methods.

## K. Billing

1. Stripe webhooks verified with the raw body, idempotent on event id, state read back from Stripe by id.
2. `subscriptions` table mirrors Stripe; the app reads entitlements from the table, not from Stripe on every request.
3. Daily reconcile job compares the table with Stripe and alerts on drift.
4. Customer Portal enabled for cancellations and card updates.
5. Trial and promo logic has tests, including expiry and downgrade paths.
6. Test mode keys in staging only.

## L. Backups and recovery (blocking)

1. PITR on in production.
2. Weekly encrypted logical dump to R2 with a 90 day lifecycle.
3. A restore to a scratch project has been done once and the duration recorded in RUNBOOK.md.
4. Manual snapshot procedure written for destructive migrations.
5. R2 versioning on for user uploads.

## M. Mobile

1. Tokens in the Keychain or `expo-secure-store`; nothing sensitive in UserDefaults, AsyncStorage or logs.
2. If Expo: EAS Update code signing on; production channel only.
3. Account deletion in app, export available, privacy policy linked in app and in the store listing.
4. Deep link parameters validated like any input.
5. Sentry with source maps; request id shown on error screens.
6. Push tokens cleaned on logout and delivery failure.
7. Offline cache persisted; app opens with data and refreshes.
8. App Store listing copy reviewed against the project's own constraints in PLAN.md.

## N. Web

1. No database client imported anywhere in the web repo (grep).
2. CSP, HSTS, nosniff, referrer policy, frame ancestors set in `next.config`.
3. `images.remotePatterns` limited to R2.
4. Server components forward the user token; browser never calls the API with cookies.
5. Sentry with source maps and release.
6. Lighthouse performance above 80 on the main authenticated page on a throttled mobile profile.

## O. Observability

1. Pino JSON logs with request id, tenant id, user id, route, status, duration, cache result.
2. Redaction paths configured; a sample log line checked by eye.
3. `/healthz` and `/readyz` implemented; the platform health check on `/readyz`.
4. Better Stack uptime on `/readyz` from two regions plus a synthetic login and feed check.
5. Sentry alert rules: new issue, error rate, 5xx spike, p95.
6. Metrics endpoint scraped; a dashboard shows RPS, latency, 429s, cache, queues, DB connections.
7. Alert thresholds from `observability-and-reliability.md` configured, each naming a runbook section.
8. Daily health report job sends.

## P. Legal and store

1. Privacy policy and terms published, dated, linked everywhere required.
2. Data retention documented in PLAN.md and implemented by the retention job.
3. Account and tenant deletion work end to end (tested on staging).
4. Apple App Privacy answers match what the app actually collects.
5. Emails have a working unsubscribe for anything that is not transactional.

## R. Portability (blocking)

1. Frontend and backend are separate repos and separate deployables.
2. The API and the worker build from a Dockerfile and run locally with `docker compose`; the API is not in serverless functions.
3. The web app has `output: 'standalone'` and a Dockerfile, even while hosted on Vercel.
4. No vendor SDK outside `src/adapters/` (grep for the provider packages in business logic and routes).
5. No PostgREST, Firebase or provider data client calls anywhere in the backend.
6. Schema, roles, RLS and functions are SQL in the repo and replay cleanly on a fresh local Postgres.
7. The app owns a `users` table mirrored from the identity provider; the API verifies tokens through a configurable issuer list.
8. Config is env only, validated on boot; secrets reach the app as env.
9. The DNS zone is owned on Cloudflare; every provider is a CNAME target.
10. Object storage uses the S3 SDK with an endpoint setting; `rclone` config for the bucket exists.
11. `OTEL_EXPORTER_OTLP_ENDPOINT` is a variable; no vendor agent in the code.
12. `infra/README.md` lists every dashboard setting, or Terraform covers it; `docs/PLAN.md` has an exit plan line per provider.

## Q. Load and failure tests

1. k6 at three times expected peak for 10 minutes: p95 under 500ms, zero 5xx, DB connections under 70 percent, queues flat.
2. Redis cache outage drill passed: API serves, auth fails closed, recovery automatic.
3. Provider outage drill passed: jobs delay, circuit opens, UI says delayed, recovery automatic.
4. One real restore from PITR done (see L3).
5. Results and dates written in RUNBOOK.md.
