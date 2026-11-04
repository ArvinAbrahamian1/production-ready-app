# Rate limiting

Four layers. Each one stops a different failure: the edge stops volume, the API stops abuse per identity, the database stops any single query from taking the service down, and the outbound layer stops the app from getting banned by the providers it depends on.

Every number below is a default. Write the project's actual numbers in PLAN.md under API surface, one tier per route group, and keep the plan multipliers in the `plans` table so limits change without a deploy.

## Layer 1: the edge (Cloudflare at every tier)

Cloudflare caps how many rate limiting rules a zone gets (one on Free, two on Pro, five on Business), so the edge rules are the coarse ones and the API does the fine grained work. If the edge is ever CloudFront, the same rules become AWS WAF rate based rules and nothing in the API changes.

1. DDoS protection: automatic, nothing to configure.
2. WAF managed rules: Cloudflare Managed Ruleset on; OWASP Core Ruleset on with Pro at paranoia level 1.
3. Rate limiting rule A, auth: path matches `/v1/auth/*` or `/auth/*`, 10 requests per minute per IP, action block for 10 minutes.
4. Rate limiting rule B, everything on the host: 600 requests per minute per IP, action managed challenge for one minute (block instead of challenge on paths under `/webhooks/`, since providers cannot answer a challenge; put webhooks in a separate rule when the plan allows, otherwise exclude the path from rule B and let the API limit them).
5. Bot Fight Mode on. On Pro, Super Bot Fight Mode: block definitely automated, allow verified bots, log likely automated for the first two weeks before blocking.
6. Turnstile on signup, login, password reset and any public form. The API calls `siteverify` with the token and the client IP before processing. No token, no request.
7. WAF custom rules: block `/admin*` unless the Cloudflare Access JWT header is present; block requests with no `User-Agent`; geo block countries the product does not serve if abuse shows up from them.
8. Watch Security Events weekly for the first month and tune.

## Layer 2: the API

Keys, in order of preference: user id from the JWT, tenant id, API key id, client IP from `cf-connecting-ip` (trusted only because the origin accepts Cloudflare alone).

Algorithm: sliding window counters in `redis-cache` via `rate-limiter-flexible` (`RateLimiterRedis`) with an `insuranceLimiter` in memory so a Redis outage degrades to per instance limits instead of taking the API down. Auth routes are the exception: they fail closed (503) when Redis is unreachable, because brute force protection matters more than login availability for a few minutes.

Default tiers:

```
tier            key            limit                       notes
auth.login      ip             5 / minute                  plus 20 / hour per email
auth.signup     ip             3 / hour                    Turnstile required
auth.reset      email          3 / hour
read            user           300 / minute                GET on any resource
read.tenant     tenant         1000 / minute               sum over the tenant's users
write           user           60 / minute                 POST, PATCH, DELETE
search          user           60 / minute, burst 10 / s   feed and search endpoints
export          user           5 / hour                    plus 1 concurrent per tenant
upload.presign  user           30 / minute
ai              user           20 / minute                 plus 500 / day per tenant
public          ip             60 / minute                 unauthenticated GET
webhook         provider       1000 / minute               keyed by provider, not IP
admin           user           120 / minute
```

Plan multipliers stored per tenant and cached 5 minutes: trial 1x, paid 3x, enterprise set by hand. A suspended tenant gets 0.

Responses:

* 429 with JSON `{ "error": "rate_limited", "retry_after": <seconds> }`.
* Headers on every response: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`, and `Retry-After` on 429.
* Log at warn with the tier and the hashed key. Count 429s as a metric; above 5 percent of requests means either an attack or limits that are too tight.

Concurrency limits for expensive work (semaphores, not counters): `INCR` with a TTL guard and `DECR` in `finally`. Max 2 concurrent exports per tenant, max 5 concurrent search queries per tenant, max 1 concurrent import per tenant. Over the cap returns 503 with `Retry-After: 5`.

Cost based budget for mixed workloads: give routes weights (list 1, detail 1, search 5, aggregate 10, export 50, LLM call 20) and enforce a per tenant budget per minute (default 2000 points). This is what stops one tenant's dashboard from starving everyone else.

```ts
// http/middleware/rateLimit.ts
import { RateLimiterRedis, RateLimiterMemory } from 'rate-limiter-flexible';

const tiers = {
  read:   { points: 300, duration: 60 },
  write:  { points: 60,  duration: 60 },
  search: { points: 60,  duration: 60 },
  export: { points: 5,   duration: 3600 },
} as const;

const limiters = Object.fromEntries(
  Object.entries(tiers).map(([name, t]) => [name, new RateLimiterRedis({
    storeClient: redisCache,
    keyPrefix: `rl:${name}`,
    points: t.points,
    duration: t.duration,
    insuranceLimiter: new RateLimiterMemory({ points: t.points, duration: t.duration }),
  })]),
);

export function rateLimit(tier: keyof typeof tiers) {
  return async (req: FastifyRequest, reply: FastifyReply) => {
    const key = req.user?.id ?? req.headers['cf-connecting-ip'] ?? req.ip;
    const multiplier = req.tenant?.planMultiplier ?? 1;
    try {
      const res = await limiters[tier].consume(key, 1);
      reply.header('RateLimit-Limit', tiers[tier].points * multiplier);
      reply.header('RateLimit-Remaining', Math.max(0, res.remainingPoints));
      reply.header('RateLimit-Reset', Math.ceil(res.msBeforeNext / 1000));
    } catch (res: any) {
      const retry = Math.ceil((res.msBeforeNext ?? 60000) / 1000);
      req.log.warn({ tier, keyHash: hash(key) }, 'rate limited');
      reply.header('Retry-After', retry);
      return reply.code(429).send({ error: 'rate_limited', retry_after: retry });
    }
  };
}

// usage: app.get('/v1/listings', { preHandler: [auth, tenant, rateLimit('read')] }, handler)
```

Idempotency on writes belongs next to rate limiting: `Idempotency-Key` header required on every POST that creates something, stored in `redis-cache` as `idem:{tenant}:{key}` for 24 hours with the final status and body. Same key twice returns the stored response. This makes client retries safe and removes the duplicate rows that retries otherwise create.

## Layer 3: the database

1. Per role timeouts, set once with `ALTER ROLE` (see `security.md`): `app_api` 5 second statements, 10 second idle in transaction, 2 second lock timeout; `app_worker` 60 second statements.
2. Pool sizes: API 10 connections per instance, worker 5, total kept under 50 percent of the pooler's max clients for the compute size. Alert at 70 percent of max clients in the database dashboard (Supabase Grafana at tier 1, Performance Insights on RDS).
3. Nothing heavy runs in the request path. Aggregates, exports, imports and reports are jobs with lower priority queues.
4. Per tenant fairness in the worker: BullMQ limiter per queue plus a Redis counter of in flight jobs per tenant (cap 5); extra jobs are delayed, not dropped.
5. `pg_stat_statements` reviewed weekly; any query over 100ms at p95 gets an index or a cache before anything else is built.

## Layer 4: outbound

1. One BullMQ queue per provider with `limiter: { max, duration }` matching the provider's published limit (email provider, messaging provider per connected number, LLM provider tokens per minute).
2. Exponential backoff with full jitter: base 1 second, cap 5 minutes, 8 attempts, then the failed set.
3. Circuit breaker per provider (`opossum`): open at 50 percent failures over 30 seconds, half open after 60 seconds. While open, jobs are delayed, not failed.
4. Honor `Retry-After` and provider specific headers (`x-ratelimit-remaining` style) by delaying the queue when remaining drops under 10 percent.
5. Per tenant outbound caps (messages per day, emails per hour) stored with the plan, enforced before enqueueing, surfaced in the UI so the user knows why something is delayed.

## Abuse controls beyond counting

1. Signup velocity: more than 3 accounts per IP per day or per device id (PostHog distinct id or a device fingerprint on mobile) flags the tenant for review and disables outbound features until reviewed.
2. Disposable email domains blocked at signup (maintained list in the repo, refreshed monthly).
3. Trial features that cost money (LLM calls, outbound messages, exports) require a verified email and a completed Turnstile, and have the lowest daily caps.
4. Kill switch per tenant: `tenants.status in ('active', 'suspended', 'deleted')`, checked in the auth middleware from the cached membership, so one update stops a bad actor everywhere within 60 seconds.
5. Honeypot field on public forms; a filled honeypot is logged and dropped silently.
6. Alerts: 429 rate above 5 percent, Turnstile failures above 20 percent, signup spike above 5x the 7 day average, webhook signature failures above 10 per minute.
