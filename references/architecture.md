# Architecture

The shape every project takes, with the tier 1 provider and the tier 3 destination written on each box so the migration path is visible from day one. Adapt names, drop what the project does not use, never add a box without a reason in PLAN.md.

## Diagram

```
                   ┌────────────────────────────────────────────────┐
                   │                    CLIENTS                     │
                   │  browser        iOS / Android app     providers│
                   │  (web app)      (native or Expo)      (webhooks)│
                   └────────┬───────────────┬──────────────────┬───┘
                            │ https         │ https            │ https
                            ▼               ▼                  ▼
          ┌─────────────────────────────────────────────────────────┐
          │ EDGE   tier 1: Cloudflare Pro                           │
          │        tier 3: Cloudflare Enterprise or CloudFront + WAF │
          │ DNS for every host, TLS, CDN, DDoS, WAF, rate rules,    │
          │ bot filtering, Turnstile, Access on /admin               │
          │   app.domain  ─► frontend     api.domain ─► gateway     │
          └──────────┬───────────────────────────┬──────────────────┘
                     │                           │
                     ▼                           ▼
   ┌────────────────────────┐   ┌────────────────────────────────────────────────┐
   │ FRONTEND               │   │ GATEWAY   tier 1: PaaS router                  │
   │ tier 1: Vercel         │   │           tier 3: ALB + Kubernetes ingress     │
   │ tier 3: container on   │   │ TLS from edge, path routing, health checks     │
   │ k8s behind the CDN     │   └──────────┬─────────────────────┬───────────────┘
   │ Next.js, SSR or static │──────────────┤                     │
   │ no DB, no secrets,     │  calls API   ▼                     ▼
   │ forwards user token    │   ┌─────────────────────┐  ┌─────────────────────────┐
   └────────────────────────┘   │ API                 │  │ WORKER                  │
                                │ one container       │  │ one container           │
                                │ tier 1: Railway     │  │ same image, worker mode │
                                │ tier 2: ECS Fargate │  │ queue consumers, cron,  │
                                │ tier 3: Kubernetes  │  │ outbound calls          │
                                │ jwt verify, tenant, │  │ tier 2+: Temporal for   │
                                │ validate, limits,   │  │ multi step workflows    │
                                │ cache, idempotency  │  │ tier 3: Kafka consumers │
                                └──┬──────────┬───────┘  └───┬──────────┬──────────┘
                                   │          │ enqueue      │          │
                                   │          ▼              ▼          │
                                   │   ┌───────────────────────────┐    │
                                   │   │ QUEUE   Redis, noeviction │    │
                                   │   │ tier 1: Railway Redis     │    │
                                   │   │ tier 3: ElastiCache, Kafka│    │
                                   │   └───────────────────────────┘    │
                                   │   ┌───────────────────────────┐    │
                                   └──►│ CACHE   Redis, allkeys-lru│◄───┘
                                       │ tier 1: Railway Redis     │
                                       │ tier 3: ElastiCache       │
                                       └───────────────────────────┘
                                   │ pooled, transaction mode       │ S3 API
                                   ▼                                ▼
   ┌───────────────────────────────────────────┐   ┌──────────────────────────────┐
   │ DATABASE   Postgres, RLS forced, pooler   │   │ OBJECT STORAGE   S3 API      │
   │ tier 1: Supabase Pro (+ Auth)             │   │ tier 1: Cloudflare R2        │
   │ tier 2: RDS Multi AZ + PgBouncer, replica │   │ tier 3: S3 or R2             │
   │ tier 3: Aurora, partitions, shards        │   │ presigned uploads, signed GET│
   │ PITR, network restricted, SSL enforced    │   │ tenant prefixed keys          │
   └───────────────────────────────────────────┘   └──────────────────────────────┘

   IDENTITY   tier 1: Supabase Auth   tier 3: Auth0, Okta or in house   (API verifies via JWKS, owns users table)
   PAYMENTS   Stripe at every tier    EMAIL   tier 1: Resend  tier 2+: SES
   ANALYTICS  tier 1: PostHog + materialized views   tier 2: ClickHouse or BigQuery by CDC   tier 3: Snowflake + Kafka
   OBSERVABILITY  OpenTelemetry + JSON logs from the app   tier 1: Sentry + Better Stack   tier 3: Datadog or Grafana
   DELIVERY   GitHub Actions CI at every tier   tier 2+: Terraform, blue green   tier 3: ArgoCD, canary, flags service
```

## The three request paths

Every feature is one of these shapes. A feature that fits none is doing too much in one place.

### 1. Read (web feed page)

1. Browser requests `/feed`. The frontend server component runs on the frontend host.
2. It reads the session cookie, takes the access token, calls `GET api.domain/v1/listings?cursor=...` with `Authorization: Bearer <token>`.
3. Edge: proxied, rate rule checked, cache bypassed because of the Authorization header.
4. API: verify JWT (JWKS cached in process), load membership from the cache (60s) else the database, open transaction, set the tenant for the transaction.
5. API: per user rate limit check in Redis.
6. API: cache key `v1:t:{tenant}:ver{n}:listings:{sha1(query)}`. Hit: return. Miss: single flight lock, keyset query through the pooler, store with TTL 30s plus jitter, return.
7. Response carries `ETag`; the next request sends `If-None-Match` and gets 304 when unchanged.
8. Client query cache holds it with a 30s stale time, refetches on focus.

### 2. Write (mobile creates a listing)

1. App sends `POST api.domain/v1/listings` with bearer token, `Idempotency-Key` header (client generated UUID kept until success), validated body.
2. Edge: proxied, write path rate rule.
3. API: JWT verify, membership, role check, per user write limit.
4. API: idempotency check in Redis (`idem:{tenant}:{key}`, 24h). Seen with a stored response: return it. Seen but in flight: 409.
5. API: validate (unknown keys rejected), transaction with tenant set, insert, `audit_log` row if auditable, commit.
6. API after commit: bump the tenant cache version, enqueue `listing.created` with job id `listing:{id}:created`, store the idempotency response, return 201 with the object.
7. Worker: process the uploaded image from object storage (validate magic bytes, resize, strip EXIF), write derived keys back, send notifications through the `notify` queue.

### 3. Ingestion (provider webhook)

1. Provider posts to `api.domain/webhooks/{provider}`.
2. Edge: proxied, path scoped rate rule at a high ceiling, no challenge (providers cannot solve one).
3. API: raw body preserved, HMAC verified with a timing safe compare, timestamp within 5 minutes, event id deduped with `SET dedupe:{provider}:{id} 1 NX EX 86400`.
4. API: insert into `inbound_events` (append only, partitioned by month), enqueue `ingest.parse` with the row id, return 200. Under 200ms total.
5. Worker: parse, normalize, compute `content_hash`, batch insert 500 rows per statement with `on conflict do nothing`, update counters, bump the tenant cache version, enqueue alerts.
6. Parse failures land in the failed set with the raw event id so they can be replayed after the fix.

### Admin path

`/admin/*` sits behind Cloudflare Access (or ALB authentication at tier 3) and still requires a `platform_admins` row for the JWT subject. Every admin action writes `audit_log`. The queue dashboard and any internal tools live here.

## Repo layouts

Two repos minimum, three with mobile: `{project}-api` (API and worker), `{project}-web`, `{project}-ios` or `{project}-mobile`. Separate repos are what let the frontend stay on one provider while the backend moves to another.

### `{project}-api`

```
src/
  server.ts              bootstrap; plugin order: sensible, helmet, cors, requestId,
                         logger, otel, auth, tenant, rateLimit, etag, routes
  worker.ts              queue workers plus cron registration; separate process, same image
  config.ts              env parsed with zod on boot; the process exits if a var is missing
  db/
    schema.ts            drizzle schema (one file per domain when it grows)
    client.ts            postgres.js via the pooler, prepare: false, max 10
    migrations/          SQL generated by drizzle-kit, reviewed by hand
    sql/                 roles.sql, rls.sql, functions.sql, applied by migrations
  auth/
    verify.ts            JWKS fetch and cache, token verification, issuer list
    tenant.ts            membership lookup, set tenant per transaction, requireRole
  http/
    middleware/          rateLimit.ts, cache.ts, idempotency.ts, errors.ts
    routes/              one file per resource, plus webhooks.ts and admin.ts
    schemas/             zod request and response schemas, shared with clients
  jobs/
    queues.ts            queue definitions and typed producers
    handlers/            one file per job type
    cron.ts              repeatable jobs
  adapters/
    storage.ts           S3 SDK with endpoint setting (R2 or S3)
    email.ts             send(template, to, data) over the current provider
    payments.ts          Stripe
    identity.ts          JWKS issuers, user mirror
  lib/
    redis.ts             two clients: cache and queue
    logger.ts, metrics.ts, otel.ts
test/
  integration/           runs against docker compose Postgres and Redis
  rls.test.ts            two tenants, every table; must pass before anything else
  load/                  k6 scripts
docs/
  PLAN.md  RUNBOOK.md  API.md
infra/
  README.md              every dashboard setting at tier 1, written down the day it is clicked
  terraform/             from tier 2: everything on the cloud account
docker-compose.yml       postgres:17 and redis:7 (two instances) for local
Dockerfile               multi stage, node:22-slim, non root user, same image for api and worker
.env.example             every variable with a placeholder and a comment
```

The `adapters/` folder is the portability boundary. Business logic imports adapters; adapters import vendor SDKs. Moving a provider touches one file there and the env.

### `{project}-web`

```
app/                     Next.js App Router; server components fetch from the API
lib/api.ts               typed client generated from the API's zod schemas
lib/auth/                the identity provider's browser and server SDK, sign in only
middleware.ts            session refresh, redirect unauthenticated routes
next.config              output: 'standalone', images.remotePatterns limited to the asset host, security headers
Dockerfile               present from day one even while hosted on Vercel; it is the exit
```

The web app never imports a database client. If a page needs data, the API gets a route.

### `{project}-ios` (Swift) or `{project}-mobile` (Expo)

```
Swift:   SwiftUI app, an APIClient with the bearer token in the Keychain, the identity provider's iOS SDK for
         sign in only, a persisted query layer (SwiftData or a simple cache) so the app opens with data
Expo:    expo-router screens, lib/api.ts (same typed client), expo-secure-store for tokens, TanStack Query
         with an MMKV persister, EAS Update code signing on
```

## Environments

* Local: `docker compose up` gives Postgres and two Redis instances. `.env` from `.env.example`. Identity through the provider's local emulator or the staging project.
* Staging: its own database project or instance, its own backend environment, frontend preview environment, test mode payment keys, a bucket with a `staging` suffix. Same edge zone, `api-staging` host proxied.
* Production: deploys are done by the owner. Migrations run against staging first, then production.

Config rule: `config.ts` parses `process.env` with a zod schema and exits on boot if anything is missing or malformed. A misconfigured service fails loud at start, not at the first request an hour later.

## Edge configuration (Cloudflare, every tier)

1. DNS: the zone lives here at every tier. `app` CNAME to the frontend host (DNS only while on Vercel, proxied once it is a container behind your own gateway). `api` CNAME to the gateway, proxied. If the origin is a VM, run `cloudflared` as a Tunnel and close every inbound port.
   While the frontend is on Vercel and DNS only, bots reach Vercel directly and Vercel's own firewall is what filters them; Cloudflare's bot filtering covers only the API host. That is fine for an MVP. Getting Cloudflare's filtering on the frontend too is the "frontend to a container behind the edge" move in `migrations.md`, which is also the only reason to do that move early.
2. SSL/TLS: Full (Strict). Always Use HTTPS. Minimum TLS 1.2. HSTS with a six month max age once stable.
3. WAF: Cloudflare Managed Ruleset on; OWASP Core Ruleset on with Pro, paranoia level 1 to start.
4. Rate limiting rules: the two coarse rules from `rate-limiting.md`; five at Business.
5. Bots: Bot Fight Mode on; Super Bot Fight Mode on Pro (block definitely automated, allow verified bots).
6. Turnstile on every public form; the API verifies the token with `siteverify` before doing anything.
7. Cache Rules: bypass when `Authorization` or `Cookie` is present; cache `GET /v1/public/*` with a 60s edge TTL and the query string in the key.
8. Access: application for `api.domain/admin*` with the owner's login and a one time PIN policy for anyone else.
9. Origin protection: the gateway accepts only Cloudflare. On AWS that is an IP allowlist of Cloudflare's ranges on the ALB security group. On Railway (and most PaaS) there is no IP allowlist, so the real version is a Tunnel: run `cloudflared` as a second service in the Railway project, point the tunnel at the API over Railway's private network (`api.railway.internal:3000`), and give the API service no public domain at all. Nothing on the internet can reach the API except through Cloudflare, which is the zero exposure setup. The worker never has a public domain either way.
10. Object storage: one bucket per environment, CORS limited to the real origins and methods, versioning on for user uploads, lifecycle expiry on `exports/` and `backups/`, a custom domain only for truly public assets.
11. Notifications: email alerts for DDoS events and WAF spikes.

## Platform configuration

### Tier 1 (Railway, or Fly and Render by picker rule)

1. One project with services `api`, `worker`, `cloudflared`, `redis-queue`, `redis-cache`. Two Redis instances, not one.
2. `api`: no public domain; reachable only from `cloudflared` over the private network. Health check on `/readyz`, restart on failure, one replica to start, vertical scaling first, region closest to users.
3. `cloudflared`: the official image with the tunnel token as its only variable, ingress rule `api.domain` to `http://api.railway.internal:3000`. The Cloudflare DNS record for `api` is created by the tunnel. Staging gets its own tunnel and `api-staging` hostname.
4. `worker`: no public domain, internal health check, one replica, concurrency per queue in code.
5. Variables: shared `DATABASE_URL` (pooled), `REDIS_QUEUE_URL`, `REDIS_CACHE_URL`, identity JWKS URL and issuer, storage endpoint and keys, payment keys, email key, `OTEL_EXPORTER_OTLP_ENDPOINT`, `SENTRY_DSN`. Separate environment for staging.
6. Logs: platform stdout retention plus a log drain to the observability tool for alerts on error level.
7. Everything above is written in `infra/README.md` the day it is set.

### Tier 2 (ECS Fargate on AWS, Terraform)

1. VPC with public subnets (ALB) and private subnets (tasks, RDS, ElastiCache). No public IPs on tasks.
2. ECS cluster, services `api` (autoscale on CPU 60 percent, min 2 tasks) and `worker` (min 1), task definitions referencing Secrets Manager, CloudWatch log groups with 30 day retention.
3. ALB with HTTPS listener (ACM certificate), target group health check on `/readyz`, security group allowing only Cloudflare IP ranges.
4. RDS Postgres Multi AZ, PgBouncer task or RDS Proxy, ElastiCache for Redis (two instances or two logical uses on one with care), S3 or R2.
5. ECR repository, GitHub Actions builds and pushes on push; the owner runs the deploy.
6. Terraform state in S3 with DynamoDB locking; `terraform plan` on every PR as a check.

### Tier 3 (Kubernetes)

The same containers under Helm or Kustomize with probes, HPA, PDB, external secrets, ArgoCD and canary ingress. See `migrations.md`. Nothing in application code changes between tier 1 and tier 3; that is the point of the portability rules.

## Local development

```
docker compose up -d                  # postgres:17 on 5432, redis:7 on 6379 (queue) and 6380 (cache)
cp .env.example .env                  # fill in staging identity and test mode payment keys
npm run db:migrate                    # migrations against local
npm run db:seed                       # two tenants, three users, sample data
npm run dev                           # api on 3000 with watch
npm run dev:worker                    # worker with watch
npm test                              # unit plus integration against the containers
```

The seed creates two tenants so the RLS test and every manual check can prove isolation from the first day.
