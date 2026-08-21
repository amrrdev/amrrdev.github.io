---
title: "Database Migrations at Scale: Zero-Downtime Schema Changes and What Companies Actually Do"
date: 2026-07-05
categories: [Database]
tags: [migrations, schema-design, zero-downtime, database]
---
## Why Migrations Break

A migration that runs in 200ms on your local machine takes 30 minutes on the production table with 50 million rows. During those 30 minutes, your `ALTER TABLE` holds a lock that blocks every read and write to that table. Your API stops responding. Your queues fill up. Users get 503 errors.

This is not theoretical. It happens every day at companies of every size. A single unsafe migration in production can cause more downtime than all your server crashes combined.

The problem is that migrations are rarely tested at production scale. You run them in development against 100 rows. You run them in staging against 10,000 rows. Neither environment reveals that `ALTER TABLE ... ADD COLUMN ... DEFAULT` on Postgres rewrites every row in the table, holding an `ACCESS EXCLUSIVE` lock for the entire duration.

This article covers how companies handle schema changes on live databases with millions of queries per second — what's safe, what's not, the tools that make it possible, and the patterns that prevent downtime.

---

## What Actually Happens When You Run ALTER TABLE

Different databases handle schema changes differently. Understanding the internals is the difference between a safe migration and a page at 3 AM.

### Postgres

Postgres uses a lock system for DDL changes. The lock level depends on the operation:

```
Safe — ShareUpdateExclusiveLock (doesn't block reads/writes):
  CREATE INDEX CONCURRENTLY
  ADD COLUMN (nullable, no default)
  DROP INDEX CONCURRENTLY (since Postgres 14)

Dangerous — AccessExclusiveLock (blocks ALL reads and writes):
  ADD COLUMN with DEFAULT or NOT NULL
  ALTER COLUMN ... TYPE
  DROP COLUMN
  ADD PRIMARY KEY
  ADD FOREIGN KEY
```

The dangerous ones acquire `AccessExclusiveLock`, which means no other transaction — not even a simple `SELECT` — can access the table until the operation completes. For a table with 50 million rows, `ADD COLUMN ... DEFAULT 'hello'` rewrites every row, which takes minutes or hours.

```sql
-- This rewrites the entire table. Blocks everything.
ALTER TABLE users ADD COLUMN is_verified boolean NOT NULL DEFAULT false;

-- This is instant. No row rewrite. No block.
ALTER TABLE users ADD COLUMN is_verified boolean;
```

**The key insight:** adding a column `WITHOUT a default value` is metadata-only in Postgres 11+. Postgres stores the column definition but doesn't write anything to existing rows — they return NULL on read. Adding a default value, however, requires writing that default to every existing row.

### MySQL (InnoDB)

MySQL 8.0 introduced "instant DDL" for some operations:

```
Instant (metadata only, no table rebuild):
  ADD COLUMN (non-last position only in 8.0.29+)
  DROP COLUMN (8.0.29+)
  ADD/DROP DEFAULT
  RENAME COLUMN

In-Place (rebuilds table, allows concurrent writes):
  ADD INDEX
  DROP INDEX
  ADD FOREIGN KEY
  RENAME TABLE

Copy (rebuilds table, blocks writes):
  ADD COLUMN with NOT NULL (requires table rebuild in MySQL)
  ALTER COLUMN ... TYPE
```

MySQL's `INPLACE` algorithm allows concurrent reads and writes during the operation. The `COPY` algorithm blocks writes while it copies data.

### The Common Thread

Every operation that requires a **table rebuild** is dangerous. Every operation that is **metadata-only** is safe. The distinction between the two is what makes a migration zero-downtime or not.

---

## The Expand-Contract Pattern

This is the foundation of every zero-downtime migration. The idea: never make a breaking schema change and a backward-incompatible code change at the same time. Instead, split the migration into three phases.

```
Phase 1 — Expand:
  Add the new schema alongside the old
  Old code still works with the old schema
  New code can use the new schema

Phase 2 — Migrate:
  Backfill historical data
  Dual-write to both old and new
  Verify data consistency

Phase 3 — Contract:
  Remove the old schema
  Deploy the final code that only uses the new schema
```

### Example: Renaming a Column

**Phase 1 — Expand** (deploy schema first, no code change):

```sql
-- Add the new column. Existing code still uses `name`.
-- New code can use `full_name`. Both are kept in sync.
ALTER TABLE users ADD COLUMN full_name text;
CREATE INDEX CONCURRENTLY idx_users_full_name ON users (full_name);
```

Deploy this migration. No code changes yet. The database now has both `name` and `full_name`.

**Phase 2 — Migrate** (deploy code that dual-writes):

```go
func (s *UserService) CreateUser(name string) error {
    // Write to both columns
    _, err := s.db.Exec(
        "INSERT INTO users (name, full_name) VALUES ($1, $1)",
        name,
    )
    return err
}

func (s *UserService) GetUser(id int) (*User, error) {
    // Read from the new column
    row := s.db.QueryRow("SELECT full_name FROM users WHERE id = $1", id)
    // ...
}
```

Also run a backfill to populate `full_name` for all existing rows:

```sql
-- Backfill in batches, not one giant UPDATE
UPDATE users SET full_name = name
WHERE full_name IS NULL AND id BETWEEN 0 AND 10000;

UPDATE users SET full_name = name
WHERE full_name IS NULL AND id BETWEEN 10001 AND 20000;
-- ... continue in batches
```

**Phase 3 — Contract** (deploy after verifying everything works):

```sql
-- Remove the old column
ALTER TABLE users DROP COLUMN name;
```

GitHub used exactly this pattern when they renamed `users.primary_email` to `users.email`. The entire migration took weeks — phase 1 was deployed, then the code was updated to read from `email` and write to both, then after a month of monitoring, the old column was dropped.

### Why Three Phases

The fundamental principle: **schema and code must always be backward-compatible.** At any point during the deployment, both the old and new versions of your application are running (canary deployments, rolling updates, etc.). If you deploy a code change that expects a column that doesn't exist yet, or a schema change that removes a column the old code still uses, you get errors.

```
Correct order:
  1. Expand schema (add column, add index)
  2. Deploy new code (reads from new, writes to both)
  3. Backfill historical data
  4. Contract schema (drop old column, drop old index)

Wrong order (causes downtime):
  1. Deploy new code that expects new column
  2. New column doesn't exist → errors
  
Also wrong:
  1. Drop old column
  2. Old code still reads old column → errors
```

---

## Schema Operations: Safe vs Unsafe by Database

### Postgres Safe Operations

These acquire a `ShareUpdateExclusiveLock` — they allow concurrent reads and writes:

```sql
-- Safe: metadata only, instant
ALTER TABLE users ADD COLUMN email text;

-- Safe: builds index in background, doesn't block writes
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);

-- Safe: Drops index without blocking (Postgres 14+)
DROP INDEX CONCURRENTLY idx_users_email;

-- Safe: validates existing constraint without blocking
ALTER TABLE users VALIDATE CONSTRAINT fk_org;
```

### Postgres Unsafe Operations

These acquire `AccessExclusiveLock` — they block everything:

```sql
-- UNSAFE: rewrites every row, blocks all access for minutes/hours
ALTER TABLE users ADD COLUMN email text DEFAULT '' NOT NULL;

-- UNSAFE: rewrites every row
ALTER TABLE users ALTER COLUMN email TYPE varchar(500);

-- UNSAFE: blocks writes during validation
ALTER TABLE users ADD CONSTRAINT fk_org
  FOREIGN KEY (org_id) REFERENCES orgs (id);
  -- Safer alternative: add without validation, then VALIDATE CONCURRENTLY
```

### MySQL Safe Operations (8.0+)

```sql
-- Safe: instant (metadata only, no lock)
ALTER TABLE users ADD COLUMN email text, ALGORITHM=INSTANT;

-- Safe: in-place rebuild with concurrent access
ALTER TABLE users ADD INDEX idx_email (email), ALGORITHM=INPLACE, LOCK=NONE;

-- Safe: rename is instant
ALTER TABLE users RENAME COLUMN name TO full_name, ALGORITHM=INSTANT;
```

### MySQL Unsafe Operations

```sql
-- UNSAFE: COPY algorithm, blocks all writes
ALTER TABLE users MODIFY COLUMN email varchar(500), ALGORITHM=COPY;

-- UNSAFE: requires table rebuild, blocks writes
ALTER TABLE users DROP COLUMN name, ALGORITHM=INPLACE;
-- DROP COLUMN in MySQL requires rebuild even with ALGORITHM=INPLACE
```

---

## Tools That Prevent Downtime

You don't have to do this manually. Several tools automate the process, and they're used by companies operating at enormous scale.

### gh-ost (GitHub's Online Schema Migration)

gh-ost is the most widely used online schema migration tool for MySQL. GitHub built it because `pt-online-schema-change` (Percona's tool) used database triggers, which caused performance problems at their scale.

**How it works:**

```
1. gh-ost connects to the MySQL replica (not the master)
2. Creates a shadow table (`_users_gho`) with the new schema
3. Reads the binary log stream from the replica
4. Applies every INSERT/UPDATE/DELETE from the original table to the shadow table
5. Backfills existing data in chunks (1000 rows at a time)
6. When done, renames the tables atomically:
   `users` → `_users_del`, `_users_gho` → `users`
7. Drops the old table (_users_del)
```

The key design decisions:

- **No triggers.** Triggers add overhead to every write on the original table. gh-ost reads the binary log instead, which is zero-impact on the master.
- **Connects to a replica for reading.** The heavy work (reading existing data) happens on a replica, not the master.
- **Throttling.** gh-ost monitors replication lag, master load, and disk space. It pauses automatically if any metric exceeds thresholds.

```bash
# gh-ost command for adding an index (real example)
gh-ost \
  --host="replica1.example.com" \
  --database="production" \
  --table="users" \
  --alter="ADD INDEX idx_email (email)" \
  --execute
```

**Used by:** GitHub, Square, Etsy, many others. GitHub has run thousands of gh-ost migrations on production tables with billions of rows.

### pgroll (Xata's Migration Tool for Postgres)

pgroll is newer but represents the best approach for Postgres. Instead of creating a shadow table, it uses Postgres views to make schema changes transparent.

**How it works:**

```
1. Creates the new column in a "sandbox" schema (separate namespace)
2. Creates a view that UNIONs the old and new columns
3. The application reads/writes through the view — it's unaware of the migration
4. Backfills data from old column to new column in batches
5. When backfill is complete, "switches" the view to only use the new column
6. Drops the old column
```

The critical feature: **instant rollback.** Because pgroll never drops the old schema until you explicitly commit, rolling back is just a metadata operation — switch the view back.

```bash
# pgroll: add a column, backfill, commit
pgroll add users full_name text --default="name"

# If something goes wrong:
pgroll rollback users full_name
```

**Used by:** Xata. Gaining adoption in Postgres-heavy shops.

### pt-online-schema-change (Percona)

The predecessor to gh-ost. Uses database triggers to keep the shadow table in sync.

```
Pros:
  - Mature, battle-tested
  - Works with older MySQL versions

Cons:
  - Triggers add overhead to every write on the original table
  - Can cause replication lag at high write volumes
  - Slower than gh-ost for large tables
```

### Liquibase / Flyway

These are migration management tools, not online schema change tools. They track which migrations have been applied but **do not** make unsafe operations safe.

```xml
<!-- Liquibase migration — still dangerous if the ALTER blocks -->
<changeSet id="add-column" author="dev">
    <addColumn tableName="users">
        <column name="email" type="varchar(255)"/>
    </addColumn>
</changeSet>
```

The responsibility is still on you to know whether `addColumn` is safe for your database version. Liquibase and Flyway just execute whatever SQL you give them, with optional locking to prevent concurrent migrations.

### How Companies Actually Combine These

The real-world process at a company like Shopify or GitHub:

```
1. Write the migration using application-level tooling
2. If the operation is safe (metadata-only), run directly against DB
3. If the operation requires a table rebuild:
   a. For MySQL: use gh-ost with throttling
   b. For Postgres: use pgroll or manual expand-contract
4. Run on a replica first
5. Verify data consistency between old and new
6. Run the cutover
7. Monitor for replication lag, error rates, query latency
```

---

## Real-World Migrations from Companies

### GitHub: Renaming `primary_email` to `email`

GitHub needed to rename a column used across their entire codebase. The table had 50M+ rows and received thousands of queries per second.

**The approach:**

1. **Phase 1 (Expand):** Added `email` column alongside `primary_email`. Zero-downtime — adding a nullable column without a default is instant.

2. **Phase 1.5 (Backfill):** Ran a backfill to populate `email` from `primary_email` in batches of 1000 rows. Ran during low traffic. Took hours but didn't block anything.

3. **Phase 2 (Dual-write):** Deployed code that wrote to both columns, read from `email`. Old code still read from `primary_email`. Both worked.

4. **Phase 3 (Verify):** Ran consistency checks comparing `email` vs `primary_email` for every row. Fixed discrepancies.

5. **Phase 4 (Contract):** After a month, deployed code that only used `email`. Then dropped `primary_email`.

**Total duration:** ~4 weeks. Zero downtime.

**Key lesson:** A column rename that would take 200ms in a migration file took a month in production. This is normal at scale.

### Shopify: Changing Primary Key from INT to BIGINT

Shopify hit the INT limit on their `orders` table — they had more than 2.1 billion orders. They needed to change the primary key from `INT` to `BIGINT`.

This is one of the hardest migrations possible because the primary key is referenced by every foreign key, every index, and every query. You can't just `ALTER TABLE ... MODIFY COLUMN`.

**How they did it:**

1. Used gh-ost to create a new table (`_orders_new`) with `BIGINT` primary key
2. gh-ost streamed all changes from the live table to the shadow table via binlog
3. All foreign key constraints were dropped and recreated to point to the new table
4. The cutover renamed `_orders_new` → `orders` atomically
5. Application code was updated to use the new table

**The tricky part:** application code stored order IDs in memory, caches, and queues. While the migration ran, some parts of the system still had INT-sized IDs. Shopify had to coordinate the cutover with a code deployment that flushed all caches and restarted workers.

**Key lesson:** The schema migration itself is only half the work. You also have to handle all the places where the old schema is baked into your code, caches, and queues.

### Stripe: Schema Changes Weeks Before Code

Stripe's approach is the most conservative and the safest: **schema changes are deployed weeks before the code changes that depend on them.**

```
Week 1:  Deploy migration that adds the new column
         No code changes. Nobody uses the new column.
         Monitor for any issues.

Week 2:  Start dual-writing to the new column.
         Old code writes to old column. New code (not deployed yet) will write to both.
         Verify data is consistent.

Week 3:  Deploy code that reads from the new column.
         Old code paths are removed.
         Old column is now unused.

Week 4:  Drop the old column (if applicable).
```

This seems slow. But Stripe processes billions of dollars in payments. A single minute of inconsistency costs real money. The week-long delays ensure that if a migration causes a problem, they have time to catch it before it affects anything.

**Key lesson:** The cost of a migration mistake at Stripe's scale exceeds the cost of the slow migration. Speed is not the goal — correctness is.

### Uber: Schema Registry and Migration Safety

Uber encountered the migration problem at massive scale: hundreds of services, thousands of tables, thousands of engineers making schema changes. They built a **Schema Registry** — a centralized system that tracks every schema change across the entire company.

**How it works:**

```
1. Engineer submits a migration to the Schema Registry
2. Schema Registry analyzes the migration:
   - Is this safe for MySQL? (table size, lock duration, replication lag)
   - Does this violate any existing constraints?
   - Will this change affect downstream consumers?
3. If unsafe, the migration is rejected or queued for a maintenance window
4. If safe, the Schema Registry coordinates the rollout:
   - Apply to replicas first
   - Wait for verification
   - Promote to the master
```

This prevented incidents like: "Engineer A drops a column that Service B depends on." The Schema Registry tracks column-level dependencies across services, something no migration tool does automatically.

**Key lesson:** At scale, migrations are a coordination problem as much as a technical one. You need to know who depends on the data you're changing.

### Instagram: Migrating from Postgres to Cassandra

Instagram had to migrate all their data from Postgres to Cassandra while serving hundreds of millions of users. The migration had to be invisible to users — no downtime, no data loss, no increased latency.

**Their approach:**

```
1. Write path: dual-write to Postgres and Cassandra
2. Read path: read from Postgres (the trusted source)
3. Background: migrate historical data from Postgres to Cassandra
4. Verification: continuously compare Postgres vs Cassandra data
5. Cutover: switch reads from Postgres to Cassandra
6. Remove: stop dual-writing to Postgres
```

This is the expand-contract pattern applied to an entire database migration, not just a column rename. The process took months. Dual-writes ran for weeks before the cutover. Verification ran continuously during the entire period.

**Key lesson:** Database migrations at this scale are not events — they are processes that take weeks or months.

---

## Rollback Strategies

The best rollback is the one you never need because your migration was already backward-compatible. But things still go wrong.

### Strategy 1: Forward-Only (Roll Forward)

Do not write rollback migrations. Instead, write a **new migration** that fixes the problem.

```
Migration 1 (add column email text):
  Applied successfully
  → Bug: the column is being populated with wrong data

Migration 2 (add column user_email, backfill from correct source):
  Applied successfully
  → Deploy code that reads from user_email

Migration 3 (drop column email):
  Applied after everything is verified
```

This is the simplest approach. It works because all your migrations should be backward-compatible anyway — so there's nothing to roll back to. You just add another change.

### Strategy 2: Expand-Contract Natural Rollback

If you used the expand-contract pattern, rollback is natural: as long as both schemas exist and old code is still deployed, you can revert the code change without reverting the schema change.

```
Expand: added column full_name, kept column name
Deploy: code that reads from full_name
→ Bug found: data is wrong

Rollback: deploy code that reads from name again
  → Contract (drop full_name) is deferred or never done
  → No data loss, no downtime
```

This is why expand-contract is the standard pattern. The rollback is just a code deployment, not a database operation.

### Strategy 3: pgroll's One-Command Rollback

pgroll keeps the old schema alive as a Postgres view until you explicitly `commit` the migration. Rollback is instant:

```bash
pgroll add users full_name text
# ... migration runs, but old schema is still available
pgroll rollback users full_name  # instant, just drops the view
```

### Strategy 4: gh-ost Cutover Reversal

gh-ost has a `--reversibly` option that keeps the old table around after the cutover:

```bash
gh-ost --alter="..." --reversibly --execute
# After cutover, both tables exist:
# users → new schema (with the change)
# _users_del → old schema (for rollback)
```

If the migration causes problems, you rename the tables back:

```sql
RENAME TABLE users TO _users_failed,
             _users_del TO users;
```

---

## The Golden Rule of Zero-Downtime Migrations

> **Never deploy schema and code changes in the same deployment.**

Every migration follows this pattern:

```
1. Deploy schema change (expand)
   → Old code works, new code doesn't exist yet
   → Monitor

2. Deploy old code that tolerates both old and new schema (dual-write)
   → Now old and new data paths both work
   → Monitor

3. Deploy new code that only reads from new schema
   → Old schema is now unused
   → Monitor

4. Deploy schema change to remove old schema (contract)
   → Nothing depends on the old schema anymore
   → Done

Wait between each step. How long? At least until you've verified that:
  - Error rates are normal
  - Data is consistent
  - Query latency is normal
  - Replication lag is normal
```

This is slow. For a simple column rename, this process takes 1-4 weeks at companies like GitHub and Stripe. But it's the only way to guarantee zero downtime.

---

## Migrations at Scale: The Process

### Step 1: Classify the Migration

Before writing any SQL, determine whether the migration is safe or unsafe:

```
Safe — deploy directly:
  - Add nullable column (no default)
  - Create index CONCURRENTLY (Postgres)
  - Add index with ALGORITHM=INPLACE, LOCK=NONE (MySQL)
  - Rename column (MySQL 8.0+)
  - Add/DROP DEFAULT

Unsafe — use expand-contract or an online tool:
  - Add column with default
  - Change column type
  - Add NOT NULL to existing column
  - Drop column
  - Add primary key
  - Add foreign key
```

### Step 2: Test at Production Scale

Production tables behave differently than staging tables. Before running a migration:

- **Size estimation:** How many rows? How large is the table on disk? How many indexes?
- **Lock impact:** If this migration blocks writes for 5 minutes, what happens to your system?
- **Replica test:** Run the migration on a read replica first. Measure how long it takes.
- **Staging with production data:** If possible, restore a production backup to staging and run the migration there.

### Step 3: Backfill Safely

Backfilling is the most dangerous part of a migration because it's a large write operation running alongside production traffic.

```
Bad backfill:
  UPDATE users SET full_name = name WHERE full_name IS NULL;
  → Single UPDATE locks millions of rows
  → Blocks reads and writes to those rows
  → Causes replication lag
  → Might take hours

Good backfill:
  -- Batch by primary key, small chunks
  UPDATE users SET full_name = name
  WHERE full_name IS NULL AND id BETWEEN 0 AND 1000;
  -- Wait 100ms
  UPDATE users SET full_name = name
  WHERE full_name IS NULL AND id BETWEEN 1001 AND 2000;
  -- ... continue in batches of 1000
  -- Sleep between batches to let the database breathe
```

Tools like gh-ost and pgroll handle this batching automatically. If you're doing it manually, the key is: **batch size that keeps replication lag under control**, and **sleep between batches to give the database time to process other work.**

### Step 4: Verify Data Consistency

After the backfill and before the cutover, verify that old and new data match:

```sql
-- Count mismatches
SELECT COUNT(*) FROM users
WHERE full_name IS NOT NULL AND full_name != name;
```

For large tables, sample instead of scanning everything:

```sql
-- Sample-based verification
SELECT COUNT(*) FROM users TABLESAMPLE BERNOULLI(1)
WHERE full_name IS NOT NULL AND full_name != name;
```

### Step 5: Cutover

The cutover is the moment when the application switches from the old schema to the new one. It should be:

- **Atomic:** Either all reads use the new schema, or none do. No intermediate state.
- **Monitorable:** Error rates, latency, and data correctness should be watched for at least 30 minutes after cutover.
- **Reversible:** If something goes wrong, you should have a plan to switch back.

### Step 6: Monitor After Migration

The migration isn't done when the SQL completes. Monitor for at least a week:

- **Query performance:** Are queries using the new indexes correctly?
- **Replication lag:** Did the migration cause a backlog?
- **Error rates:** Are there any errors related to missing columns, wrong types, or constraint violations?
- **Disk usage:** Did the migration increase table size significantly?

---

## What to Actually Do

1. **Classify every migration** as safe or unsafe before writing it. If you're not sure, assume it's unsafe and use the expand-contract pattern.

2. **Use expand-contract for every breaking change.** Add the new schema first, dual-write, verify, then remove the old schema. Never combine schema and code changes in the same deploy.

3. **Backfill in batches.** Never run `UPDATE table SET col = val WHERE col IS NULL` on a production table. Batch by primary key, sleep between batches, and monitor replication lag.

4. **Use gh-ost for MySQL.** If you're on MySQL and an ALTER will cause downtime, gh-ost is the standard. It's been proven at GitHub, Square, and Etsy on tables with billions of rows.

5. **Use pgroll or manual expand-contract for Postgres.** Postgres's concurrent index creation and metadata-only DDL make many operations safe. For the rest, expand-contract is manageable.

6. **Test on a replica first.** Before running any migration on the master, run it on a read replica. Measure the time, check the lock impact, verify no errors.

7. **Wait between phases.** Between expanding and contracting, wait long enough to verify everything works. A week is normal. Don't rush.

8. **Verify data consistency.** After backfilling, compare old and new data. Use sampling for large tables. Fix discrepancies before cutting over.

9. **Plan the rollback.** Know exactly what you'll do if the migration causes problems. Is it a forward fix? A revert? A pgroll rollback? Write it down before you start.

10. **Prefer additive changes.** Adding a column is always safer than modifying or removing one. When possible, design your system so that schema changes are always additive — add a new column, write to both, remove the old one months later.

---

## Key Takeaways

**Migrations lock tables.** An `ALTER TABLE` that rewrites rows blocks all reads and writes on that table. This is the most common cause of database-related downtime.

**Not all ALTER TABLE operations are equal.** Some are metadata-only (instant, no lock). Others rewrite every row (minutes to hours, blocks everything). Know which is which for your database.

**The expand-contract pattern** is the foundation of zero-downtime migrations. Add new schema, dual-write, backfill, verify, then remove old schema. Never combine schema and code changes.

**gh-ost** (MySQL) and **pgroll** (Postgres) automate online schema changes. gh-ost uses binlog streaming instead of triggers to avoid write overhead. pgroll uses Postgres views for transparent zero-downtime migrations.

**Backfill in batches.** Large UPDATE statements block rows, cause replication lag, and take too long. Batch by primary key, sleep between batches, and monitor replication lag.

**GitHub, Shopify, and Stripe** all use week-long, multi-phase migrations for schema changes that look simple in SQL. The cost of a mistake at their scale exceeds the cost of a slow migration.

**The golden rule:** Never deploy schema and code changes in the same deployment. Expand the schema first, deploy the code, then contract the old schema.

**Roll forward, not backward.** Most migrations should not have a rollback script. Instead, write a new migration that fixes the problem. Expand-contract naturally supports rollback by keeping both schemas alive.
