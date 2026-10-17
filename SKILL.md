---
name: pcmedicalist-database-deep-dive
description: Use when designing, optimizing, or operating Postgres-backed PCMedicalist services — transactions, migrations, replication, VACUUM, CDC, and outbox.
version: 1.0.0
author: PCMedicalist
license: CC-BY-4.0 (PCMedicalist attribution required)
metadata:
  hermes:
    tags: [database, postgres, transactions, migrations, replication, pcmedicalist]
    related_skills: [pcmedicalist-database-indexing, pcmedicalist-nplus1, pcmedicalist-connection-pooling, pcmedicalist-database-migrations, pcmedicalist-optimistic-locking, pcmedicalist-pessimistic-locking, pcmedicalist-read-replicas, pcmedicalist-sharding, pcmedicalist-partitioning, pcmedicalist-query-optimization, pcmedicalist-replication, pcmedicalist-eventual-consistency, pcmedicalist-cap-theorem, pcmedicalist-backup-dr]
---

# PCMedicalist Database Deep Dive

## When to Use
- Designing a new table or query for baseLINE / any PCMedicalist Postgres service.
- Debugging slow queries, deadlocks, replication lag, or bloat.
- Writing or reviewing a migration (up AND down), a transaction, or a CDC/outbox integration.
- Deciding UUID vs serial PK, cursor vs offset pagination, soft vs hard delete, raw vs materialized view.
- Operating Postgres on the VPS (backup/restore/PITR/VACUUM) or planning a multi-region read topology.

## Portfolio Reality
PCMedicalist runs **PostgreSQL** behind the baseLINE Node/Express + Prisma app (PM2 on DreamHost VPS <DEPLOY_HOST>), with **Redis** for sessions/cache. There is currently a single primary; read replicas are planned (item 63). On-chain tip verification hits Base RPC. This skill covers the *operational + transactional* depth that the narrower indexing/locking/migration skills assume but don't fully document. Where a topic has its own skill (indexing, N+1, pooling, migrations, locking, replicas, sharding, partitioning, query-optimization, replication), this skill links rather than duplicates.

## Topics Covered
1. EXPLAIN ANALYZE
2. Full Table Scans
3. Down Migrations
4. Transactions
5. ACID
6. Isolation Levels
7. Deadlocks
8. Optimistic Locking
9. Pessimistic Locking
10. Normalization
11. Denormalization
12. Foreign Keys
13. Unique Constraints
14. Cascading Deletes
15. Soft Deletes
16. UUIDs vs Auto-Increment
17. Cursor-Based Pagination
18. Full-Text Search
19. Read Replicas
20. Replication Lag
21. Sharding
22. Partitioning
23. Hot Partitions
24. Postgres VACUUM
25. Materialized Views
26. Backups
27. Testing Your Restores
28. Point-in-Time Recovery
29. Change Data Capture
30. Outbox Pattern
31. ORM Lazy Loading

Each topic follows **Why → Pattern → Validate → Fix**.

---

## 1. EXPLAIN ANALYZE
**Why:** You cannot optimize blind. `EXPLAIN` (plan only) lies about costs; `EXPLAIN ANALYZE` runs the query and reports real timings + row counts so you see where time goes.

**Pattern:**
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT t.* FROM tips t
JOIN users u ON u.id = t."userId"
WHERE t."tenantId" = '...' AND t.status = 'pending'
ORDER BY t."createdAt" DESC LIMIT 20;
```
Read the plan top-down: look for `Seq Scan` on large tables, `actual rows` ≫ `estimated rows` (stale stats), high `Buffers: read=` (cache misses), and `actual time` concentration in one node.

**Validate:** After adding the right index, re-run; the `Seq Scan` becomes `Index Scan` and `actual time` total drops by ≥10×.

**Fix:** Capture plans in a `slow_queries.md` log. Use `auto_explain` in non-prod to auto-log plans over a threshold. Never ship an index change without a before/after `EXPLAIN ANALYZE`.

---

## 2. Full Table Scans
**Why:** A sequential scan on a large table is the #1 latency source. Sometimes correct (small table, majority returned) but usually a missing index or a non-sargable predicate.

**Pattern (sargable predicates):**
```sql
-- Non-sargable: index on created_at NOT used
WHERE DATE("createdAt") = '2026-01-01'        -- function wraps column
-- Sargable:
WHERE "createdAt" >= '2026-01-01' AND "createdAt" < '2026-01-02'
```
Also avoid `LIKE '%foo%'` (leading wildcard defeats index), `OR` across columns without composite index, and implicit type casts (`varchar = 123`).

**Validate:** `EXPLAIN ANALYZE` shows `Index Scan` / `Bitmap Heap Scan`, not `Seq Scan`, for the hot query on the big table.

**Fix:** Add a targeted index; rewrite the predicate to be sargable; if the column is low-cardinality, consider a partial index (`WHERE status='pending'`).

---

## 3. Down Migrations
**Why:** Up-only migrations mean you cannot roll back a bad deploy cleanly; a failed release strands the schema.

**Pattern (Prisma + raw SQL pair):**
```sql
-- migration 20260101_add_tips_status/up.sql
ALTER TABLE tips ADD COLUMN status varchar(20) NOT NULL DEFAULT 'pending';
-- migration 20260101_add_tips_status/down.sql
ALTER TABLE tips DROP COLUMN status;
```
Rules: every up has a reversible down (drop column, drop index, drop constraint — NOT data-destroying without a guard). Never write a down that drops data you can't recreate. Test the down on a copy before trusting it.

**Validate:** In a staging DB, apply up, then apply down; schema returns to the prior `pg_dump --schema-only` fingerprint.

**Fix:** If a migration has no down, author one immediately. Add CI that runs `prisma migrate diff` and rejects up-only changes lacking a reversible pair (except truly irreversible, which must be flagged + manually approved).

---

## 4. Transactions
**Why:** A multi-step write (tip record + ledger + notification) must be atomic — partial failure leaves inconsistent state.

**Pattern (Prisma):** 
```ts
await prisma.$transaction(async (tx) => {
  const tip = await tx.tips.create({ data });
  await tx.ledger.create({ data: { tipId: tip.id, amountWei } });
  await tx.notifications.create({ data: { userId: tip.creatorId, kind: 'tip' } });
  // throw here => all rolled back
});
```
Keep transactions **short** (no network calls / external API inside). For cross-service atomicity, use the Outbox pattern (topic 30) instead of a distributed transaction.

**Validate:** Force a failure mid-transaction (inject throw after step 2); confirm zero rows written for all three tables.

**Fix:** Wrap every multi-write operation in a transaction. Move external calls (Bankr, email) OUTSIDE the DB transaction, after commit, via outbox/queue.

---

## 5. ACID
**Why:** These four guarantees are why we use Postgres over a key-value store for financial data (tips, payouts).

- **Atomicity** — all-or-nothing (topic 4).
- **Consistency** — every committed transaction leaves the DB in a valid state (all constraints/FKs hold).
- **Isolation** — concurrent txns don't see each other's uncommitted writes (topic 6).
- **Durability** — a committed txn survives crashes (WAL + fsync).

**Pattern:** Trust Postgres defaults; don't disable `fsync` or `synchronous_commit` on the primary unless you accept potential data loss (acceptable only for ephemeral caches).

**Validate:** After a forced `kill -9` of Postgres mid-commit, a restarted DB has either the full txn or none — never a torn write (verify via a checksum row written in the same txn).

**Fix:** If durability was traded for speed, document the accepted risk. For tip/payout paths, `synchronous_commit = on` is non-negotiable.

---

## 6. Isolation Levels
**Why:** The wrong level causes dirty reads, non-repeatable reads, or phantom reads; the right level balances correctness vs throughput.

| Level | Dirty | Non-repeatable | Phantom | Notes |
|---|---|---|---|---|
| Read Uncommitted | ❌(PG: same as RC) | ✅ | ✅ | PG doesn't truly support |
| Read Committed (default) | — | ✅ | ✅ | fine for most |
| Repeatable Read | — | — | ✅(PG prevents) | snapshot isolation |
| Serializable | — | — | — | full predicate locks, slowest |

**Pattern:**
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- or in Prisma: prisma.$transaction(fn, { isolationLevel: 'RepeatableRead' })
```
Use **Serializable** only for critical financial invariants (e.g. ensuring a user can't double-spend within a window) and accept retry-on-serialization-failure.

**Validate:** Two concurrent txns at Serializable that would conflict → one gets `serialization_failure` (SQLSTATE 40001) and must retry. Confirm the retry loop handles it.

**Fix:** Default to Read Committed. Escalate to Repeatable Read / Serializable only where a demonstrated anomaly exists. Always implement a retry wrapper for serialization failures.

---

## 7. Deadlocks
**Why:** Two txns acquire locks in opposite order → both wait forever → Postgres kills one with `40P01`. Common in tip + payout + ledger writes.

**Pattern (consistent lock ordering):**
```ts
// ALWAYS lock in the same global order: users, then tips, then ledger
await tx.users.update({ where: { id }, data: {...} });
await tx.tips.update({ where: { id }, data: {...} });
await tx.ledger.update({ where: { id }, data: {...} });
```
Keep txns short, access tables in a fixed order, and use `SELECT ... FOR UPDATE` deliberately (topic 9). Set `lock_timeout` (e.g. 2s) so a deadlock-prone path fails fast instead of hanging.

**Validate:** A chaos test firing the two conflicting flows concurrently triggers at most one `40P01` (the loser retries successfully); no hang.

**Fix:** Detect `40P01` in the app, sleep+backoff, retry the whole txn. Audit code for varying lock orders. Add `lock_timeout` to long txns.

---

## 8. Optimistic Locking
**Why:** Prevent lost updates when two requests edit the same row, without holding a DB lock (better concurrency for low-contention paths).

**Pattern (version column):**
```ts
// schema: "version" integer default 0
const updated = await prisma.tips.update({
  where: { id, version: expectedVersion },   // where includes the version
  data: { status: 'confirmed', version: { increment: 1 } },
});
if (!updated) throw new ConflictError('stale_write');
```
If the version changed underneath, the `where` matches 0 rows → conflict → client retries with fresh state.

**Validate:** Two concurrent updates with the same `expectedVersion` → exactly one succeeds, the other gets a 409.

**Fix:** Add a `version`/`updatedAt`-compare column to mutable entities. Handle `0 rows updated` as a conflict, not a silent no-op.

---

## 9. Pessimistic Locking
**Why:** When contention is real (e.g. discharging a payout balance), you want to serialize access up front rather than gamble on optimistic retries.

**Pattern:**
```sql
BEGIN;
SELECT * FROM balances WHERE "userId" = $1 FOR UPDATE;   -- locks the row
-- mutate, then COMMIT releases
UPDATE balances SET amount = amount - $2 WHERE "userId" = $1;
COMMIT;
```
In Prisma: `prisma.balances.findFirst({ where, lock: { mode: 'pessimistic_write' } })`.

**Validate:** A second concurrent txn on the same balance BLOCKS until the first commits (or hits `lock_timeout`). No double-spend.

**Fix:** Use `FOR UPDATE` for balance/payout mutations. Always wrap in a short txn with `lock_timeout`. Never hold the lock across a network call.

---

## 10. Normalization
**Why:** Normal forms (1NF–3NF) eliminate redundancy/anomalies. baseLINE's tips/users/ledger should be 3NF: no repeating groups, no partial-key dependency, no transitive dependency.

**Pattern:** Separate concerns — `users`, `tips`, `creators`, `ledger` — with FK links, not duplicated column blobs. Store derived counts in a view, not a denormalized column you must keep in sync.

**Validate:** No table stores another table's attributes redundantly (e.g. `tips.creatorEmail` is a smell — join to `creators.email`).

**Fix:** If you find duplicated columns, extract to a related table + FK. Recompute derived data via view/materialized view.

---

## 11. Denormalization
**Why:** Reads that join 6 tables per request are slow. *Deliberate* denormalization (a cached `tip_count` on `creators`) trades write complexity for read speed — only where measured.

**Pattern:**
```sql
-- maintained by trigger, NOT app code
CREATE OR REPLACE FUNCTION update_creator_tip_count() RETURNS trigger AS $$
BEGIN
  UPDATE creators SET tip_count = (SELECT count(*) FROM tips WHERE "creatorId" = NEW."creatorId")
  WHERE id = NEW."creatorId";
  RETURN NEW;
END; $$ LANGUAGE plpgsql;
```
Prefer triggers over app-side maintenance (avoids drift). Document the invariant.

**Validate:** `creators.tip_count` always equals `SELECT count(*) FROM tips WHERE "creatorId"=...`. A reconciliation job confirms.

**Fix:** Don't denormalize prematurely. When you do, use a trigger + a nightly reconciliation check, not scattered app updates.

---

## 12. Foreign Keys
**Why:** FKs enforce referential integrity — a tip without a valid user is a data-corruption bug waiting to happen.

**Pattern:**
```sql
ALTER TABLE tips ADD CONSTRAINT fk_tips_user
  FOREIGN KEY ("userId") REFERENCES users(id) ON DELETE RESTRICT;
```
Always index the FK column (Postgres doesn't auto-index the child side). Choose the `ON DELETE` behavior deliberately (topic 14).

**Validate:** `INSERT INTO tips ("userId") VALUES (999999)` → `foreign_key_violation` (23503), not a silent orphan.

**Fix:** Add missing FKs to all relationship columns. Index each FK column. Verify no existing orphans before adding (`WHERE NOT EXISTS (SELECT 1 FROM users WHERE users.id = tips."userId")`).

---

## 13. Unique Constraints
**Why:** Prevent duplicate business keys — two creators with the same handle, two tips with the same on-chain txn hash (double-count!).

**Pattern:**
```sql
ALTER TABLE creators ADD CONSTRAINT uq_creators_handle UNIQUE (handle);
ALTER TABLE tips ADD CONSTRAINT uq_tips_txhash UNIQUE ("txHash");  -- idempotency guard
```
Unique constraints are also your idempotency mechanism for webhook replays (topic 30 outbox + this).

**Validate:** A second insert with a duplicate `txHash` → `unique_violation` (23505). The app treats it as "already processed".

**Fix:** Add unique constraints on every natural/business key. For case-insensitive handles, use `LOWER(handle)` unique index.

---

## 14. Cascading Deletes
**Why:** `ON DELETE CASCADE` is convenient but can wipe a subtree you didn't mean to (delete a creator → all their tips, ledger, payouts gone).

**Pattern:** Prefer `ON DELETE RESTRICT` (fail loud) for financial data; `CASCADE` only for true ownership (e.g. delete a user → delete their sessions).
```sql
ALTER TABLE tips DROP CONSTRAINT fk_tips_user;
ALTER TABLE tips ADD CONSTRAINT fk_tips_user
  FOREIGN KEY ("userId") REFERENCES users(id) ON DELETE RESTRICT;
```

**Validate:** Attempting to delete a creator with existing tips → `foreign_key_violation`, not silent cascade.

**Fix:** Audit all FKs; change `CASCADE`→`RESTRICT` on anything financial. Use soft-delete (topic 15) for user-facing "delete" anyway.

---

## 15. Soft Deletes
**Why:** Users expect "delete" to be recoverable; regulations may require retention; cascading hard deletes are dangerous (topic 14).

**Pattern:**
```sql
ALTER TABLE users ADD COLUMN "deletedAt" timestamptz;
CREATE INDEX idx_users_active ON users(id) WHERE "deletedAt" IS NULL;
-- all queries add: WHERE "deletedAt" IS NULL
```
Better: a Postgres **partial unique index** so a soft-deleted row frees the unique slot:
```sql
CREATE UNIQUE INDEX uq_creators_handle_active
  ON creators(LOWER(handle)) WHERE "deletedAt" IS NULL;
```

**Validate:** "Delete" sets `deletedAt`; the row stays, queries exclude it; the unique handle is reusable by a new active row.

**Fix:** Add `deletedAt` to entities that need undo. Centralize the `WHERE "deletedAt" IS NULL` in the data-access layer (Prisma `global` scope / query extension) so no query forgets it.

---

## 16. UUIDs vs Auto-Increment
**Why:** Serial PKs leak count + are enumerateable (feeds IDOR, topic 1 in security-fundamentals); UUIDv7 (time-ordered) gives unordered-to-attacker + index-friendly.

**Pattern:**
```sql
id uuid PRIMARY KEY DEFAULT gen_random_uuid();   -- or uuid_generate_v7() for time-ordered
```
Use UUIDv4 for external/object IDs (never expose serial PKs). Use a **bigserial** internal PK only if you need compactness and already hide it behind UUIDs externally.

**Validate:** External API returns `id` as a UUID; an attacker incrementing `?id=123→124` yields `404` (no sequential enumeration).

**Fix:** Migrate external IDs to UUIDv4/v7. Keep any existing serial PKs internal-only. See `pcmedicalist-security-fundamentals` topic 1 (IDOR) for the why.

---

## 17. Cursor-Based Pagination
**Why:** `OFFSET 100000 LIMIT 20` re-scans + discards 100k rows (O(n)); deep pages get slower and skip rows if data shifts mid-scan.

**Pattern (keyset/cursor):**
```sql
SELECT * FROM tips
WHERE "createdAt" < $lastSeenTs OR ("createdAt" = $lastSeenTs AND id < $lastSeenId)
ORDER BY "createdAt" DESC, id DESC
LIMIT 20;
-- next cursor = (lastRow.createdAt, lastRow.id)
```
Requires a composite index on `("createdAt" DESC, id DESC)`. Encode the cursor as opaque base64 (not raw values).

**Validate:** Fetch page 1 → cursor → page 2 returns the next 20 with zero overlap and stable order even if rows are inserted concurrently. `EXPLAIN` shows index-only scan, not offset discard.

**Fix:** Replace all `OFFSET` pagination on large tables with keyset. Provide both `first`/`after` and `last`/`before` for bidirectional.

---

## 18. Full-Text Search
**Why:** `LIKE '%query%'` can't use an index and is slow + dumb (no ranking, no stemming). Postgres has `tsvector` built in.

**Pattern:**
```sql
ALTER TABLE creators ADD COLUMN search tsvector
  GENERATED ALWAYS AS (to_tsvector('english', coalesce(handle,'')||' '||coalesce(bio,''))) STORED;
CREATE INDEX idx_creators_search ON creators USING GIN (search);
-- query:
SELECT * FROM creators
WHERE search @@ websearch_to_tsquery('english', $1)
ORDER BY ts_rank(search, websearch_to_tsquery('english', $1)) DESC
LIMIT 20;
```

**Validate:** A search for `crypto tip` returns creators mentioning both, ranked; `EXPLAIN` uses the GIN index; latency < 50ms on 100k rows.

**Fix:** If search is `ILIKE '%...%'`, add a `tsvector` column + GIN index. For multi-language or heavy search, consider a dedicated engine (Meili/Typesense) — but start with built-in FTS.

---

## 19. Read Replicas
**Why:** Offload read traffic (dashboards, leaderboards) from the primary so writes stay fast. baseLINE plans this (item 63).

**Pattern:**
```ts
// route reads explicitly
const creators = await replicaPrisma.creators.findMany({ where: { active: true } });
// writes ALWAYS go to primary
await primaryPrisma.tips.create({ data });
```
Use a connection pool per role. Never read-after-write on a replica without considering lag (topic 20).

**Validate:** A read-heavy dashboard query hits the replica (confirmed via `pg_stat_activity`/`application_name`); primary write throughput unaffected.

**Fix:** Stand up a streaming replica (`primary_conninfo` + `hot_standby=on`). Split the Prisma client into primary/replica instances. Route analytical reads to the replica.

---

## 20. Replication Lag
**Why:** A user updates their profile, refreshes, and sees the old value because the replica hasn't caught up. Lag also breaks read-after-write.

**Pattern:**
```ts
// read-your-own-writes: read from primary for the current user's own data
const me = await primaryPrisma.users.findUnique({ where: { id: req.user.id } });
// everyone else's public data can come from replica
```
Monitor `pg_stat_replication.replay_lag`. Alert if > 1s. For critical consistency, use `synchronous_commit` or read-from-primary for owner reads.

**Validate:** Under write load, `replay_lag` stays bounded; a just-written row is always readable by its owner (primary read).

**Fix:** Add lag monitoring + alerting. Use primary-read for owner data. If lag spikes, check replica CPU / WAL replay backlog / long-running queries blocking replay.

---

## 21. Sharding
**Why:** When one Postgres node can't hold the write volume (tens of thousands of tips/sec), split by shard key.

**Pattern:** Shard by `tenantId` / `creatorId` hash across N primaries. Route writes by `hash(creatorId) % N`. Use a sharding proxy (e.g. Citus, or app-level routing). See `pcmedicalist-sharding` for the dedicated skill.

**Validate:** Write throughput scales ~linearly with shard count; no cross-shard transactions for the hot path.

**Fix:** Only shard when vertical scaling + read replicas are exhausted. Plan the shard key to avoid hotspots (topic 23). Prefer Citus over hand-rolled routing.

---

## 22. Partitioning
**Why:** A single giant table (tips by date) makes deletes/vacuum slow and indexes bloated. Range partition by time.

**Pattern:**
```sql
CREATE TABLE tips (
  id uuid, "createdAt" timestamptz, ...
) PARTITION BY RANGE ("createdAt");
CREATE TABLE tips_2026_01 PARTITION OF tips
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```
Automate partition creation (pg_partman or a cron). Old partitions can be detached + archived cheaply.

**Validate:** `EXPLAIN` on a date-bounded query shows **partition pruning** (only relevant child tables scanned). Dropping a month = `DETACH` (ms), not `DELETE` (hours).

**Fix:** If a time-series table is unpartitioned and > 50M rows, add range partitioning. Backfill into partitions online (concurrent). See `pcmedicalist-partitioning`.

---

## 23. Hot Partitions
**Why:** A single partition/shard absorbs disproportionate traffic (e.g. one viral creator's tips all land in `creatorId=VIP`), starving others.

**Pattern:** Avoid monotonic shard keys that collide. For time partitioning, don't let all writes hit "today's" partition exclusively without the partition being sized for it. For sharding, salt the key if one entity dominates: `shard = hash(creatorId + salt)` and keep a lookup.

**Validate:** Per-partition/shard write distribution is roughly even; no single partition > 30% of writes.

**Fix:** If a hot partition emerges, re-key or split it. For a known-viral creator, pre-split their data or use a dedicated shard. Monitor per-partition row counts.

---

## 24. Postgres VACUUM
**Why:** MVCC leaves dead tuples; without `VACUUM`, tables bloat, indexes swell, and queries slow. `AUTOVACUUM` usually handles it — until it doesn't (long transactions block it).

**Pattern:**
```sql
-- manual if autovacuum falls behind
VACUUM (ANALYZE, VERBOSE) tips;
-- tune per-table for high-churn tables
ALTER TABLE tips SET (autovacuum_vacuum_scale_factor = 0.05, autovacuum_analyze_scale_factor = 0.02);
```
Watch `pg_stat_user_tables.n_dead_tup` vs `n_live_tup`; if dead ≫ live, vacuum is behind. A long-running transaction (even `idle in transaction`) blocks vacuum from reclaiming.

**Validate:** After `VACUUM ANALYZE`, `EXPLAIN` row estimates are accurate (no 1000× misestimate) and table size (via `pg_total_relation_size`) drops.

**Fix:** Kill idle-in-transaction connections (set `idle_in_transaction_session_timeout`). Tune autovacuum scale factors on churny tables. Add a bloat-monitoring dashboard.

---

## 25. Materialized Views
**Why:** An expensive aggregate (daily tip totals per creator) recomputed per request is wasteful.

**Pattern:**
```sql
CREATE MATERIALIZED VIEW creator_daily_tips AS
SELECT "creatorId", date_trunc('day', "createdAt") d, sum(amountWei) total
FROM tips GROUP BY 1, 2;
CREATE UNIQUE INDEX ON creator_daily_tips ("creatorId", d);
-- refresh (CONCURRENTLY needs the unique index, no lock):
REFRESH MATERIALIZED VIEW CONCURRENTLY creator_daily_tips;
```
Refresh on a schedule (cron) or incrementally. For real-time, prefer a maintained table (topic 11) over full refresh.

**Validate:** Dashboard reads the MV in < 5ms; `REFRESH CONCURRENTLY` doesn't block reads; data lag is within SLA (e.g. < 5 min).

**Fix:** Replace per-request aggregates with an MV + scheduled refresh. Use `CONCURRENTLY` to avoid read locks. Index the MV for fast lookups.

---

## 26. Backups
**Why:** A deleted table, a corrupted volume, or a bad migration = total loss without backups. baseLINE holds financial data — backups are non-negotiable.

**Pattern:**
```bash
# continuous: WAL archiving (for PITR, topic 28)
pg_basebackup -D /backups/base -Ft -z -P
# nightly dump
pg_dump --format=custom --no-owner pcmedicalist | gzip > /backups/daily/$(date +%F).dump.gz
```
Store backups OFF the VPS (object storage / separate region). Encrypt at rest. Retain ≥ 30 daily + 12 monthly.

**Validate:** Backup file exists, is non-zero, and is younger than 24h. (Then go to topic 27 — a backup you haven't restored is a hope, not a backup.)

**Fix:** If no automated backup exists, set up `pg_basebackup` + WAL archiving + a nightly `pg_dump` to offsite storage TODAY. See `pcmedicalist-backup-dr`.

---

## 27. Testing Your Restores
**Why:** Unverified backups fail exactly when you need them. A restore test proves the backup is valid AND that you know the procedure.

**Pattern:**
```bash
# on a throwaway instance
pg_restore --format=custom -d restore_test /backups/daily/2026-01-01.dump.gz
psql restore_test -c "SELECT count(*) FROM tips;"   # matches prod count?
```
Run this **monthly** (or on a schedule). Record the restore time — that's your real RTO input (topic 28/151).

**Validate:** Restored DB has row counts + checksums matching prod for sampled tables. Restore completes within the documented RTO.

**Fix:** Add a scheduled restore drill to cron. Alert if the drill fails. Keep a runbook with exact commands.

---

## 28. Point-in-Time Recovery (PITR)
**Why:** A bad migration at 14:32 destroyed data; the nightly backup only has 00:00 state. PITR rewinds to 14:31.

**Pattern:** Requires WAL archiving enabled (`archive_mode = on`, `archive_command` shipping WAL to safe storage).
```bash
# restore base backup, then replay WAL up to target
restore_command = 'cp /wal-archive/%f %p'
recovery_target_time = '2026-01-01 14:31:00'
```
Base backup + continuous WAL = PITR window = however long you retain WAL.

**Validate:** Restore to a target time 1 minute before a known bad write; the bad row is absent; good rows present.

**Fix:** Enable WAL archiving if not on. Retain WAL for your RPO window (e.g. 7–30 days). Document the PITR runbook + test it (topic 27).

---

## 29. Change Data Capture (CDC)
**Why:** Other systems (search index, analytics, cache) need to react to DB changes without polling.

**Pattern (logical replication / WAL):**
```sql
-- Debezium / Supabase Realtime / Postgres logical replication
CREATE PUBLICATION pcm_cdc FOR TABLE tips, users, ledger;
-- consumer streams changes -> updates MeiliSearch / emits to Kafka
```
Use logical decoding (`wal_level = logical`). CDC feeds the Outbox consumer (topic 30) and downstream caches.

**Validate:** Insert a row in `tips` → within seconds the search index reflects it; no full-table poll ran.

**Fix:** If you're polling for changes, switch to logical replication / CDC. Set `wal_level=logical`. Secure the replication slot (monitor for unconsumed WAL buildup).

---

## 30. Outbox Pattern
**Why:** You must write to the DB AND publish an event (tip confirmed → notify creator), atomically. Two-phase commit is fragile; publishing before commit risks a rollback orphan; committing then crashing before publish loses the event.

**Pattern:**
```sql
BEGIN;
INSERT INTO tips (...) VALUES (...);
INSERT INTO outbox (id, topic, payload, "createdAt")
  VALUES (gen_random_uuid(), 'tip.confirmed', '{"tipId":"..."}', now());
COMMIT;
-- SEPARATE worker polls outbox (or CDC streams it), publishes to queue, then deletes row
```
The outbox row is written in the SAME transaction as the business data → atomic. A relay worker (or CDC, topic 29) drains it. Guarantees at-least-once delivery (topic 109) → consumer must be idempotent (unique constraint, topic 13).

**Validate:** Force a crash after COMMIT but before publish → the outbox row remains → worker later publishes it. No lost events, no double-publish of the same row (idempotency key).

**Fix:** Add an `outbox` table to any service that writes-then-publishes. Replace inline `queue.publish()` inside transactions with outbox writes. Build the relay worker + idempotent consumer.

---

## 31. ORM Lazy Loading
**Why:** An ORM that lazily fetches relations (`user.tips` triggers N queries) produces N+1 explosions (see `pcmedicalist-nplus1`).

**Pattern (eager / explicit):**
```ts
// Prisma: explicit include, no lazy navigation
const creators = await prisma.creators.findMany({
  include: { tips: { take: 5, orderBy: { createdAt: 'desc' } } },
});
// disable lazy loading entirely if the ORM supports it (e.g. set lazyLoadingEnabled: false)
```
Prefer `select`/`include` at the query site. Never iterate a collection that triggers a per-row query.

**Validate:** A request that renders 20 creators + their tips issues 1–2 queries (check the query log), not 21. See `pcmedicalist-nplus1` for the dedicated skill + detection.

**Fix:** Find N+1s via query-logging / APM. Add `include`/`select` or a batch loader (DataLoader). Disable lazy loading in ORM config so the bug fails loud.

---

## Validation Checklist (all 31 topics)
- [ ] Hot queries have `EXPLAIN ANALYZE` before/after proof (1)
- [ ] No `Seq Scan` on large-table hot paths (2)
- [ ] Every migration has a reversible down (3)
- [ ] Multi-writes wrapped in `$transaction` (4)
- [ ] `synchronous_commit=on` for financial writes (5)
- [ ] Isolation level chosen per-path; serialization retries handled (6)
- [ ] Lock order consistent; `lock_timeout` set (7)
- [ ] Mutable entities have `version` column (8)
- [ ] Balance/payout mutations use `FOR UPDATE` (9)
- [ ] 3NF schema, no duplicated attrs (10)
- [ ] Deliberate denorm maintained by trigger + reconciliation (11)
- [ ] FKs present + indexed (12)
- [ ] Unique constraints on business keys (13)
- [ ] Financial FKs `RESTRICT`, not `CASCADE` (14)
- [ ] Soft-delete `deletedAt` + partial unique index (15)
- [ ] External IDs are UUIDv4/v7 (16)
- [ ] Large tables use cursor pagination (17)
- [ ] Search uses `tsvector` + GIN (18)
- [ ] Reads split primary/replica (19)
- [ ] Owner reads from primary (20)
- [ ] Shard key avoids hotspots (21)
- [ ] Time-series partitioned + pruned (22)
- [ ] No partition > 30% of traffic (23)
- [ ] Autovacuum tuned; no idle-in-tx blocks (24)
- [ ] Expensive aggregates = MV + CONCURRENTLY refresh (25)
- [ ] Automated offsite backups (26)
- [ ] Monthly restore drill (27)
- [ ] WAL archiving for PITR (28)
- [ ] CDC / logical replication for downstream (29)
- [ ] Outbox for write-then-publish (30)
- [ ] No lazy-loading N+1s (31)

## Fix Recipes
- **"Query is slow"** → topics 1–2: `EXPLAIN ANALYZE`, kill the Seq Scan with a sargable predicate + index.
- **"Bad deploy stranded schema"** → topic 3: author the down migration, test it.
- **"Double-spend / lost update"** → topics 4, 7–9: transaction + lock ordering + optimistic/pessimistic.
- **"Orphaned rows"** → topics 12–14: add FKs, RESTRICT, unique constraints.
- **"Deep pagination times out"** → topic 17: keyset pagination.
- **"Search is `ILIKE`"** → topic 18: tsvector + GIN.
- **"We have no backups"** → topics 26–28: basebackup + WAL + offsite + restore drill + PITR.
- **"Event published but txn rolled back"** → topic 30: outbox.
- **"N+1 in the loop"** → topic 31 + `pcmedicalist-nplus1`: explicit include / DataLoader.

## Cross-References
- `pcmedicalist-database-indexing` — topics 1–2 index design
- `pcmedicalist-nplus1` — topic 31
- `pcmedicalist-connection-pooling` — pool sizing for replica/primary split (19)
- `pcmedicalist-database-migrations` — topic 3 deeper
- `pcmedicalist-optimistic-locking` / `pcmedicalist-pessimistic-locking` — topics 8–9 deeper
- `pcmedicalist-read-replicas` — topics 19–20 deeper
- `pcmedicalist-sharding` — topic 21 deeper
- `pcmedicalist-partitioning` — topic 22 deeper
- `pcmedicalist-query-optimization` — topics 1–2, 18 deeper
- `pcmedicalist-replication` — topics 19–20, 29 deeper
- `pcmedicalist-eventual-consistency` / `pcmedicalist-cap-theorem` — topics 19–20, 29 consistency model
- `pcmedicalist-backup-dr` — topics 26–28 disaster recovery
- `pcmedicalist-security-fundamentals` — topic 16 (UUIDs/IDOR), topic 13 (unique = idempotency)
