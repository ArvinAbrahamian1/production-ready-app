# Security

Order matters: tenancy first, because a tenancy bug is the one failure that ends a SaaS. Then auth, secrets, transport, input, uploads, webhooks, data protection, dependencies, mobile, audit, incidents.

## 1. Tenancy

The tenant is whatever owns data: a dealer account, a building, a company. Users join tenants through memberships. A user with no membership sees nothing.

Rules:

1. `tenant_id uuid not null` on every tenant owned table, with a foreign key to `tenants`.
2. `memberships(user_id, tenant_id, role, created_at)` with a unique index on `(user_id, tenant_id)`. Roles: `owner`, `admin`, `member`, `viewer`.
3. The API connects as `app_api`, a login role that cannot bypass RLS. Never as `postgres`, never with the Supabase service role.
4. RLS is enabled and forced on every table. Policies compare `tenant_id` to a transaction local setting.
5. The API resolves the tenant from the verified JWT subject and the memberships table, then sets it inside the transaction. The tenant id in a request body, query string or header is never trusted; at most it selects which of the user's memberships applies, and the membership lookup is what sets the context.
6. Application queries still filter by `tenant_id` explicitly. RLS is the net under the tightrope, not the tightrope.
7. Composite foreign keys `(tenant_id, parent_id)` so a child row cannot point at another tenant's parent.
8. Cross tenant requests return 404, not 403, so an attacker cannot enumerate ids.
9. `test/rls.test.ts` creates two tenants and, for every table, proves tenant B's connection sees zero rows of tenant A and cannot insert into A. This test gates every migration.

Why transaction local: Supavisor and PgBouncer in transaction mode hand the same server connection to different clients between transactions. A session level `SET` leaks the previous tenant to the next request. `set_config(..., true)` dies with the transaction.

```sql
-- roles (run once per database, as the owner role: postgres on Supabase, app_migrator on RDS)
create role app_api login password '<from secrets manager>' nobypassrls;
create role app_worker login password '<from secrets manager>' nobypassrls;
create schema if not exists app;
grant usage on schema app to app_api, app_worker;
revoke all on schema public from public;
grant usage on schema public to app_api, app_worker;
grant select, insert, update, delete on all tables in schema public to app_api, app_worker;
alter default privileges in schema public grant select, insert, update, delete on tables to app_api, app_worker;
alter role app_api set statement_timeout = '5s';
alter role app_api set idle_in_transaction_session_timeout = '10s';
alter role app_api set lock_timeout = '2s';
alter role app_worker set statement_timeout = '60s';

-- tenant context helper
create or replace function app.set_tenant(p_tenant uuid) returns void
language plpgsql as $$
begin
  if p_tenant is null then raise exception 'tenant required'; end if;
  perform set_config('app.tenant_id', p_tenant::text, true);
end $$;

create or replace function app.current_tenant() returns uuid
language sql stable as $$
  select nullif(current_setting('app.tenant_id', true), '')::uuid
$$;

-- per table
alter table listings enable row level security;
alter table listings force row level security;
create policy listings_tenant_select on listings for select
  using (tenant_id = app.current_tenant());
create policy listings_tenant_insert on listings for insert
  with check (tenant_id = app.current_tenant());
create policy listings_tenant_update on listings for update
  using (tenant_id = app.current_tenant()) with check (tenant_id = app.current_tenant());
create policy listings_tenant_delete on listings for delete
  using (tenant_id = app.current_tenant());
create index listings_tenant_created_idx on listings (tenant_id, created_at desc);
```

```ts
// auth/tenant.ts: every handler that touches data runs inside this
export async function withTenant<T>(tenantId: string, fn: (tx: Tx) => Promise<T>) {
  return db.transaction(async (tx) => {
    await tx.execute(sql`select app.set_tenant(${tenantId}::uuid)`);
    return fn(tx);
  });
}
```

On Supabase specifically: because clients never call PostgREST, revoke every grant on the app schema from `anon` and `authenticated` so the Data API exposes nothing, and leave the `auth` schema alone since Supabase Auth owns it. On RDS or any plain Postgres none of that exists; the roles above are the whole story, which is why the move is painless.

## 2. Authentication and authorization

1. The identity provider (Supabase Auth at tier 1, Auth0, Cognito or in house later; see `providers-and-tiers.md`) issues JWTs. The API verifies them itself against a list of trusted issuers: fetch each issuer's JWKS, cache keys 10 minutes, check signature, `exp`, `iss`, `aud`. Reject tokens with `alg` none or an unexpected algorithm. Because the API trusts issuers rather than a vendor SDK, changing providers means adding an issuer, migrating users, then removing the old issuer.
2. Access tokens live one hour. Clients refresh with the provider's SDK; the API never refreshes.
3. Email verification required before the API accepts the user (check the verified claim or the `users` mirror row).
4. Password policy at the provider: minimum 10 characters, leaked password protection on.
5. Login abuse: rate limits from `rate-limiting.md`, lock the account for 15 minutes after 10 failed logins, notify the user by email on a new device login.
6. iOS: Sign in with Apple is mandatory in the App Store if any third party login is offered. Offer Apple, Google and email.
7. Authorization is two layers: `requireRole('admin')` on the route for coarse checks, RLS for the rows. Resource ownership inside a tenant (only the author can edit) is a policy or an explicit `where author_id = ...`, decided per resource in PLAN.md.
8. Platform admin (the owner and staff): `platform_admins(user_id)` table, checked on `/admin/*` in addition to Cloudflare Access. Admin reads that cross tenants use a dedicated `app_admin` role with a policy allowing `app.current_tenant() is null` only when `current_setting('app.is_admin', true) = 'true'`, set the same transaction local way. Every admin action writes `audit_log`.
9. API keys for integrations (if the product has them): random 32 bytes, stored as sha256 hash, prefix shown once, scoped to a tenant and a permission list, revocable, last used timestamp updated asynchronously.
10. Sessions on web: the provider's SSR cookie handling (`@supabase/ssr` at tier 1), `SameSite=Lax`, `Secure`, `HttpOnly`. Server components forward the access token to the API as a bearer. Client side calls (client components, refetch on focus) send the same bearer token from the provider's browser session. The browser never sends cookies to the API, so CSRF is out of the picture and CORS is the only cross origin control to configure.

## 3. Secrets

1. Where they live: the platform's secret store at every tier (Railway variables, Vercel env, EAS secrets and GitHub Actions secrets at tier 1; AWS Secrets Manager or Vault from tier 2). They reach the app as environment variables, so the store can change without the code noticing. Nowhere else.
2. `.env` in `.gitignore`. `.env.example` committed with placeholders and one comment per variable.
3. One set per environment. Production keys never touch staging or a laptop beyond the owner's own `.env`.
4. Rotation: 90 days for third party API keys, immediately on any suspicion, and whenever someone with access leaves. Rotating is a documented step in RUNBOOK.md per secret.
5. Scope keys down: Stripe restricted keys, R2 tokens scoped to one bucket, Resend keys per environment, Cloudflare tokens with only the permissions used (cache purge, DNS edit).
6. Mobile: anything bundled into the app binary is public (`EXPO_PUBLIC_*` on Expo, any constant in Swift). Only the API URL, the identity provider's public URL and anon key, the analytics key and the error tracking DSN belong there.
7. Build gate in CI: fail if `service_role`, `sk_live`, `BEGIN PRIVATE KEY`, `AKIA` or `rk_live` appear in `.next/`, `dist/` or the Expo export.
8. Database password separate from any dashboard password. Network restrictions on the database: only the API and worker egress IPs at tier 1, the private subnet from tier 2.

## 4. Transport and headers

1. TLS everywhere. Cloudflare Full (Strict) to the origin. HSTS with `max-age` of six months once stable, then preload.
2. The origin only accepts Cloudflare. Either Tunnel (no public ingress) or a check that the connection came from a Cloudflare IP range before trusting `cf-connecting-ip`.
3. API headers via `@fastify/helmet`: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Content-Security-Policy: default-src 'none'; frame-ancestors 'none'` (an API serves no HTML), `Permissions-Policy` denying camera, microphone, geolocation.
4. Web headers in `next.config`: same plus a CSP allowing `'self'`, the R2 host for images, PostHog and Sentry hosts for connect, and nonces for inline scripts.
5. CORS on the API: explicit allowlist of the web origins per environment. Never `*` with credentials. Mobile uses bearer tokens so CORS does not apply.
6. Body limit 1 MB by default on the API. Uploads never pass through the API; see section 6.

## 5. Input and output

1. Every route declares zod schemas for params, query, body and response. `.strict()` on objects so unknown keys are rejected. String max lengths everywhere. Enums as `z.enum`. IDs as `z.string().uuid()`.
2. Pagination: cursor based, `limit` capped at 100, default 25.
3. Queries are parameterized through Drizzle. `sql.raw` with user input is a review blocker.
4. Search input: trim, cap at 200 characters, escape for `ILIKE` or use full text search. Never build regex from user input.
5. Server side fetches of user supplied URLs (link previews, imports): allowlist of hostnames, resolve DNS and reject private and link local ranges, 5 second timeout, no redirects, 2 MB response cap, done in the worker.
6. Responses expose only what the client needs. Internal columns (`content_hash`, provider ids, raw payloads) stay out of public DTOs. Error responses never include stack traces or SQL.
7. Sorting and filtering fields come from an allowlist map, never from the raw query string.
8. Numeric money fields are integers in cents or `numeric`; never floats.
9. Dates in and out are ISO 8601 UTC. Timezone is a presentation concern on the client.

## 6. Uploads and files

1. Client asks the API for a presigned PUT: `POST /v1/uploads` with `{ kind, contentType, size }`. The API validates kind and content type against an allowlist, caps size (images 10 MB, video 500 MB, documents 20 MB), creates a `files` row with status `pending`, returns the URL and the key.
2. Key schema: `t/{tenantId}/{kind}/{uuid}.{ext}`. Tenant prefix makes per tenant deletion and export a prefix listing.
3. Client PUTs directly to the bucket (R2 or S3, same API). Then `POST /v1/uploads/{id}/complete`.
4. Worker job `media.process`: `HEAD` the object (size matches), read the first bytes and check magic numbers against the declared type, images get decoded and re encoded with `sharp` (strips EXIF including GPS, kills polyglot files), videos get probed with `ffprobe`, documents get scanned with ClamAV if the product accepts documents from the public. Reject and delete on any mismatch.
5. Never accept SVG from users (script injection). Convert to PNG in the worker or refuse.
6. Serving: private files through short lived signed GET URLs (15 minutes) generated by the API after a permission check. Public assets (avatars, listing photos on a public site) from a public bucket on a custom domain with immutable cache headers, keys unguessable.
7. Deletion: deleting a row enqueues a bucket delete; a nightly job lists orphans older than 24 hours and removes them.

## 7. Inbound webhooks

1. Keep the raw body (Fastify `rawBody` plugin) for signature verification.
2. Verify HMAC with `crypto.timingSafeEqual`. Reject when the signature header is missing, when the timestamp is older than 5 minutes, or when the secret for that provider is not configured.
3. Dedupe by the provider's event id: `SET dedupe:{provider}:{eventId} 1 NX EX 86400`. Duplicate: return 200 and do nothing.
4. Persist the raw event to `inbound_events` and enqueue. Respond 200 in under two seconds. Never do the work inline.
5. Stripe: `stripe.webhooks.constructEvent(rawBody, sig, secret)`. Handle `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`. The handler reads the subscription from Stripe by id (not from the event payload) before writing, so out of order events cannot regress state.
6. Each provider has its own route and its own secret. One leaked secret affects one provider.
7. Alert when signature failures exceed 10 per minute: either the secret rotated or someone is probing.

## 8. Outbound calls

1. Every call has a timeout: 5 seconds default, 30 seconds for uploads to third parties, 60 seconds for LLM calls.
2. Retries only on network errors, 429 and 5xx, only for idempotent operations, exponential backoff with full jitter, maximum 5 attempts in the request path (better: none, enqueue instead) and 8 in the worker.
3. Circuit breaker per provider (`opossum`): open at 50 percent failures over 30 seconds, half open after 60 seconds. When open, the feature degrades (queue for later, show "delayed") instead of hanging.
4. Honor `Retry-After`.
5. Provider credentials are scoped and per environment. Never share a production provider account with staging.

## 9. Data protection

1. At rest: every managed database and object store in `providers-and-tiers.md` encrypts by default. Field level encryption on top for provider tokens, phone numbers, session blobs and anything that would let an attacker impersonate a user or a provider: AES 256 GCM through `node:crypto`, key from env, `key_version` column so rotation is a background job.
2. Anything only ever compared (API keys, invite tokens, reset tokens) is stored as a hash, never plain.
3. Minimization: do not store what the product does not use. Raw provider payloads expire after 90 days (partition drop). Audit log keeps one year.
4. Logs never contain tokens, passwords, cookies, card data or full phone numbers. Pino `redact` paths: `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.token`, `*.phone`.
5. User rights, built before launch because Apple and the privacy laws both require them:
   * `DELETE /v1/me`: soft delete immediately (sign out everywhere, hide data), hard delete after 30 days by a job that walks the tenant prefix in R2 and every table. Apple requires in app account deletion.
   * `POST /v1/me/export`: a job builds a zip of the user's data in R2 under `exports/`, emails a signed link valid 7 days, logs the export.
   * Tenant deletion (owner only, with a typed confirmation): same pattern at tenant scope.
6. Privacy policy and terms URLs in the app, on the web, and in the App Store listing. Data retention written down in PLAN.md.
7. Backups are encrypted and restore tested; see `database.md`.

## 10. Dependencies and supply chain

1. Lockfile committed. `npm ci` in CI. `npm audit --audit-level=high` fails the build.
2. Renovate or Dependabot weekly, grouped minor updates, majors reviewed.
3. Pin GitHub Actions to commit SHAs.
4. No package with install scripts gets added without reading what the script does.
5. Docker image: official `node:22-slim`, non root user, `npm ci --omit=dev`, no secrets in build args.

## 11. Mobile

1. Tokens in the Keychain (Swift) or `expo-secure-store` (Expo), never UserDefaults or AsyncStorage, never logged.
2. No secrets in the bundle (see section 3). The app talks only to the API and to the identity provider.
3. On Expo, EAS Update code signing on, so only the owner's key can push JavaScript to installed apps. On Swift there is no over the air path; every change ships through review.
4. Deep links and universal links validated: the app only navigates to routes it knows, and never passes link parameters straight into API calls without the same zod validation.
5. App Transport Security stays at the default (no arbitrary loads).
6. Certificate pinning and jailbreak detection are not worth it unless the product handles money directly. Skip them.
7. Push tokens are stored per device with the user id, removed on logout and on delivery failure.
8. Screens with sensitive data hide content in the app switcher snapshot (`expo-screen-capture` or a blur overlay on background).

## 12. Audit log

```sql
create table audit_log (
  id uuid primary key,
  tenant_id uuid,
  actor_id uuid,
  action text not null,
  target_type text,
  target_id text,
  ip inet,
  user_agent text,
  metadata jsonb not null default '{}',
  created_at timestamptz not null default now()
);
create index audit_log_tenant_created_idx on audit_log (tenant_id, created_at desc);
revoke update, delete on audit_log from app_api, app_worker;
```

Write a row on: login, logout everywhere, role change, invite, billing change, export, deletion, API key create or revoke, settings change, every `/admin/*` mutation. Append only: the app roles cannot update or delete it.

## 13. Incident basics

`docs/RUNBOOK.md` holds, as numbered steps with the exact commands:

1. Block an IP, an ASN or a country at Cloudflare.
2. Suspend a tenant: `update tenants set status = 'suspended'`, cache version bump, every request then gets 403 from the auth middleware.
3. Pause the worker (scale the worker service to 0 on whatever runs it) and resume.
4. Rotate each secret, one section per secret, including where it is consumed.
5. Restore from PITR to a new project and repoint, with the expected duration from the last test.
6. Purge the cache (tenant version bump or `FLUSHDB` on `redis-cache` only, never on `redis-queue`).
7. Who to tell: affected users (template email), Apple if a store listing changes, Stripe if billing is affected.
8. Where to look first: the error tracker, platform logs filtered by request id, the database dashboard's connections panel, the edge's security events.
