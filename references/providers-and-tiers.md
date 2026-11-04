# Providers and tiers

One table per layer: what large companies actually run, then the provider at each tier, the trigger that moves it, and the rough monthly cost. After the tables, the picker rules that turn a project description into choices. Prices are 2026 ballparks; confirm on the provider's page before putting a number in PLAN.md.

## 1. Edge: DNS, CDN, WAF, DDoS

```
big companies   Cloudflare (Shopify, Discord, DoorDash), Akamai, Fastly, CloudFront + AWS WAF + Shield
tier 1          Cloudflare Free or Pro ($20)          DNS for every host, proxy on the API host, WAF, 2 rate rules
tier 2          Cloudflare Business ($200)            100 custom WAF rules, 5 rate rules, 100 percent uptime SLA, Access
tier 3          Cloudflare Enterprise, or CloudFront + AWS WAF when everything is on AWS and procurement wants one vendor
trigger         Pro to Business: need more than 2 rate rules or an SLA. Enterprise: contract requirement or Magic Transit
cost            $0 to $20 / $200 / custom
note            Cloudflare stays through every tier for most companies; it is the layer that almost never moves
```

## 2. Frontend hosting

```
big companies   Next.js or React in a Docker container on their own Kubernetes behind their CDN; Vercel for marketing sites,
                storefronts and some whole products (OpenAI, Under Armour, Perplexity, Nike's store, Washington Post)
tier 1          Vercel Pro ($20 per seat)              zero config Next.js, previews, image optimization
tier 2          Vercel Pro still, or the Next.js container on the same ECS or EKS as the API when the team wants one platform
tier 3          Next.js container on Kubernetes behind the CDN, or Vercel Enterprise if the frontend team is small
trigger         Vercel bill above $500 a month from bandwidth or functions, compliance says all compute in your account,
                or you need the frontend inside a private network
cost            $20 / $20 to $500 / $500 plus or container cost
note            Keep the frontend portable with the standalone output (next.config output: 'standalone') and no Vercel
                only APIs; then the move is a Dockerfile and a CNAME
```

## 3. Gateway and load balancing

```
big companies   ALB or NLB, Envoy, NGINX ingress, Kong; service mesh (Istio, Linkerd) only at hundreds of services
tier 1          the PaaS router (Railway, Fly, Render) plus Cloudflare in front; nothing to run
tier 2          ALB in front of ECS or EKS, with target group health checks on /readyz
tier 3          ALB plus NGINX or Envoy ingress on Kubernetes; API gateway product only if selling a public API
trigger         moves with the backend, it is not chosen separately
cost            $0 / about $20 to $40 per ALB / included in cluster
```

## 4. Backend compute

```
big companies   containers on Kubernetes (GKE at Shopify, EKS at Notion and Figma), a monolith for years, services later;
                deploys by canary with feature flags and instant rollback
tier 1          Railway ($5 base plus usage, typically $20 to $60)   Docker image, env vars, SIGTERM, health check, logs
                alternatives by rule: Fly.io when latency to a global audience matters, Render when PR previews matter
tier 2          ECS Fargate on AWS (or Cloud Run on GCP)            same container, Terraform, ALB, autoscaling, private VPC
tier 3          EKS or GKE with Karpenter or node autoscaling, ArgoCD, canary via the ingress
trigger         tier 1 to 2: a customer asks for SOC 2 or an SLA, AWS credits available, need private networking to the
                database, or the Railway bill passes $300. tier 2 to 3: more than about 10 services, or sustained 1k rps,
                or an ops person exists
cost            $20 to $60 / $150 to $600 / $1,500 plus and a person
note            never put the API in serverless functions; containers lift to every tier, functions do not
```

## 5. Workers, queues, workflows, events

```
big companies   Sidekiq or Celery style queues on Redis for simple jobs; Temporal for multi step workflows
                (Netflix, Duolingo, Coinbase, most fintechs); Kafka as the event backbone (Shopify at 66M msgs/s peak,
                Notion on Confluent Cloud); SQS where everything is AWS native
tier 1          BullMQ on a Redis instance set to noeviction, one worker container on Railway
tier 2          BullMQ still for jobs; Temporal Cloud (from about $100 a month) or self hosted Temporal when workflows have human waits,
                multi day timers or more than 3 steps that must survive failure; SQS plus a consumer if already deep in AWS
tier 3          Temporal for workflows, Kafka (MSK or Confluent) for events consumed by more than 3 services, CDC to the
                warehouse through Debezium or a managed connector
trigger         BullMQ to Temporal: the first workflow that must resume after a crash mid flow, or any approval or drip
                flow. Kafka: the first time three or more services need the same event stream, or CDC to a warehouse
cost            $10 (Redis) / $100 plus / $500 plus
```

## 6. Relational database

```
big companies   Postgres on RDS with PgBouncer (Notion, Figma, Instacart), Aurora, Cloud SQL or AlloyDB on GCP,
                MySQL with Vitess or app level sharding (Shopify 100 plus shards in pods, GitHub, Slack); self hosted with
                Patroni only with a DBA team (GitLab, Adyen, Cloudflare)
tier 1          Supabase Pro ($25 plus compute)      Postgres, Supavisor pooler, daily backups, PITR add on ($100), Auth bundled
                alternative by rule: Neon ($19 plus) when preview branches per PR matter more than bundled auth
tier 2          RDS Postgres Multi AZ (db.m7g.large about $250) plus RDS Proxy or PgBouncer on a small instance,
                PITR built in, one read replica when reads dominate; Aurora Serverless v2 for spiky loads
tier 3          Aurora or RDS at the largest instance with replicas, then partitioning, then sharding (Citus, Vitess for
                MySQL, or application sharding by tenant like Notion's 480 logical shards)
trigger         tier 1 to 2: paying customers need automatic failover (Supabase requires Enterprise for it), or compute
                above 16 cores with queries already indexed and cached, or compliance wants the data in your account.
                tier 2 to 3: primary near 1 TB or write bound after replicas and caching, or multi region reads required
cost            $25 to $125 / $300 to $1,500 / $3,000 plus
note            Postgres on every tier; the move is logical replication, not a dump and restore
```

## 7. Cache

```
big companies   Redis (ElastiCache, Memorystore) and Memcached; Shopify runs one Redis and one Memcached cluster per pod
tier 1          Railway Redis ($10) with allkeys-lru, separate from the queue Redis
tier 2          ElastiCache for Redis (cache.t4g.small about $25) in the VPC, cluster mode off, replica for failover
tier 3          ElastiCache cluster mode on, or Memcached for pure cache at very high read rates
trigger         moves with the backend so it stays on the same private network; cluster mode past one node's memory
cost            $10 / $25 to $100 / $300 plus
```

## 8. Object storage

```
big companies   S3 (everyone on AWS), GCS, R2 for egress heavy public assets
tier 1          Cloudflare R2 ($0.015 per GB, zero egress)   S3 API, presigned uploads, custom domain for public assets
tier 2          R2 still, or S3 when the backend moves into AWS and private VPC access matters
tier 3          S3 with lifecycle tiers, or R2 for anything served to the public at volume
trigger         only moves when egress or network locality makes it cheaper; usually it never moves
cost            under $10 / $20 to $200 / usage
note            the S3 SDK with an endpoint setting covers both; keep rclone configured for a one command copy
```

## 9. Identity

```
big companies   in house users and sessions plus Okta, Auth0 or WorkOS for enterprise SSO; Cognito on AWS shops;
                Firebase Auth on consumer apps on GCP
tier 1          Supabase Auth (included) issuing JWTs; the API verifies via JWKS and mirrors users into its own table
                alternative by rule: Clerk when the product needs polished hosted sign in UI and B2B org management today
tier 2          keep Supabase Auth even after the database moves (it is a separate service), or move to Auth0 or Cognito
                when enterprise SSO (SAML, SCIM) is sold; WorkOS for SSO only
tier 3          in house identity service or Auth0 or Okta CIC with SSO, MFA policies, audit
trigger         the first enterprise deal that requires SAML or SCIM, or MFA policy requirements, or 1M plus users
cost            $0 to $25 / $100 to $500 / $1,000 plus
note            because the API only trusts JWKS and owns the users table, changing issuer is a config change plus a
                one time password hash export or a forced reset
```

## 10. Email and messaging

```
big companies   SendGrid, Amazon SES, Postmark; Twilio for SMS; own IP pools at volume
tier 1          Resend ($20)                        transactional email, also wired as the auth provider's SMTP
tier 2          Amazon SES ($0.10 per thousand) when the backend lives in AWS and volume passes 100k a month
tier 3          SES or SendGrid with dedicated IPs and a deliverability owner
trigger         volume or cost; behind an interface the swap is one adapter
cost            $20 / $20 to $100 / $200 plus
```

## 11. Payments

```
big companies   Stripe for cards and subscriptions, Adyen or Braintree at very high volume, Stripe Connect for marketplaces
tier 1          Stripe Checkout, Customer Portal, webhooks, subscriptions mirrored to a table
tier 2          same, plus Stripe Tax, Revenue Recognition, and a daily reconcile job
tier 3          same Stripe account with volume pricing, or Adyen when processing fees dominate
trigger         fees; Stripe does not get outgrown by most companies
```

## 12. Search

```
big companies   Elasticsearch or OpenSearch, Algolia for product search, Typesense or Meilisearch at smaller scale
tier 1          Postgres (pg_trgm, tsvector) until search is a core feature
tier 2          Typesense Cloud or Meilisearch when relevance and typo tolerance matter, fed by the worker
tier 3          OpenSearch or Elasticsearch cluster, fed by CDC
trigger         search is on the main screen and Postgres p95 passes 200ms, or faceting and typo tolerance are required
cost            $0 / $30 to $200 / $500 plus
```

## 13. Analytics and warehouse

```
big companies   Snowflake or BigQuery fed by Kafka or CDC (Notion: Kafka to S3 and Snowflake); ClickHouse for
                product analytics at scale; PostHog, Amplitude or Mixpanel for product events
tier 1          PostHog Cloud (free tier) for product events; Postgres materialized views for dashboards
tier 2          ClickHouse Cloud or BigQuery for reporting when aggregates hurt the primary; fed by CDC, never dual writes
tier 3          Snowflake or BigQuery as the warehouse, Kafka or Debezium CDC, dbt for models
trigger         reporting queries over more than 100M rows, or the finance team asks for a warehouse
cost            $0 / $100 to $500 / $2,000 plus
```

## 14. Observability

```
big companies   Datadog (most), Grafana Cloud or self hosted Prometheus and Loki, Honeycomb, Sentry for errors,
                OpenTelemetry as the emission standard, PagerDuty for on call
tier 1          Sentry (free or $26), Better Stack (free) for uptime, logs and status page, platform logs
tier 2          Grafana Cloud (free tier is generous) or Datadog at about $15 per host plus logs; PagerDuty or Better Stack on call
tier 3          Datadog or Grafana stack with SLOs, traces sampled, log pipelines, PagerDuty rotations
trigger         more than 3 services, or the need to trace a request across worker and API, or an on call rotation
cost            $0 to $30 / $100 to $400 / $1,000 plus
note            because the app emits OpenTelemetry, each move is an exporter endpoint change
```

## 15. CI, infrastructure as code, deploys

```
big companies   GitHub Actions or Buildkite for CI, Terraform or Pulumi for infra, ArgoCD or Spinnaker for CD,
                canary deploys, feature flags (LaunchDarkly, Statsig, in house), merge queues
tier 1          GitHub Actions for typecheck, lint, tests, secret grep; provider dashboards documented in infra/README.md;
                feature flags as a table in Postgres
tier 2          Terraform for every AWS resource in the repo's infra/ folder, GitHub Actions runs plan on PRs,
                blue green on ECS, flags still in Postgres or a hosted flag service
tier 3          Terraform modules, ArgoCD, progressive delivery, LaunchDarkly or Statsig, merge queue
trigger         Terraform the day the first AWS resource is created; never later
cost            $0 / $0 to $100 / $500 plus
```

## 16. Mobile

```
big companies   native Swift and Kotlin (Shopify since September 2026, Notion, Airbnb, Uber, Robinhood);
                React Native at Discord, Coinbase, Microsoft, Meta surfaces; Flutter at Google Pay and some banks
tier 1          Swift and SwiftUI for iOS only or iOS first; Expo when iOS and Android ship the same day with one or two people
tier 2 and 3    native for both platforms; shared business logic through the API, not through a shared mobile codebase
trigger         none; the choice is made at the start and an existing app is not rewritten for its own sake
```

## Tier totals

```
tier 1 MVP      $60 to $150 a month for everything above, set up in a day
tier 2 growth   $500 to $3,000 a month, two to four weeks of migration work spread over a quarter
tier 3 scale    $10,000 plus a month and at least one person whose job is the platform
```

## Picker rules

Apply in order. Each rule overrides the tier 1 default for one layer only.

1. Regulated data (health, payments stored, government buyers) or a signed enterprise deal pending: start the database and backend at tier 2 on AWS, Terraform from the first resource. Everything else stays tier 1.
2. AWS Activate or GCP credits above $10k: backend and database at tier 2 on that cloud from day one; the PaaS platforms cannot take the credits.
3. Global audience with latency as part of the product (chat, gaming, trading): backend on Fly.io in 3 regions at tier 1, Cloudflare in front, read replicas per region at tier 2.
4. Heavy uploads or video: R2 with direct presigned uploads from day one, a dedicated `media` queue with concurrency 2, and transcoding in the worker (or Mux when video is the product).
5. Search on the main screen: Typesense at tier 1 instead of waiting for the Postgres trigger.
6. Realtime collaboration or chat as the core feature: a WebSocket service as its own container next to the API, Redis pub/sub for fan out, and sticky sessions at the gateway; polling for everything else.
7. Multi step workflows with human waits (approvals, onboarding over days, payouts): Temporal from the start, not BullMQ with timers.
8. Marketplace with payouts: Stripe Connect from the start; retrofitting Connect onto plain Stripe is a migration of its own.
9. B2B product selling to companies with IT departments: identity that can do SAML and SCIM within a quarter (WorkOS alongside Supabase Auth, or Clerk Organizations, or Auth0), and an audit log on day one.
10. Preview environment per pull request is important to the team: Neon for the database (branching) and Render or Vercel for the apps.
11. iOS only or iOS first: Swift. Both platforms on the same day with one or two people: Expo. Web only: no mobile layer, do not add one.
12. Expected rows above 1M a month on any table: partition that table from the first migration and plan ClickHouse at tier 2.
13. Any LLM feature: a per tenant daily budget in the rate limiter, an `ai` queue with a provider rate limiter, and cost logged per request from day one.
14. Nothing above applies: the tier 1 column in every table, exactly as written.

Write the chosen provider, the rule that chose it, and the trigger that will move it, per layer, into PLAN.md under Providers by layer.
