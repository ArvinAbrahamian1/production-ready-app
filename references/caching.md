# Caching

Principles that hold at every layer:

1. Cache aside: the app reads the cache, misses go to the source, the app writes the cache. No write through magic.
2. Every key is scoped to the tenant and includes a schema version. A cache that can serve one tenant's data to another is a security bug, not a performance bug.
3. Short TTLs with jitter. Staleness measured in seconds is fine for feeds; staleness in minutes is not fine for billing or permissions.
4. Nothing gets cached until its invalidation is written in the caching map in PLAN.md: what, key, TTL, which writes clear it.
5. Measure hit ratio per cache. A cache under 50 percent hits is costing more than it saves.
6. Never cache authenticated responses at the CDN. Never cache secrets, billing state or permission checks longer than 60 seconds.

## Layer 1: browser and the frontend CDN (web)

* Static assets: content hashed, `Cache-Control: public, max-age=31536000, immutable`. Next.js does this for `/_next/static`.
* Public pages (marketing, public profile pages, public listing pages): ISR with `revalidate` between 60 and 300 seconds, plus on demand revalidation: the worker calls a `POST /api/revalidate` route on the web app with a shared secret and the tag when underlying content changes. Tags per resource (`listing:{id}`, `tenant:{id}:public`).
* Authenticated pages: `Cache-Control: private, no-store`. Server components fetch from the API per request; the API's own cache does the work.
* Images: `next/image` pointed at the public asset host. Objects are written with `Cache-Control: public, max-age=31536000, immutable` under hashed keys, so a replaced image is a new key and old caches never go stale.
* Fonts self hosted through `next/font`.

## Layer 2: Cloudflare edge (API host)

Only for GETs that are the same for everyone: `/v1/public/*`, OpenGraph images, sitemaps, public embed data.

* Cache Rule: match `GET` on those paths, Edge TTL 60 seconds, cache key includes the full query string, bypass when `Authorization` or `Cookie` is present.
* The API sends `Cache-Control: public, s-maxage=60, stale-while-revalidate=300` on these routes and `private, no-store` on everything else.
* Purge by URL from the worker when the underlying row changes, using a Cloudflare API token scoped to cache purge on the zone.
* Tiered Cache on (one setting) so misses from many edge locations collapse to one origin fetch.

## Layer 3: HTTP validation between API and clients

* `@fastify/etag` on list and detail routes. The ETag is a hash of the response body. Clients send `If-None-Match` and get 304 with no body. This cuts bandwidth on mobile and makes polling cheap.
* `Last-Modified` from `max(updated_at)` on lists when computing the body hash is expensive.
* Mobile client sends `If-None-Match` automatically through the typed client.

## Layer 4: application cache (Redis, `redis-cache`)

Instance settings: `maxmemory` set to 80 percent of the plan, `maxmemory-policy allkeys-lru`, no persistence needed. This instance is disposable; the app must work with it empty.

Key schema:

```
v{schema}:t:{tenantId}:ver{tenantVersion}:{resource}:{idOrHash}    tenant scoped data
v{schema}:g:{resource}:{id}                                           global, non tenant data
t:{tenantId}:ver                                                      the tenant's version counter
lock:{key}                                                            single flight locks
idem:{tenantId}:{key}                                                 idempotency responses
rl:{tier}:{key}                                                       rate limit counters
dedupe:{provider}:{eventId}                                           webhook dedupe
```

`{schema}` increments when the shape of a cached object changes (deploy time constant). `{tenantVersion}` is an integer in Redis; bumping it with `INCR` invalidates every list and aggregate for that tenant at once and old keys expire on their own. No `KEYS`, no `SCAN` in the request path, ever.

TTLs (defaults, with plus or minus 10 percent jitter so keys do not expire together):

```
feed and list pages          30 to 60 s
detail objects               5 min
aggregates and counters      5 min
membership and roles         60 s
tenant plan and limits       5 min
feature flags                30 s
negative cache (404)         30 s
idempotency responses        24 h
```

Stampede protection, in this order of preference:

1. Serve stale while revalidating: store `{ data, freshUntil }` with a Redis TTL of twice the freshness window. On a request after `freshUntil`, return the stale data immediately and refresh in the background (one refresh, guarded by a lock).
2. Single flight lock for cold keys: `SET lock:{key} 1 NX PX 5000`. The winner computes and stores. Losers wait up to 200ms polling the key, then compute themselves if still empty (bounded wait beats a pile up).

Write path ordering: write to Postgres inside the transaction, commit, then invalidate (bump the tenant version or `DEL` the specific keys), then enqueue jobs. Never write the cache before the commit; a rollback would leave a lie in the cache.

Never cache: anything containing another user's private data under a shared key, billing state (read it from Postgres, it is one indexed row), permission decisions beyond the 60 second membership lookup, responses over 1 MB (paginate instead), error responses other than 404.

Size: JSON over 10 KB is stored gzipped (`zlib` in the helper). Measure before adding MessagePack; it rarely matters.

```ts
// lib/cache.ts
export async function cached<T>(opts: {
  key: string; ttlSec: number; staleSec?: number; fn: () => Promise<T>;
}): Promise<T> {
  const { key, ttlSec, staleSec = ttlSec, fn } = opts;
  const raw = await redisCache.get(key);
  if (raw) {
    const { data, freshUntil } = JSON.parse(raw) as { data: T; freshUntil: number };
    if (Date.now() < freshUntil) { metrics.cacheHit(key); return data; }
    metrics.cacheStale(key);
    void refresh(key, ttlSec, staleSec, fn);  // background, lock guarded
    return data;
  }
  metrics.cacheMiss(key);
  const got = await redisCache.set(`lock:${key}`, '1', 'PX', 5000, 'NX');
  if (!got) {
    for (let i = 0; i < 10; i++) {
      await sleep(20);
      const again = await redisCache.get(key);
      if (again) return (JSON.parse(again) as { data: T }).data;
    }
  }
  return refresh(key, ttlSec, staleSec, fn, true);
}

async function refresh<T>(key: string, ttlSec: number, staleSec: number, fn: () => Promise<T>, awaited = false) {
  if (!awaited) {
    const got = await redisCache.set(`lock:${key}`, '1', 'PX', 5000, 'NX');
    if (!got) return;
  }
  try {
    const data = await fn();
    const jitter = 1 + (Math.random() * 0.2 - 0.1);
    const freshUntil = Date.now() + ttlSec * 1000 * jitter;
    await redisCache.set(key, JSON.stringify({ data, freshUntil }), 'EX', Math.ceil((ttlSec + staleSec) * jitter));
    return data;
  } finally {
    await redisCache.del(`lock:${key}`);
  }
}

export const tenantKey = async (tenantId: string, resource: string, suffix: string) => {
  const ver = (await redisCache.get(`t:${tenantId}:ver`)) ?? '0';
  return `v${SCHEMA_VERSION}:t:${tenantId}:ver${ver}:${resource}:${suffix}`;
};
export const invalidateTenant = (tenantId: string) => redisCache.incr(`t:${tenantId}:ver`);
```

## Layer 5: in process cache (per instance)

`lru-cache` for things every request needs and that change rarely: JWKS keys (10 minutes), tenant plan (60 seconds), feature flags (30 seconds), the disposable email list. Short TTLs because instances do not share it and a change must propagate within a minute without a deploy.

## Layer 6: Postgres

The first cache is an index. Before adding Redis for a slow query, fix the query.

* Covering indexes for list queries: `(tenant_id, created_at desc) include (title, price, status)` so the list is served from the index alone.
* Partial indexes for hot subsets: `where deleted_at is null`, `where status = 'active'`.
* Keyset pagination on `(created_at, id)`. `OFFSET` past the tenth page is forbidden; it reads and discards every skipped row.
* Materialized views for aggregates (per tenant daily stats, leaderboards), refreshed by the worker with `refresh materialized view concurrently` (needs a unique index) every 5 minutes. Request path reads the view.
* Denormalized counters (`tenants.listing_count`) updated in the same transaction as the insert, or by the ingest job in batch. `count(*)` on a large table never runs in a request.
* Prepared statements off under any transaction mode pooler (Supavisor, PgBouncer, RDS Proxy) with `prepare: false` in postgres.js, or the pooler returns errors under load.
* Buffer cache hit ratio in the database dashboard should stay above 99 percent; below that the compute is too small for the working set, and no amount of Redis fixes a cold Postgres.

## Layer 7: client state (web and mobile)

* TanStack Query everywhere: `staleTime` 30 seconds for lists, 5 minutes for details, `gcTime` 24 hours. Refetch on window focus (web) and app foreground (mobile). Invalidate the relevant query keys on mutation success.
* Mobile persists the query cache (MMKV persister) so the app opens with data and refreshes behind it.
* Optimistic updates for the user's own writes, rolled back on error. The API returns the created object on 201 so no refetch is needed.
* Realtime: poll every 30 to 60 seconds or use push notifications. Database change subscriptions (Supabase Realtime, logical decoding listeners) hold database connections and are enabled only on the one or two tables a chat like screen needs, never on feeds. At tier 2 and up a WebSocket service with Redis pub/sub does this job without touching the database.

## The caching map (required in PLAN.md)

One row per cached thing, before it ships:

```
what                      key                                   ttl    invalidated by
listing feed page         t:{tenant}:ver{n}:listings:{hash}     30 s   any listing write (version bump)
listing detail            t:{tenant}:ver{n}:listing:{id}        5 min  listing update (version bump)
tenant daily stats        t:{tenant}:ver{n}:stats:daily         5 min  worker refresh job
membership                g:membership:{userId}                 60 s   role change, suspension (DEL)
tenant plan               g:plan:{tenantId}                     5 min  Stripe webhook (DEL)
```
