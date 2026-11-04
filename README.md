![production-ready-app](assets/banner.png)

A Claude skill for building apps the way startups that made it do: ship an MVP on cheap managed providers in a day, but on the same architecture the big companies run, so when the users show up you move to real infrastructure one layer at a time instead of rewriting.

The idea in one line: the shape never changes between a weekend MVP and Shopify. Only the provider behind each box does.

## What it does

Install it, then describe an app to Claude Code. The skill takes over the infrastructure decisions:

1. Asks only the intake questions your description left out: who the tenant is, what data is heavy, web or iOS or both, realtime needs, how money moves, anything regulated, audience geography, existing credits.
2. Picks a provider for every layer from the tier tables, starting at the MVP tier unless a picker rule overrides it (regulated data goes to AWS on day one, a global chat app goes multi region, a marketplace starts on Stripe Connect, and so on).
3. Writes `docs/PLAN.md`: the diagram, the data model with tenant isolation marked, the API surface with a rate limit tier per route, background jobs, the caching map, a threat model, scale assumptions, cost at launch, and an exit plan per provider.
4. Builds in a fixed order so nothing gets skipped: schema and row level security first, then identity and tenant context, then routes with validation and idempotency, then rate limiting and caching, then the worker and queues, then the web frontend, then mobile, then logs, metrics, traces and alerts.
5. Runs a 140 item launch gate before anything goes live and reports only the failures, each with its fix. Tenancy, auth, secrets, backups and portability failures block launch outright.
6. Later, when a measurable trigger fires (a customer asks for an SLA, database CPU stays above 60 percent, the PaaS bill passes $500), it follows the playbook for that one layer and moves it, with a rollback and a verification step.

For an app that already exists it audits first: maps the repo onto the shape, lists the gaps ordered by risk, and stops until you pick what to fix.

## The shape

Twelve layers, each with a rule that keeps it portable:

* Edge: Cloudflare in front of every host. DNS, TLS, CDN, DDoS, WAF, rate limiting, bot filtering, Turnstile, Tunnel. You own the DNS zone, so every provider is a CNAME.
* Frontend: a separate deployable with no business logic and no database access. Builds as a standard container even while hosted on Vercel.
* Gateway: the platform router at first, a load balancer and ingress later.
* Backend: one monolith in a Docker container behind the gateway, never in serverless functions.
* Workers and queue: a second container consuming jobs from Redis, a workflow engine when jobs get multi step.
* Relational database: Postgres with row level security forced on every table, reached only through a transaction mode pooler.
* Cache: a second Redis instance, keys scoped to the tenant and versioned.
* Object storage: the S3 API, tenant prefixed keys, presigned uploads, signed downloads.
* Identity: a provider issuing JWTs that the API verifies itself against an issuer list, with a users table the app owns.
* Events and analytics: product analytics first, a warehouse fed by change data capture later, never dual writes.
* Observability: OpenTelemetry and JSON logs from the app, so the tool that receives them is an exporter setting.
* Delivery: CI on every push, infrastructure as code from the first cloud resource, feature flags, fast rollback.

## Tiers

* Tier 1, MVP: Vercel, Railway, Supabase, two Redis instances, Cloudflare R2, Cloudflare, Sentry, Better Stack. Under about $150 a month, set up in a day.
* Tier 2, growth: RDS or Aurora Multi AZ with PgBouncer, ECS Fargate, ElastiCache, Terraform, a read replica, Temporal for workflows. $500 to $3,000 a month. Triggered by an SLA request, sustained database load, a $500 PaaS bill, or a second engineer.
* Tier 3, scale: Kubernetes, a sharded or partitioned primary, Kafka as the event backbone, a warehouse by CDC, Datadog or Grafana, a platform team. Triggered by sustained 1,000 requests per second, a primary near 1 TB, or a multi region requirement.

Layers move one at a time. The database usually goes first, the backend second, the frontend last or never.

## What makes the move cheap

Twelve portability rules enforced from the first commit, the most important being: frontend and backend in separate repos, the API and worker as Docker images, infrastructure reached only through standard protocols (Postgres wire, Redis, S3 API, JWKS, OpenTelemetry), vendor SDKs confined to one adapters folder, the schema and the users table owned by the app, config as environment variables, your own DNS zone, and a documented export for every layer. The launch gate checks all twelve.

## What is in the files

* `SKILL.md`: the workflow, the shape, the portability rules, the tiers, the build order, the PLAN.md template, the mobile rule.
* `references/providers-and-tiers.md`: sixteen layers, each with what big companies run, the provider at tiers 1, 2 and 3, the trigger that moves it, and rough cost, followed by fourteen picker rules.
* `references/migrations.md`: the step by step playbook for every move. Vercel to a container, Railway to ECS to Kubernetes, Supabase to RDS by logical replication with the exact SQL and a five minute cutover, cache and queue Redis, BullMQ to Temporal, adding Kafka through an outbox table, R2 to S3, identity provider changes with the password hash export, plus a four week calendar for a full tier 1 to tier 2 move.
* `references/architecture.md`: the diagram with tier labels, the three request paths (read, write, ingestion) step by step, repo layouts, environments, the Cloudflare configuration including the zero exposure Tunnel setup, Railway and ECS configuration, local development.
* `references/security.md`: tenancy and RLS with the SQL, identity verification, secrets, headers and CORS, input validation, uploads, inbound webhooks, outbound calls, encryption and data rights, dependencies, mobile, the audit log, incident basics.
* `references/rate-limiting.md`: four layers (edge, API, database, outbound) with default numbers per route tier, plan multipliers, concurrency caps, cost based budgets, abuse controls, and the middleware code.
* `references/caching.md`: seven cache layers from the browser to Postgres, the key schema with tenant versioning, TTLs, stampede protection with the helper code, what never gets cached, the caching map every plan must fill in.
* `references/database.md`: how to design the schema in nine steps, the schema rules, roles and pooling, expand and contract migrations, backups and restore tests, performance hygiene, scale triggers, and a SQL appendix.
* `references/jobs-and-queues.md`: queue setup, job design, retries and dead letters, priorities and fairness, cron, backpressure, graceful shutdown, when to add Temporal or Kafka.
* `references/observability-and-reliability.md`: logging, errors, traces, metrics, health endpoints, alert thresholds, reliability patterns, testing including load and failure drills, the runbook.
* `references/launch-checklist.md`: the gate, grouped by tenancy, auth, secrets, network, API hardening, rate limiting, caching, database, jobs, storage, billing, mobile, web, observability, backups, legal, portability, and load tests.

## Install

```
git clone https://github.com/ArvinAbrahamian1/production-ready-app ~/.claude/skills/production-ready-app
```

Then describe a project to Claude Code. The skill loads on its own whenever hosting, database, auth, caching, rate limiting, queues or scaling come up. You can also point it at an existing repo and ask for an audit.

Example:

```
I'm building a marketplace for apartment residents to sell furniture to each other. Buildings are the accounts, residents sign up with a code from their building, listings have photos, buyers message sellers, iOS first, web later, no payments at launch.
```

From that it will pick the tier 1 providers, note that photos mean R2 presigned uploads and a media queue, that messaging means a WebSocket service next to the API, that iOS first means Swift, and write the plan before touching code.

## Notes

Prices and plan limits in the provider tables are November 2026 ballparks. Check the provider's page before relying on one.

The picker rules and tier triggers are opinionated on purpose: one answer per layer, so the skill produces a plan instead of a list of options. If you disagree with a number, open an issue with what you would set it to and why.

## License

MIT
