# Database

Postgres at every tier. The provider changes by tier (Supabase or Neon, then RDS or Aurora, then partitions and shards) and each move is logical replication, documented in `migrations.md`. The rules below are the same everywhere, which is what makes the moves cheap.

## 0. How to design the schema

Do this in order, before the first migration. The rules in section 1 are what the result must satisfy; this is how to get there.

1. Nouns first, then ownership. List every thing the product talks about (tenant, user, membership, listing, message, file, subscription, event). Draw the ownership tree: the tenant owns everything, users join tenants through memberships. Anything that does not hang off a tenant is a global reference table (plans, countries). That tree is the ERD, and it goes into PLAN.md under Data model before any SQL exists.
2. Classify every table into one of four kinds, because each kind gets different rules:
   * Reference: small, rarely changes (plans, categories). Lookup table, no `tenant_id`, editable by admins only.
   * Core entities: mutable, moderate size (listings, users, files). Full constraints, RLS, soft delete only where undo matters to the user.
   * Events: append only and big (`inbound_events`, messages, `audit_log`). Partitioned by month from day one, retention by dropping partitions, never updated.
   * Derived: counters, materialized views, search columns. Rebuildable from the other three kinds and never the only copy of a fact; the rebuild job is written the same day as the column.
3. Normalize the system of record. Every fact lives in exactly one place (third normal form). Denormalize only on purpose, for a read path that was measured, and only as a derived column with a rebuild. A `listing_count` on `tenants` is fine; a copied `tenant_name` on every listing is a bug waiting to happen.
4. Put the invariants in the schema, not in the app. `not null` by default, `check` constraints for enumerations and ranges, `unique` on natural keys, a foreign key on every relation with `on delete` chosen deliberately, composite `(tenant_id, parent_id)` foreign keys so rows cannot cross tenants. Bad data that cannot be inserted is a class of bugs that cannot exist.
5. Write the top ten queries before the indexes. Every list screen, every search, every scan a job does. Each tenant scoped list gets `(tenant_id, filter columns, sort column desc)`; every foreign key gets an index; hot subsets get partial indexes. Then stop; write heavy tables pay for every extra index on every insert.
6. The same four columns on everything: `id` (UUID v7, generated in the app), `tenant_id` where owned, `created_at`, `updated_at` kept by the trigger in the appendix.
7. One naming convention, never broken: `snake_case`, plural table names, `{singular}_id` for foreign keys, `{table}_{columns}_idx` for indexes, `timestamptz` always, money as integer cents.
8. Schema as code, changed by expand and contract (section 4). Additive changes in one release, drops in a later one, staging before production.
9. The RLS test is the schema review. Two tenants, every table, proven isolated. A new table that does not pass it does not merge.

Output of this step: the ERD and the table list with kinds in PLAN.md, the top ten queries as comments above their indexes in the first migration, and the RLS test extended to every new table.

## 1. Schema rules

1. Primary keys are UUID v7 generated in the app (`uuidv7` package). Time ordered, so B tree inserts append instead of splatter, and ids never leak counts the way sequences do.
2. Every table: `id`, `created_at timestamptz not null default now()`, `updated_at timestamptz not null default now()` kept current by a trigger. Tenant owned tables add `tenant_id uuid not null references tenants(id)`.
3. Soft delete (`deleted_at`) only where undo matters to the user (listings, documents). Everything else hard deletes. Soft deleted rows are excluded by a partial index and by the default query helper, never by remembering to add a `where`.
4. Types: `timestamptz` always, never `timestamp`. Money as integer cents or `numeric(12,2)`, never float. `text` with `check (length(x) <= n)` over `varchar`. `jsonb` only for genuinely variable data, with a GIN index if queried. Enumerations as `check (status in (...))` or a lookup table; Postgres enums are painful to alter.
5. Naming: `snake_case`, plural table names, foreign key columns `{singular}_id`, index names `{table}_{columns}_idx`, constraints `{table}_{what}_key` or `_fk` or `_check`.
6. `not null` by default. Nullable is a decision written in the schema comment.
7. Foreign keys everywhere, with `on delete` chosen on purpose: `cascade` for owned children, `restrict` for shared references, `set null` rarely. Composite `(tenant_id, parent_id)` foreign keys on child tables so rows cannot cross tenants.
8. Unique constraints on natural keys: `(tenant_id, external_id)`, `(tenant_id, slug)`, `(user_id, tenant_id)`.
9. Dedup column for ingested content: `content_hash bytea` (sha256 of the normalized body), unique per `(tenant_id, source, content_hash)` with `on conflict do nothing` on insert.
10. Indexes: every foreign key; every tenant scoped list query gets `(tenant_id, <filter cols>, <sort col> desc)`; partial indexes for hot subsets; `pg_trgm` GIN for substring search or a `tsvector` column for full text. Write heavy tables get the minimum set, since each index is paid on every insert.
11. Big append only tables (events, messages, raw listings, logs) that will pass 1 million rows a month are partitioned by month on `created_at` from day one. Retention is `drop table` on the oldest partition, which is instant, instead of a `delete` that bloats for days.
12. Schema comments (`comment on column`) for anything not obvious. Claude reads them later.

## 2. Roles and access

See `security.md` section 1 for the SQL. Summary: the owner role (`postgres` on Supabase, a dedicated `app_migrator` on RDS) owns everything and runs migrations over the direct connection; `app_api` and `app_worker` are login roles with `nobypassrls`, DML only, no DDL, per role timeouts; RLS enabled and forced on every table; on Supabase, grants on the app schema revoked from `anon` and `authenticated` because clients never use PostgREST.

## 3. Connections

1. A transaction mode pooler for the API and the worker: Supavisor on port 6543 at Supabase, PgBouncer or RDS Proxy on AWS. The direct connection is for migrations, `pg_dump` and a human with `psql`.
2. postgres.js client: `max: 10` per API instance, `max: 5` per worker, `prepare: false`, `idle_timeout: 20`, `connect_timeout: 10`, `ssl: 'require'`.
3. Keep the sum of all pools under half of the pooler's max clients for the compute size. Alert at 70 percent in the database dashboard.
4. Transactions are short. Never hold one across a network call (R2, Stripe, an LLM). Read, release, call, then write in a new transaction.
5. Timeouts per role (`statement_timeout`, `idle_in_transaction_session_timeout`, `lock_timeout`) are set with `alter role`, not per query, so nobody forgets.
6. When something must be claimed by exactly one worker from Postgres (rare, since BullMQ handles jobs): `select ... for update skip locked limit n`.

The classic outage: serverless functions opening direct connections per invocation and a worker with an oversized pool, on a compute size with a low client cap. Pooled URL, small pools and the alert above prevent it.

## 4. Migrations

1. Drizzle Kit generates SQL into `src/db/migrations/`. Read the generated SQL before applying it. Hand written SQL (roles, RLS, functions, triggers, partitions) goes in `src/db/sql/` and is referenced from a migration.
2. Apply with `drizzle-kit migrate` over the direct connection as the owner role. Staging first, run the integration tests against staging, then production. Print the full SQL in the response as well so the owner can review it before it runs.
3. Never change a production schema from the dashboard. If an emergency forces it, write the same change as a migration the same day.
4. Zero downtime pattern (expand and contract):
   * Add a nullable column or a new table. Deploy code that writes both old and new.
   * Backfill in batches of 1000 rows per transaction from the worker, sleeping 50ms between batches.
   * Add the `not null` or the constraint with `not valid` first, then `validate constraint` separately.
   * Deploy code that reads the new column only.
   * Drop the old column in a later migration, after a release has been live for a day.
5. Indexes on tables over 100k rows are created `concurrently`, outside a transaction (Drizzle migration marked as non transactional).
6. Never rename a column or table in one step. Add, migrate, drop.
7. Every migration states in a comment whether it is reversible and how.
8. Migrations never contain data from a user. Seed data lives in `db:seed`, not in migrations.

## 5. Backups and recovery

1. Daily automated backups are the floor on every provider. PITR (an add on at Supabase, built in on RDS and Aurora) goes on the day the first paying customer arrives. PITR is what turns "a bad migration deleted a column" into a 15 minute recovery instead of a lost day.
2. Weekly logical dump by the worker: `pg_dump --format=custom` over the direct connection, encrypted with `age` or GPG, uploaded to R2 under `backups/` with a 90 day lifecycle. This copy is outside the provider; it is the answer to the provider having a bad day.
3. Restore test once per quarter: restore the latest PITR point to a scratch project, run the smoke tests, write the elapsed time in RUNBOOK.md, delete the scratch project.
4. R2 object versioning on for the uploads bucket. Exports and backups get lifecycle expiry.
5. Before every migration that drops or rewrites data: a manual snapshot and the restore command ready in the runbook.

## 6. Performance hygiene

1. `pg_stat_statements` on. Weekly: top 10 by total time and by mean time. Anything over 100ms at p95 gets `explain (analyze, buffers)` and an index, a rewrite or a cache.
2. Dashboard panels that matter (Supabase Grafana at tier 1, RDS Performance Insights later): active connections versus max, CPU, buffer cache hit ratio (above 99 percent), disk IO, replication lag if a replica exists, dead tuples on the five biggest tables.
3. Autovacuum on big hot tables: `alter table listings set (autovacuum_vacuum_scale_factor = 0.01, autovacuum_analyze_scale_factor = 0.005)` so vacuum runs on a few hundred thousand dead rows, not on 20 percent of the table.
4. Bulk writes: multi row inserts of 500 rows per statement, `copy` for imports over 100k rows, `on conflict do nothing` for dedup, all from the worker.
5. `work_mem` stays default; a query that needs more is a query that needs an index.
6. Long reports run on a read replica when one exists, otherwise in the worker at night with `statement_timeout = '10min'` on `app_worker` for that job only (`set local`).
7. Table bloat check monthly (`pgstattuple` or the provider's report); `pg_repack` or a rebuild when bloat passes 50 percent on a hot table.

## 7. Scale triggers and how to act on each

* Read replica (an add on at Supabase, one Terraform change on RDS): when p95 read latency climbs and primary CPU is above 60 percent from reads, after caching is in place. Route only reports and heavy lists to it; never auth or writes.
* Compute upgrade: connections or CPU above 70 percent for an hour with queries already indexed. Vertical first; it is a dashboard change.
* Partitioning an existing big table: create the partitioned table, `insert ... select` in batches from the worker, swap names in one transaction, keep the old table a week, drop.
* RDS plus PgBouncer: when failover must be automatic, or compute is past 16 cores with queries optimized. Cut over with logical replication; the step by step is in `migrations.md`. Identity can stay where it is; the database is what moves.
* ClickHouse for analytics: when dashboards aggregate more than 100 million rows or reporting is visibly hurting the primary. Feed it by CDC (logical replication to ClickHouse or ClickPipes), never by writing from the app twice.
* Sharding: past 1 TB on a single primary. Not before. At that size hire a DBA for the week.

## 8. SQL appendix

```sql
-- updated_at trigger
create or replace function app.touch_updated_at() returns trigger
language plpgsql as $$
begin new.updated_at = now(); return new; end $$;
-- per table:
create trigger listings_touch before update on listings
  for each row execute function app.touch_updated_at();

-- tenants and memberships
create table tenants (
  id uuid primary key,
  name text not null check (length(name) <= 120),
  slug text not null unique,
  status text not null default 'active' check (status in ('active','suspended','deleted')),
  plan text not null default 'trial',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
create table memberships (
  id uuid primary key,
  tenant_id uuid not null references tenants(id) on delete cascade,
  user_id uuid not null,              -- auth.users.id
  role text not null check (role in ('owner','admin','member','viewer')),
  created_at timestamptz not null default now(),
  unique (user_id, tenant_id)
);
create index memberships_tenant_idx on memberships (tenant_id);

-- a tenant owned table with the full pattern
create table listings (
  id uuid primary key,
  tenant_id uuid not null references tenants(id) on delete cascade,
  author_id uuid not null,
  title text not null check (length(title) <= 200),
  price_cents integer not null check (price_cents >= 0),
  status text not null default 'active' check (status in ('active','sold','archived')),
  content_hash bytea,
  deleted_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (tenant_id, content_hash)
);
create index listings_tenant_active_idx on listings (tenant_id, created_at desc)
  where deleted_at is null;
create index listings_title_trgm_idx on listings using gin (title gin_trgm_ops);
alter table listings enable row level security;
alter table listings force row level security;
-- policies: see security.md section 1

-- an append only partitioned event table
create table inbound_events (
  id uuid not null,
  provider text not null,
  external_id text not null,
  tenant_id uuid,
  payload jsonb not null,
  received_at timestamptz not null default now(),
  primary key (id, received_at)
) partition by range (received_at);
create table inbound_events_2026_11 partition of inbound_events
  for values from ('2026-11-01') to ('2026-12-01');
create unique index inbound_events_provider_ext_idx
  on inbound_events (provider, external_id, received_at);
-- the worker creates next month's partition on the 25th and drops partitions older than 90 days

-- keyset pagination
select id, title, price_cents, created_at
from listings
where tenant_id = app.current_tenant()
  and deleted_at is null
  and (created_at, id) < ($1::timestamptz, $2::uuid)
order by created_at desc, id desc
limit 25;

-- what to look at when it is slow
select calls, round(total_exec_time::numeric, 1) as total_ms,
       round(mean_exec_time::numeric, 2) as mean_ms, left(query, 120) as query
from pg_stat_statements
order by total_exec_time desc
limit 10;
```
