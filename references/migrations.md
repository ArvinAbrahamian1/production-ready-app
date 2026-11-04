# Migrations between tiers

One playbook per layer. Each assumes the portability rules in SKILL.md were followed; where they were not, the first step is to fix that in place before moving anything. Move one layer at a time, verify, wait a week, move the next.

## Ground rules for every move

1. Lower the DNS TTL on the affected host to 60 seconds a day before. Raise it back a day after.
2. Write the rollback before the cutover: the exact command or DNS change that puts traffic back, and how long it is valid (for a database, until the old primary is more than a few minutes stale).
3. Dual run where the layer allows it (two backends behind the edge, two issuers trusted by the API, two buckets read from). Cut writes last.
4. Verify with the same checks every time: `/readyz` on the new target, the k6 smoke script, the RLS test against the new database, error rate and p95 in the dashboard for an hour.
5. Record the move in `docs/RUNBOOK.md` with dates, durations and what surprised you.

The usual order when a project moves whole: database first (it is the only layer with state that cannot be rebuilt), then backend and worker (so they sit on the same private network as the database), then cache and queue (they move with the backend), then object storage only if it needs to, then frontend last or never.

## Frontend: Vercel to a container on your cloud

Only when the trigger fires (bandwidth bill, compliance, private network). Vercel is a fine place to leave a frontend for years.

1. `next.config` has `output: 'standalone'` and no Vercel only APIs (check for `@vercel/*` imports, `unstable_cache` tied to Vercel's data cache, and ISR that assumes Vercel's shared cache).
2. Dockerfile: multi stage build, copy `.next/standalone`, `.next/static` and `public`, run `node server.js` on port 3000 as a non root user.
3. If more than one instance runs: set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` the same on every instance, point the Next.js cache handler at Redis (a custom `cacheHandler` file) so ISR invalidation reaches every instance, and set `deploymentId` so rolling deploys do not mix asset versions.
4. Deploy to the same ECS or EKS as the API, behind the ALB on a separate target group, host based routing for `app.domain`.
5. Cloudflare: change the `app` CNAME from Vercel to the ALB, turn the proxy on now (Vercel's objection to proxies no longer applies), add a cache rule for `/_next/static/*` with a one year edge TTL.
6. Image optimization: `next/image` works with the built in loader in the container, but the CDN should cache the output; or switch the loader to Cloudflare Images.
7. Verify: Lighthouse on the main pages, ISR revalidation across two instances, previews now come from a staging deployment instead of Vercel preview URLs.

## Backend and worker: Railway to ECS Fargate (tier 2)

1. The service already has a Dockerfile, `/healthz`, `/readyz`, SIGTERM handling and env based config. If anything is missing, add it on Railway first and deploy there once.
2. Terraform in `infra/`: VPC with private subnets, ALB in public subnets, ECS cluster, task definitions for `api` and `worker`, security groups (ALB to api only, api and worker to the database and Redis only), Secrets Manager entries referenced by the task definitions, CloudWatch log groups, autoscaling on CPU 60 percent for `api`.
3. Build and push the image to ECR from GitHub Actions on push (build only; the deploy itself is a task definition update run by the owner).
4. Point the new tasks at the current database and Redis (over TLS; Supabase and Railway Redis are reachable from AWS) so the backend moves alone.
5. Dual run: Cloudflare Load Balancer (or two origins with weights) sending 10 percent to the ALB, then 50, then 100 over a day, watching error rate and p95.
6. Scale Railway to zero but keep the service for a week as the rollback target. Then delete it, along with the `cloudflared` service: on ECS the ALB security group allowlists Cloudflare's IP ranges instead, or keep a Tunnel as an ECS sidecar if you prefer no public ALB at all.
7. Worker: run it on ECS only; two workers on the same queue would double process repeatable jobs unless the leader lock is already in place (it is, from `jobs-and-queues.md`).

## Backend: ECS to Kubernetes (tier 3)

Only with an ops person. The container does not change.

1. EKS with managed node groups or Karpenter, the AWS Load Balancer Controller for ALB ingress, external secrets operator pointed at Secrets Manager, metrics server and HPA on CPU and request rate.
2. Helm chart or Kustomize for `api` and `worker` with the same env, probes on `/healthz` and `/readyz`, `terminationGracePeriodSeconds: 45`, PodDisruptionBudget of one.
3. ArgoCD watching the repo's `infra/k8s` folder; canary through the ingress (10, 50, 100 percent) with automatic rollback on error rate.
4. Cut over at the ALB: the Kubernetes ingress gets its own ALB; Cloudflare CNAME moves from the ECS ALB to the ingress ALB.

## Database: Supabase (or Neon) to RDS Postgres (tier 2)

Logical replication, not dump and restore, so the cutover is minutes and the old database stays as a rollback for a week.

1. RDS Postgres Multi AZ on the same major version, in the private subnet, parameter group with `rds.logical_replication = 1`, PITR on, a PgBouncer task on ECS (or RDS Proxy) in transaction mode.
2. Replay the schema on RDS from the repo: roles SQL, then `drizzle-kit migrate`, then the RLS and function SQL files. Diff the result against production with `pg_dump --schema-only` on both and compare.
3. On the source, as the owner role:

```sql
create publication app_pub for all tables;
```

4. On RDS, as the owner role (the subscription initial copy brings existing rows, then streams changes):

```sql
create subscription app_sub
  connection 'host=db.<ref>.supabase.co port=5432 dbname=postgres user=postgres password=<pw> sslmode=require'
  publication app_pub;
```

5. Wait until `select * from pg_stat_subscription;` shows the initial sync done and lag near zero. Compare row counts per table on both sides. Sequences are not replicated: after cutover, run `select setval(...)` for every serial sequence to the source's current value plus a margin (UUID v7 keys mean few or none).
6. Run the RLS test and the integration tests against RDS with the pooled URL.
7. Cutover window (under five minutes): set the app to maintenance or pause writes (feature flag `writes_paused`), confirm lag is zero, drop the subscription on RDS, point `DATABASE_URL` at PgBouncer in front of RDS, restart api and worker, unpause writes.
8. Rollback within the window: point `DATABASE_URL` back. After the window: reverse replication would be needed, so treat the move as final once writes resume on RDS.
9. Supabase Auth keeps working: it lives in a separate service and schema and the API verifies JWTs via JWKS. The `users` mirror table on RDS is already fed by first login. Decide separately whether identity moves (see Identity below).
10. A week later: snapshot and delete the Supabase database compute, keep Auth if still in use.

## Database: RDS to Aurora, replicas, partitioning, sharding (tier 3)

1. Read replica: one Terraform change; route only reports and heavy lists to it through a second connection string, never auth or writes.
2. Aurora: migrate by snapshot restore into an Aurora cluster or by logical replication exactly as above; the app sees a new endpoint.
3. Partitioning an existing table: create the partitioned table, copy in batches from the worker, swap names in one transaction, keep the old table for a week.
4. Sharding: Citus for Postgres keeps SQL; application sharding by tenant the way Notion did (logical shards as schemas, a routing table, move shards between instances) keeps vendors out of it. Either is a quarter of work and needs a DBA for the design review. Do not start this before replicas, caching and partitioning are exhausted.

## Cache: Railway Redis to ElastiCache

1. ElastiCache for Redis in the private subnet, same major version, `maxmemory-policy allkeys-lru`, one replica.
2. Change `REDIS_CACHE_URL`, restart api and worker. The cache warms itself; the `cache.warm` cron job front loads the top tenants.
3. Rate limit counters reset at the cutover, which is harmless. Idempotency keys reset too, so do the cutover in a quiet window to avoid a duplicate from a client retry across the switch.
4. Delete the old instance after a day.

## Queue: Railway Redis to ElastiCache, BullMQ to Temporal, adding Kafka

Queue Redis move:

1. New ElastiCache instance with `noeviction` and AOF.
2. Pause producers (feature flag), let the worker drain the old queue to empty, confirm `waiting` and `active` are zero.
3. Change `REDIS_QUEUE_URL` on api and worker, restart, unpause producers. Repeatable jobs are re registered by the worker on boot.

BullMQ to Temporal (only for the workflows that need it; simple jobs stay on BullMQ):

1. Temporal Cloud namespace or self hosted Temporal in the cluster; a `workflows` worker container with the Temporal SDK.
2. Rewrite one workflow at a time as a Temporal workflow with activities that call the same handlers BullMQ used. Activities stay idempotent.
3. Dual run: new instances of that workflow start on Temporal; in flight BullMQ jobs finish on BullMQ. Remove the BullMQ producer when the old queue is empty.

Adding Kafka (tier 3, when three or more services need the same events, or for CDC):

1. MSK or Confluent Cloud, one topic per event type, schema registry from day one.
2. The API publishes after commit through the outbox pattern: write the event into an `outbox` table in the same transaction, a worker publishes and marks it sent. Never publish before the commit.
3. CDC to the warehouse with Debezium or the managed Postgres connector reading the WAL, so the app never dual writes.

## Object storage: R2 to S3 (or the reverse)

Usually unnecessary. When it is:

1. New bucket with the same key layout, CORS and lifecycle rules from Terraform.
2. `rclone sync` from old to new for the bulk copy (hours to days depending on size), run from a worker or a bastion.
3. Dual read in the storage adapter: try new bucket, fall back to old, and copy on read.
4. Switch writes to the new bucket, run a final `rclone sync` for anything written during the copy, then remove the fallback after a week.
5. Public asset domain: move the CNAME; the CDN cache covers the gap.

## Identity: Supabase Auth to Auth0, Cognito or in house

1. The API already verifies via JWKS and owns `users`. Add the new issuer as a second trusted issuer in the verifier; both token types work during the transition.
2. Export users: on Supabase, `select id, email, encrypted_password, email_confirmed_at, raw_user_meta_data from auth.users;` gives bcrypt hashes. Auth0 and Cognito both accept bcrypt imports (Cognito through a migration Lambda on first login instead). In house: store the hashes and verify with bcrypt on first login, then rehash with argon2.
3. Social logins (Google, Apple) carry over by matching on the provider subject id stored in `users`; Apple sign in needs the team's transfer process if the app's bundle moves teams, otherwise nothing.
4. Clients switch their sign in SDK to the new provider in a release; old sessions remain valid until their refresh tokens expire. For mobile, force a sign in after two release cycles.
5. Remove the old issuer from the verifier after 30 days, then disable the old provider.

## Email and messaging

The adapter interface (`send(template, to, data)`) is the whole surface. New provider behind the adapter, domain verified (SPF, DKIM, DMARC), 10 percent of sends through the new provider for a day watching bounces and spam complaints, then 100 percent. Keep the old provider's API key for a week.

## Observability

1. New backend (Datadog, Grafana Cloud): create the OTLP endpoint and API key.
2. Change `OTEL_EXPORTER_OTLP_ENDPOINT` and headers on api and worker; logs ship through the same collector or the platform's log drain.
3. Recreate the alert rules from `observability-and-reliability.md` in the new tool before switching the on call route. Run both for a week.

## Edge: Cloudflare to CloudFront plus AWS WAF (rare)

Only when procurement forces a single vendor. Cloudflare in front of AWS is the common and better configuration.

1. CloudFront distribution per host with the ALB as origin, AWS WAF web ACL with the managed core rule set and rate based rules matching `rate-limiting.md` layer 1, Shield Standard is automatic.
2. Certificates in ACM, Route 53 for DNS (the zone moves, which is the painful part; export the zone file from Cloudflare, import, lower TTLs a week ahead).
3. Turnstile is replaced by a WAF challenge action or reCAPTCHA Enterprise; Access is replaced by ALB authentication with Cognito or an IAM Identity Center app.
4. Switch nameservers at the registrar last, after every record has been verified on Route 53.

## What a full tier 1 to tier 2 move looks like on a calendar

```
week 1   Terraform skeleton, VPC, RDS, ElastiCache, ECR, ECS cluster; nothing live
week 2   backend and worker on ECS against the existing database and Redis, 10 then 100 percent of traffic
week 3   logical replication to RDS, tests against RDS, cutover in a quiet window; cache and queue Redis cut over
week 4   observability exporters, alert rules, runbook updates, delete old resources after the rollback window
```

The frontend stays on Vercel through all four weeks and usually after.
