# Observability and reliability

You cannot fix what you cannot see, and the rule for debugging is to verify the cause before changing code. That only works if the logs, errors, metrics and traces exist before the incident. The app emits OpenTelemetry and JSON logs from day one so the tool that receives them can change by tier without touching code. Build them in phase 4 step 8, before launch, not after the first outage.

## Logging

1. Pino, JSON to stdout, one line per request and per job. The platform keeps the stream; a log drain ships it to the observability tool (Better Stack at tier 1, Datadog or Grafana Loki later) for search and alerts.
2. Fields on every line: `reqId`, `tenantId`, `userId`, `route`, `method`, `status`, `durationMs`, `cache` (hit, stale, miss, none), `jobId` and `attempt` in the worker, `release`.
3. Request id: take `cf-ray` when present, else generate a UUID. Return it as `x-request-id`. The web and iOS error screens show it ("something went wrong, code 8f3a...") so a user can report it and support can find the exact request.
4. Redaction: `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.token`, `*.secret`, `*.phone`, card fields. Emails stay (support needs them) but never in URLs.
5. Levels: `error` for anything that paged or lost data, `warn` for degraded (rate limited, circuit open, retry), `info` for requests and jobs, `debug` off in production.
6. Never log request bodies by default. Log a body only on a specific error path, redacted.

## Errors

1. Sentry in all four apps: web (`@sentry/nextjs`), API and worker (`@sentry/node`), iOS (`@sentry/react-native`). Releases set from the git SHA, source maps uploaded in CI for web and iOS.
2. Tags: `tenant_id`, `route`, `job`. User context: user id only, no email.
3. Alert rules: a new issue in production, error rate above 2 percent over 5 minutes, any 5xx spike above 10 per minute, p95 latency above 1 second over 5 minutes.
4. Every unhandled error becomes a 500 with the request id and a generic message. The stack goes to Sentry, never to the client.

## Traces

1. OpenTelemetry SDK in the API and the worker from day one, with automatic instrumentation for HTTP, Postgres, Redis and the queue library.
2. `OTEL_EXPORTER_OTLP_ENDPOINT` and its headers point at the current backend: Sentry or Grafana Cloud at tier 1, Datadog or a self hosted collector later. Changing tools is changing those two variables.
3. The trace id joins every log line as `traceId`, so a slow request can be followed from the edge through the API into the job it enqueued.
4. Sampling: 10 percent of requests in production, 100 percent of errors; tail sampling at a collector once one exists.

## Metrics

1. Metrics flow through the same OpenTelemetry exporter, or the API exposes `/metrics` (`prom-client`) for a scraper. Either way the set is fixed:

```
http_requests_total{route,method,status}
http_request_duration_seconds{route}            p50, p95, p99
http_429_total{tier}
cache_ops_total{result}                          hit, stale, miss
queue_depth{queue}   queue_oldest_job_age_seconds{queue}   jobs_failed_total{queue}
job_duration_seconds{queue,name}
db_pool_in_use   db_pool_waiting
circuit_state{provider}
business: signups_total, active_tenants, ingest_rows_total
```

2. The database dashboard (Supabase Grafana at tier 1, RDS Performance Insights later) for the database side: connections versus max, CPU, cache hit ratio, disk IO, dead tuples, replication lag.
3. Redis: memory used versus `maxmemory`, evictions (must be zero on `redis-queue`), connected clients.
4. Cloudflare analytics: requests, cached ratio, WAF events, rate limited requests, origin response time.

## Health endpoints

```ts
app.get('/healthz', async () => ({ ok: true }));                 // liveness: process is up
app.get('/readyz', async (req, reply) => {                       // readiness: can serve traffic
  const [db, redis] = await Promise.allSettled([
    sql`select 1`, redisCache.ping(),
  ]);
  const ok = db.status === 'fulfilled' && redis.status === 'fulfilled' && !shuttingDown;
  return reply.code(ok ? 200 : 503).send({ ok, db: db.status, redis: redis.status });
});
```

The platform health check hits `/readyz`. Better Stack hits `/readyz` every 30 seconds from two regions and runs a synthetic login plus feed load every 5 minutes with a staging test account. The worker exposes `/healthz` on an internal port.

## Alert thresholds (defaults)

```
error rate                   > 2 percent over 5 min
p95 latency                  > 1 s over 5 min
5xx                          > 10 per min
429 share                    > 5 percent of requests
DB connections               > 70 percent of max clients
DB CPU                       > 80 percent over 10 min
DB disk                      > 80 percent
buffer cache hit ratio       < 99 percent over 1 hour
queue oldest job age         > 5 min
failed jobs                  > 50 in any queue
Redis memory                 > 80 percent
Redis evictions on queue     > 0
circuit breaker open         any provider, immediately
webhook signature failures   > 10 per min
Turnstile failures           > 20 percent
certificate expiry           < 14 days
uptime check                 2 consecutive failures
```

Every alert names the runbook section that handles it. An alert without a next step gets deleted.

## Reliability patterns (what the code does)

1. Timeouts on every outbound call, every database statement (per role), every job.
2. Retries with exponential backoff and full jitter, only for idempotent operations, only on retryable errors (network, 429, 5xx). In the request path prefer no retry and an enqueue.
3. Circuit breakers per provider so a dead provider degrades one feature instead of filling every worker slot with hanging calls.
4. Bulkheads: separate pools and queues for separate workloads (ingest versus notify versus exports), so one backlog does not starve the rest.
5. Idempotency keys on every creating POST, stored 24 hours, so client retries never duplicate.
6. Feature flags in a `flags(key, enabled, tenant_allowlist jsonb)` table cached 30 seconds. Every new risky feature ships behind a flag so turning it off is a row update, not a deploy. Kill switches for outbound messaging, LLM calls and ingestion exist from day one.
7. Graceful shutdown on `SIGTERM` in both processes; `/readyz` flips to 503 first.
8. Zero downtime deploys: the platform (Railway, ECS, Kubernetes) only routes to the new instance after `/readyz` passes; migrations follow expand and contract (`database.md` section 4) so old and new code run side by side for a few minutes.
9. Read your writes: the API returns the created or updated object, the client renders it without refetching, and list caches are invalidated by the version bump before the response is sent.
10. Clock and time: UTC everywhere in storage and logs; the client formats.
11. Degraded mode is designed, not discovered: Redis down means limits go in memory and caches miss (slower, still correct); a provider down means jobs delay and the UI says "delayed"; the database down means `/readyz` fails and Cloudflare serves a static maintenance page from a Worker route for `api.domain` with a 503 and `Retry-After`.

## Testing

1. Unit tests for pure logic (parsers, pricing, permissions matrix).
2. Integration tests against the docker compose Postgres and Redis: every route's happy path, auth rejection, validation rejection, and the rate limit response.
3. `rls.test.ts`: two tenants, every table, isolation proven. Runs first; a failure blocks the rest.
4. Webhook contract tests with recorded real payloads per provider, including signature verification and duplicate delivery.
5. Load test with k6 before launch and before any big release: the feed endpoint and the main write endpoint at three times the expected peak for 10 minutes. Pass means p95 under 500ms, zero 5xx, DB connections under 70 percent, no queue backlog growth. Script lives in `test/load/`.
6. Failure drill in staging once before launch: stop `redis-cache` and confirm the API still serves and auth fails closed; stop a provider (point its URL at a black hole) and confirm jobs delay and the circuit opens; restore both and confirm recovery without manual steps.
7. CI runs 1 to 4 on every push. 5 and 6 are manual, logged in RUNBOOK.md with dates.

## Runbook (docs/RUNBOOK.md)

Numbered steps with exact commands for:

1. Rollback a deploy (the owner does it; the runbook records the exact platform steps so it takes a minute under stress).
2. Rotate each secret, one section each.
3. Restore from PITR, with the measured duration from the last test.
4. Block an IP, ASN or country at Cloudflare; suspend a tenant.
5. Pause and resume the worker; drain and replay a queue.
6. Purge caches (tenant version bump, or `FLUSHDB` on `redis-cache` only).
7. Scale up compute (database, backend) and what to check after.
8. Replay inbound events after a parser fix.
9. Contact list: every provider's support channel, the payment processor, Apple App Review.
10. Postmortem template: what happened, timeline, impact, root cause, what changes, owner, date. One per incident, kept in `docs/postmortems/`.
