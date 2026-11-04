# Jobs and queues

Rule: anything slower than 200ms, anything that calls a third party, anything that can fail and needs a retry, leaves the request path and becomes a job. The request writes a row and enqueues; the worker does the work. This is what keeps the API fast under load and what makes provider outages a delay instead of an outage.

## Setup

* BullMQ on `redis-queue`: `maxmemory-policy noeviction`, `appendonly yes`, persistence on. Jobs are data; this instance is never shared with the cache.
* One worker process (`src/worker.ts`) runs every queue with concurrency set per queue. Scale by adding a second worker replica, not by raising concurrency past what the database pool allows.
* Queues and default concurrency:

```
ingest     10    parse raw events, normalize, batch insert
media       2    image and video processing (CPU bound)
notify     20    push, email, in app notifications
billing     5    Stripe sync, invoice state, plan changes
exports     1    user and tenant data exports
outbound    per provider, see rate-limiting.md layer 4
cron        1    scheduled jobs
```

## Job design

1. Idempotent, always. Deterministic job ids (`listing:{id}:created`, `export:{tenantId}:{date}`) so a duplicate enqueue is a no op, and handlers that can run twice without double effects (check state before acting, `on conflict do nothing`, idempotency keys on outbound calls).
2. Small payloads: ids and a version, never blobs. The handler fetches fresh state, so a job delayed by an hour acts on current data.
3. Tenant id in every payload, set as the transaction tenant inside the handler, so RLS applies in the worker exactly as in the API.
4. Options per job type:

```ts
const defaultJobOptions = {
  attempts: 8,
  backoff: { type: 'custom' },           // exponential with full jitter, see below
  removeOnComplete: { age: 86400, count: 1000 },
  removeOnFail: { age: 7 * 86400 },
};
// worker settings: backoffStrategy(attempt) => Math.min(300000, 1000 * 2 ** attempt * Math.random())
```

5. Timeouts inside handlers: every external call has one; the job as a whole has a lock duration (`lockDuration: 60000`) and calls `job.updateProgress` on long loops so the lock renews.
6. Failures that are the caller's fault (bad input, missing row) throw `UnrecoverableError` so BullMQ does not retry them.
7. Handlers log `jobId`, `tenantId`, `attempt`, duration, and wrap in a Sentry transaction.

## Dead letters and visibility

1. Jobs that exhaust attempts stay in the failed set with the error and the stack. Nothing is silently dropped.
2. A `cron` job every 5 minutes counts failed jobs per queue and the age of the oldest waiting job. Alert when failed exceeds 50 in any queue or oldest waiting is older than 5 minutes.
3. Bull Board mounted at `/admin/queues` on the API, behind Cloudflare Access and the platform admin check. Retry, remove and inspect from there.
4. Ingest failures keep the `inbound_events.id` in the job data so the raw event can be replayed after a parser fix: a `replay` job that re enqueues by id range.

## Priorities and fairness

1. `priority` on enqueue: paid tenants 1, trial 5, internal backfills 10. Lower number runs first.
2. Per tenant in flight cap: before processing, `INCR inflight:{queue}:{tenant}` with a 10 minute TTL; over the cap (default 5), `moveToDelayed` for 5 seconds and return. `DECR` in `finally`. One tenant importing a million rows cannot starve everyone else.
3. Per queue rate limiter for outbound queues to match provider limits.

## Cron

1. BullMQ job schedulers (`queue.upsertJobScheduler`) in `src/jobs/cron.ts`, registered by exactly one worker process (the first replica; guard with a Redis lock `SET cron:leader NX EX 120` renewed every 60 seconds).
2. All schedules in UTC. Never Vercel cron or `setInterval` in the API process: both run in places that restart and scale without coordination.
3. Standard set:

```
aggregates.refresh      every 5 min     materialized views, counters
cache.warm              every 10 min    top 50 tenants' first feed page
partitions.rotate       daily 02:00     create next month's partitions, drop expired
retention.purge         daily 03:00     hard delete soft deleted rows past 30 days, orphan files in R2
billing.reconcile       daily 04:00     compare subscriptions table with Stripe, fix drift, alert
backups.logical         weekly Sun 05:00 pg_dump to R2, encrypted
health.report           daily 08:00     one email or Slack message: signups, errors, queue stats, DB size
```

## Backpressure

1. Webhooks always get accepted and stored; the raw event is the buffer. If `ingest` depth passes 10,000, the worker raises ingest concurrency within the pool budget and non critical producers (cache warm, aggregates) pause.
2. API endpoints that enqueue user work (exports, imports) check queue depth and return 503 with `Retry-After` when the queue is past its cap, instead of accepting work that will not run for an hour.
3. Alerts on depth and on oldest job age, not on throughput. Age is what the user feels.

## Graceful shutdown

1. On `SIGTERM` (every platform sends it on deploy and scale down): stop taking new jobs (`worker.close()` waits for active jobs up to 30 seconds), close Redis and Postgres, exit 0.
2. The API does the same: `app.close()` drains in flight requests, `/readyz` returns 503 first so the load balancer stops routing.
3. Jobs longer than 30 seconds are written to checkpoint progress in the database so a restart resumes instead of restarting from zero.

## Code skeleton

```ts
// jobs/queues.ts
import { Queue } from 'bullmq';
export const queues = {
  ingest:  new Queue('ingest',  { connection: redisQueue, defaultJobOptions }),
  media:   new Queue('media',   { connection: redisQueue, defaultJobOptions }),
  notify:  new Queue('notify',  { connection: redisQueue, defaultJobOptions }),
  billing: new Queue('billing', { connection: redisQueue, defaultJobOptions }),
  exports: new Queue('exports', { connection: redisQueue, defaultJobOptions }),
};
export const enqueue = {
  listingCreated: (p: { tenantId: string; listingId: string }) =>
    queues.notify.add('listing.created', p, { jobId: `listing:${p.listingId}:created` }),
};

// worker.ts
import { Worker } from 'bullmq';
const handlers = { 'listing.created': onListingCreated, 'ingest.parse': onIngestParse };
new Worker('notify', async (job) => {
  const handler = handlers[job.name as keyof typeof handlers];
  if (!handler) throw new UnrecoverableError(`no handler for ${job.name}`);
  return withTenant(job.data.tenantId, (tx) => handler(job, tx));
}, {
  connection: redisQueue,
  concurrency: 20,
  settings: { backoffStrategy: (attempt) => Math.min(300000, 1000 * 2 ** attempt * Math.random()) },
});
process.on('SIGTERM', async () => { await Promise.all(workers.map((w) => w.close())); process.exit(0); });
```

## When BullMQ is not enough

* Workflows that must resume after a crash mid flow, wait days for a human, or chain more than three steps that each must survive failure: Temporal (Temporal Cloud or self hosted). This is what Netflix, Duolingo and most fintechs run; `migrations.md` has the dual run path from BullMQ, and picker rule 7 starts on Temporal when the product is built on such flows.
* Events that three or more services need, or change data capture into a warehouse: Kafka (MSK or Confluent), published through an outbox table after commit, never from inside the request.
* Jobs that run for hours on big machines (video renders, model pipelines): a dedicated worker service with its own queue and concurrency 1, or Trigger.dev when there is no server to run it on.
* Everything else is BullMQ on one worker, which carries a product through tier 1 and most of tier 2.
