---
name: production-app
description: Plan and build apps and sites with user accounts and heavy user data the way large companies do, starting as a cheap MVP on managed providers and moving to big company infrastructure later without a rewrite. Use this whenever a new project is described, whenever someone asks which hosting, database, auth, cache, queue, storage, CDN or mobile stack to use, whenever an existing app needs to scale or move off a provider (Vercel, Supabase, Railway, Heroku, Firebase), and whenever a request touches a production concern (rate limiting, caching, security, webhooks, migrations, backups, observability), even if the words production or infrastructure never appear. Also use it to audit an existing app against the production gate.
---

# Production App

## The idea

Every large company converges on the same shape: an edge in front, a frontend deployable, a separate backend behind a gateway, workers pulling from a queue, one relational system of record, a cache, an event or job backbone, object storage, and observability around all of it. Shopify, Notion, Figma, Instacart and Airbnb differ in languages and vendors, not in shape.

The shape does not change between a weekend MVP and a company with a platform team. Only the provider behind each box does. So this skill builds the full shape on day one with cheap managed providers, enforces a short list of portability rules, and keeps a playbook per layer for swapping the provider when a measurable trigger fires. Done this way, going from Vercel and Supabase to Kubernetes and RDS is a series of config changes and data copies, never a rewrite.

## How to use this skill

1. Read the project description. Ask the intake questions below only for what it leaves out.
2. Pick a provider per layer with `references/providers-and-tiers.md`. Start at tier 1 unless a picker rule says otherwise.
3. Write `docs/PLAN.md` from the template below, including the diagram adapted from `references/architecture.md`, and show it before scaffolding.
4. Scaffold and build in the order below.
5. Before launch, run `references/launch-checklist.md` and report only failures with fixes.
6. When a trigger fires later, open `references/migrations.md` and move that one layer. Never move two layers in the same week.

For an existing app: audit first. Map the repo onto the shape, list the gaps ordered by risk (data leak, outage, abuse, lock in, cost), and stop. Changes come after the owner picks from the list.

## Intake

Ask only what the description leaves unanswered, in one message.

1. Who are the users, and what is the tenant: the unit that owns data (a company, a building, a dealer, a single person)?
2. What is heavy: feeds, messages, uploads, video, events, search? Rough rows or files per day, and growth.
3. Web, iOS, Android, or a mix? Which ships first?
4. Realtime needs: chat, live presence, collaborative editing, or is polling fine?
5. Money: subscriptions, one time payments, marketplace payouts, none?
6. Anything regulated or sensitive: payments stored, health, minors, location, government or enterprise buyers who will ask for SOC 2?
7. Audience geography: one country or global?
8. Constraints: existing cloud credits, an existing provider, a required language.

The answers drive the picker rules in `references/providers-and-tiers.md`.

## The shape

Each layer has a role, a big company reference, and the one rule that keeps it portable. Vendors per tier live in `references/providers-and-tiers.md`.

1. Edge: DNS, TLS, CDN, DDoS, WAF, rate limiting, bot filtering, in front of every host. Big companies: Cloudflare, Akamai, Fastly, CloudFront with AWS WAF. Rule: you own the DNS zone; every provider sits behind a CNAME, so swapping a provider is a DNS change.
2. Frontend: a separate deployable, static or server rendered, holding no business logic and no database access. Big companies: a React or Next.js app built into a container and served through the CDN. Rule: the frontend builds as a standard Node container or static bundle and talks only to the public API.
3. Gateway: a load balancer or API gateway that terminates TLS from the edge, routes by path, and enforces coarse limits. Big companies: ALB, Envoy, NGINX ingress, Kong. Rule: the API never depends on gateway features it cannot reproduce in its own middleware.
4. Backend: one monolith service behind the gateway, split into services only when teams collide. Big companies: Rails at Shopify and GitHub, Django at Instagram, Node at Notion, gRPC inside, GraphQL or REST outside. Rule: a Docker container with a health endpoint, config from env, no vendor SDK in business logic.
5. Workers and queue: separate long running processes consuming jobs; a workflow engine when jobs have multiple steps or human waits. Big companies: Sidekiq and Kafka consumers at Shopify, Temporal at Netflix and Duolingo, Kafka at Notion. Rule: jobs are idempotent, carry ids not blobs, and the queue is standard Redis or Kafka protocol.
6. Relational database: one Postgres (or MySQL) primary as the system of record, replicas for reads, sharding only past a single machine's limits. Big companies: RDS Postgres with PgBouncer at Notion and Figma, 100 plus MySQL shards at Shopify. Rule: Postgres wire protocol only, SQL migrations in the repo, roles and RLS in SQL, nothing that only works on one provider's API.
7. Cache: Redis or Memcached for hot reads, rate limit counters, locks, sessions. Rule: standard Redis protocol, keys tenant scoped and versioned, the app must run with the cache empty.
8. Object storage: S3 or an S3 compatible bucket for every file; never the database. Rule: the S3 API only, keys prefixed by tenant, served through signed URLs or the CDN.
9. Identity: an identity provider issuing JWTs, a `users` table owned by the app. Big companies: in house plus Okta or Auth0 for enterprise SSO. Rule: the API verifies tokens itself via JWKS and owns the users table, so the issuer can change.
10. Events and analytics: product analytics and, later, a warehouse fed by change data capture. Big companies: Kafka into Snowflake or BigQuery. Rule: the warehouse reads from the database, never dual writes from the app.
11. Observability: structured logs, errors, metrics, traces, uptime, alerts tied to a runbook. Big companies: Datadog, Grafana, Honeycomb, Sentry, OpenTelemetry. Rule: the app emits OpenTelemetry and JSON logs, so the backend is an exporter setting.
12. Delivery: CI on every push, infrastructure as code, canary or blue green deploys, feature flags, fast rollback. Big companies: Terraform, Buildkite or GitHub Actions, ArgoCD or Spinnaker, LaunchDarkly. Rule: anything clicked in a dashboard is written down in `infra/` the same day, and Terraform from tier 2.

## Portability rules

These are what make the later move cheap. They cost almost nothing at MVP size and they are the difference between a weekend migration and a quarter long rewrite.

1. Frontend and backend are separate repos and separate deployables from the first commit. They talk over a versioned HTTP API with shared schemas.
2. The backend and the worker are Docker images that run anywhere a container runs. No serverless functions for the API: Vercel functions, Lambda handlers and edge functions are the one thing that cannot be lifted into a container later without rewriting.
3. Talk to infrastructure through standards only: Postgres wire protocol, Redis protocol, the S3 API, SMTP or an HTTP email API behind an interface, JWKS for tokens, OpenTelemetry for telemetry. A provider's own client library is allowed only in a thin adapter module that can be swapped.
4. No provider feature in business logic: no PostgREST calls, no Supabase or Firebase client in the backend, no Vercel specific caching APIs, no Workers KV. The frontend may use a provider's auth SDK for sign in only.
5. The app owns its schema: SQL migrations in the repo, applied by a migration tool, never by a dashboard. Roles, RLS and functions are SQL files the repo can replay on any Postgres.
6. The app owns its users: a `users` table mirrored from the identity provider on first login, with provider ids stored as a column. Permission data lives in the app's tables.
7. Config is environment variables validated on boot. Secrets live in the platform's secret store but reach the app as env, so the store can change.
8. You own the DNS zone (Cloudflare). Every provider is a CNAME target. Nothing is pointed at a provider's nameservers.
9. Object storage keys and bucket layout are the same on R2 and S3; code uses the S3 SDK with an endpoint setting.
10. Every layer has a documented export: `pg_dump` and logical replication for the database, `rclone` for objects, a Redis dump is never needed because the cache is disposable and the queue drains.
11. Health endpoints, graceful shutdown on SIGTERM and horizontal statelessness exist from day one, because every platform from Railway to Kubernetes relies on them.
12. Telemetry and logs are OpenTelemetry and JSON from day one; vendor agents are not installed in the code.

## Tiers

Three tiers per layer. Triggers are measurable; the full table with vendors, cost and the picker rules is in `references/providers-and-tiers.md`.

* Tier 1, MVP: managed providers that take minutes to set up and cost under about $150 a month total. Frontend on Vercel, backend container on Railway, Postgres on Supabase, Redis on Railway, objects on R2, edge on Cloudflare, Sentry and Better Stack. Right until there are paying customers who ask about uptime, or the numbers below.
* Tier 2, growth: the same shape on your own cloud account where it matters: Postgres on RDS or Aurora Multi AZ with PgBouncer or RDS Proxy, containers on ECS Fargate or a small EKS, ElastiCache, S3 or R2, Terraform for everything, a read replica, Temporal for workflows. Trigger: a paying customer asks for an SLA or SOC 2, or sustained 5k monthly active users with database CPU above 60 percent, or the PaaS bill passes $500 a month, or a second engineer joins.
* Tier 3, scale: what Shopify and Notion run. Kubernetes with autoscaling, sharded or partitioned primary, Kafka as the event backbone, warehouse by CDC, multi region reads, Datadog or Grafana, a platform team. Trigger: sustained 1,000 requests per second, a single primary near 1 TB or write bound after replicas and caching, a multi region requirement, or an ops team exists to run it.

Moving tiers is per layer, not all at once. The database usually moves first, then the backend, then the frontend last, because the frontend is the cheapest to leave where it is.

## Build order

1. Schema, migrations, roles, RLS (`references/database.md`, `references/security.md`).
2. Identity verification and tenant context middleware.
3. Core API routes with validation and idempotency on writes.
4. Rate limiting and caching middleware (`references/rate-limiting.md`, `references/caching.md`).
5. Worker and queues; anything slow or external leaves the request path (`references/jobs-and-queues.md`).
6. Frontend against the API.
7. Mobile against the same API.
8. Logs, metrics, traces, health checks, alerts (`references/observability-and-reliability.md`).
9. Launch gate, load test at three times expected peak, one restore test (`references/launch-checklist.md`).

## PLAN.md template

```
# <Project> Plan
## Product in three sentences
## Users, tenant, roles
## Providers by layer         (tier per layer, why, the trigger that moves it, from providers-and-tiers.md)
## Data model                 (tables, tenant_id marked, indexes, partitioned tables)
## API surface                (routes by resource, auth level, rate limit tier)
## Background jobs            (queue, trigger, idempotency key, retry policy)
## Third parties              (service, purpose, secret name, limit, failure mode)
## Caching map                (what, key, TTL, which writes invalidate it)
## Threat model               (top 5 abuse cases and the control for each)
## Scale assumptions          (users, rows, requests per minute at launch and at 10x)
## Cost at launch             (per provider, per month)
## Exit plan per provider     (the migrations.md playbook that applies, in one line each)
## Diagram
```

## Non negotiable rules

1. Every tenant owned table has `tenant_id`; RLS enabled and forced; the API sets the tenant per transaction from verified membership, never from request input.
2. Clients never touch the database or hold privileged keys. Everything goes through the API.
3. Every input validated at the boundary with unknown keys rejected; every query parameterized.
4. The API and workers connect through a pooler with small pools; statement timeouts set per role.
5. Rate limits at the edge and in the API, keyed per user, tenant and IP. Cache keys tenant scoped and versioned.
6. Secrets in the platform store, one set per environment, never in git or a mobile bundle. Production and staging are separate projects and accounts.
7. PITR on and a restore tested once.
8. Webhooks signature verified, replay protected, idempotent, answered in under two seconds.
9. Timeouts on every outbound call; retries with jitter only on idempotent work; failed jobs visible to a human.
10. Request id and tenant id on every log line; alerts on error rate, p95 latency, 5xx count, queue age, failed jobs, database connections.

## Mobile

1. Existing React Native or Expo app: stays.
2. New app, iOS only or iOS first: native Swift and SwiftUI. Large companies went native in 2026 (Shopify back to Swift and Kotlin, Notion to SwiftUI) because coding agents removed the cost of the second codebase; the same holds for a small team. Kotlin only when Android is actually built.
3. New app that must ship iOS and Android on the same day with one or two people: Expo.
4. Either way the app talks only to the API, keeps tokens in secure storage, and ships nothing secret in the bundle.

## Reference files

* `references/providers-and-tiers.md`: the per layer table (big company practice, tier 1, 2, 3 vendors, trigger, cost) and the picker rules that turn a project description into choices.
* `references/migrations.md`: the step by step playbook for moving each layer up a tier.
* `references/architecture.md`: the diagram with tier labels, the three request paths, repo layouts, environments, edge and platform configuration.
* `references/security.md`: tenancy and RLS, identity, secrets, headers, input, uploads, webhooks, encryption, deletion and export, mobile, audit log, incidents.
* `references/rate-limiting.md`: four layers with numbers and the middleware.
* `references/caching.md`: every cache layer, key schema, TTLs, invalidation, stampede protection.
* `references/database.md`: schema rules, roles, pooling, migrations, backups, performance, SQL appendix.
* `references/jobs-and-queues.md`: queues, job design, retries, cron, backpressure, when to add a workflow engine or Kafka.
* `references/observability-and-reliability.md`: logs, errors, metrics, traces, health, alerts, reliability patterns, testing, runbook.
* `references/launch-checklist.md`: the gate.
