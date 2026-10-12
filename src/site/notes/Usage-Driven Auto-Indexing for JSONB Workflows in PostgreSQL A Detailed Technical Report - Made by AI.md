---
{"dg-publish":true,"permalink":"/usage-driven-auto-indexing-for-jsonb-workflows-in-postgre-sql-a-detailed-technical-report-made-by-ai/","dg-note-properties":{}}
---

## 1. Executive Summary

This report presents a complete architecture for **usage-driven auto-indexing** in a hybrid relational-JSONB PostgreSQL system, designed for environments where no human database administrator is available to approve index creation. The system observes actual query patterns against JSONB fields, automatically creates indexes when usage thresholds are crossed, and automatically drops indexes that fall into disuse. All index creation is performed with `CREATE INDEX CONCURRENTLY` to avoid blocking writes, and idempotency is guaranteed through a combination of a `UNIQUE` constraint on an `index_registry` table, a `NOT EXISTS` pre-check, and atomic claiming via `UPDATE ... RETURNING`. The architecture is self-governing through automatic policies, auditable through a persistent registry, and safe against race conditions through advisory locks and atomic state transitions.

---

## 2. Problem Statement

### 2.1 The Indexing Dilemma in Dynamic Workflow Systems

In a hybrid system where users define workflows with arbitrary fields stored in JSONB columns, the question of which fields to index cannot be answered at design time. A user defining a "Building Permit" workflow might create a `height` integer field, a `completion_date` date field, and a `contractor_name` text field. Whether any of these fields will actually be queried — and with what frequency — is unknowable until the system is in production.

**Indexing every field is wasteful**: Each index adds disk space (40–60 bytes per row), write overhead (every INSERT and UPDATE must update all indexes), and vacuum cost. With 50 workflows each containing 10 fields, naive indexing would create 500 indexes on the `transactions` table. At 1 million rows, that is approximately 25 GB of index data, and every write would need to update all 500 indexes.

**Indexing no fields is dangerous**: Without indexes, queries that filter on frequently accessed JSONB fields degrade to sequential scans. A query filtering on `custom_values->>'height'` against 1 million rows takes approximately 85 ms in memory and 3+ seconds on disk without an index.

### 2.2 The Constraint: No Human Administrator

In many government and enterprise deployments, the system operates without a dedicated DBA. Index creation cannot wait for human approval because there is no human to approve. The system must therefore **govern itself** through automatic policies that replace human judgment with measurable thresholds and rules.

### 2.3 Why Dynamic DDL is Dangerous Without Guardrails

Before presenting the solution, it is essential to understand the risks that automatic index creation introduces:

**`CREATE INDEX` locks the table**: A normal `CREATE INDEX` takes a `SHARE` lock on the table, blocking all writes for the duration of the build. On a table with millions of rows, this can be minutes to hours of downtime. The fix is `CREATE INDEX CONCURRENTLY`, which allows writes to continue. However, `CONCURRENTLY` **cannot run inside a transaction block or a PL/pgSQL function** — it must be executed as a top-level statement from a client connection.

**Index count explosion**: Each index adds disk space and write overhead. PostgreSQL's planner also degrades when there are too many indexes, as it must evaluate more candidates for every query. Best practice suggests limiting indexes on a table to approximately 6, with a maximum of 10–25 in automated tuning systems.

**SQL injection via field keys**: If the `field_key` comes from user input and is not rigorously sanitized, it opens a direct SQL injection vector through dynamic `EXECUTE format(...)`.

**Orphaned indexes**: If a workflow is deleted or a field is removed, the index remains. Without cleanup, indexes accumulate indefinitely, consuming resources for no benefit.

**No statistics after creation**: A newly created index has no statistics until `ANALYZE` runs. The planner may not use it immediately, or may use it poorly.

**Government governance problem**: In government work, schema changes are typically audited and controlled. Automatically creating database objects based on user actions means no change control board review, no documentation of why the index exists, and no ability to roll back cleanly.

---

## 3. Architecture Overview

### 3.1 High-Level Data Flow

The architecture decouples **decision** (when to index) from **execution** (creating the index). This separation is mandatory because `CREATE INDEX CONCURRENTLY` cannot run inside a transaction, and because atomic decision-making prevents duplicate work.

```
┌─────────────────────────────────────────────────────────────────────┐
│  APPLICATION LAYER                                                  │
│  User executes a query filtering on custom_values->>'height'        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  QUERY LOGGING                                                      │
│  Append row to field_query_log (workflow_id, field_key, field_type) │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼  (scheduled, e.g. every hour)
┌─────────────────────────────────────────────────────────────────────┐
│  CANDIDATE SELECTION (inside a transaction)                         │
│  SELECT fields with >= 50 hits in 24h AND NOT already in registry  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ATOMIC REGISTRATION (inside the same transaction)                  │
│  INSERT INTO index_registry ... ON CONFLICT DO NOTHING              │
│  RETURNING id, index_name, ...                                      │
│  ← UNIQUE (workflow_id, field_key) prevents duplicates             │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼  (outside any transaction)
┌─────────────────────────────────────────────────────────────────────┐
│  INDEX EXECUTION (background worker)                                │
│  CREATE INDEX CONCURRENTLY idx_txn_wf123_height ...                 │
│  ANALYZE transactions                                               │
│  UPDATE index_registry SET status = 'active'                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Core Components

| Component | Purpose |
| :--- | :--- |
| `field_query_log` | Records every query that filters on a JSONB field, with workflow ID, field key, field type, and timestamp. |
| `index_registry` | The central state machine. Tracks every index from `pending` to `active` to `dropped`. The `UNIQUE` constraint is the ultimate safeguard against duplicates. |
| Candidate selector | A SQL query (run inside a transaction) that identifies fields crossing the usage threshold and not yet registered. |
| Background worker | An external process (Python, Node, Go, or a shell script) that executes `CREATE INDEX CONCURRENTLY`, `ANALYZE`, and registry updates outside any transaction. |
| Cleanup job | A scheduled task that queries `pg_stat_user_indexes` for indexes with `idx_scan = 0` and drops them after a configurable grace period. |

---

## 4. Detailed Implementation

### 4.1 The Query Log Table

```sql
CREATE TABLE field_query_log (
    id bigserial PRIMARY KEY,
    workflow_definition_id int NOT NULL REFERENCES workflow_definition(id),
    field_key text NOT NULL,
    field_type text NOT NULL,  -- 'integer', 'text', 'date', 'numeric'
    queried_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_fql_lookup
ON field_query_log (workflow_definition_id, field_key, queried_at);
```

The application writes to this table **outside** the user's query transaction (fire-and-forget) to avoid adding latency. The `queried_at` index supports the 24-hour window queries used by the candidate selector.

### 4.2 The Index Registry Table

```sql
CREATE TABLE index_registry (
    id serial PRIMARY KEY,
    workflow_definition_id int NOT NULL REFERENCES workflow_definition(id),
    field_key text NOT NULL,
    field_type text NOT NULL,
    index_name text NOT NULL UNIQUE,
    status text NOT NULL DEFAULT 'pending',
        -- pending | in_progress | active | failed | dropped
    created_at timestamptz DEFAULT now(),
    activated_at timestamptz,
    last_indexed_at timestamptz,
    last_used_at timestamptz,
    drop_after timestamptz,
    error_message text,
    UNIQUE (workflow_definition_id, field_key)
);

CREATE INDEX idx_ir_status ON index_registry (status);
CREATE INDEX idx_ir_last_used ON index_registry (last_used_at);
```

The **`UNIQUE (workflow_definition_id, field_key)`** constraint is the single most important safeguard. It makes the entire process idempotent at the database level. Even if the worker runs 1,000 times, or 10 workers run simultaneously, the same field can never be registered twice.

### 4.3 Candidate Selection Query

This query runs **inside a transaction** and returns fields that are ready for indexing:

```sql
WITH candidates AS (
    SELECT
        fql.workflow_definition_id,
        fql.field_key,
        fql.field_type,
        count(*) AS hits
    FROM field_query_log fql
    WHERE fql.queried_at > now() - interval '24 hours'
      AND NOT EXISTS (
          SELECT 1 FROM index_registry ir
          WHERE ir.workflow_definition_id = fql.workflow_definition_id
            AND ir.field_key = fql.field_key
            AND ir.status IN ('active', 'in_progress')
      )
    GROUP BY 1, 2, 3
    HAVING count(*) >= 50
)
INSERT INTO index_registry (
    workflow_definition_id, field_key, field_type,
    index_name, status, created_at
)
SELECT
    workflow_definition_id,
    field_key,
    field_type,
    format('idx_txn_wf%s_%s', workflow_definition_id, field_key),
    'pending',
    now()
FROM candidates
ON CONFLICT (workflow_definition_id, field_key) DO NOTHING
RETURNING id, index_name, workflow_definition_id, field_key, field_type;
```

The `NOT EXISTS` subquery prevents re-processing fields that already have an active or in-progress index. The `ON CONFLICT DO NOTHING` handles the race condition where another worker registered the same field between the `NOT EXISTS` check and the `INSERT`. The `UNIQUE` constraint ensures the conflict is detected.

### 4.4 The Background Worker

The worker processes each row returned by the candidate selection query. It runs **outside any transaction** and must use `CREATE INDEX CONCURRENTLY`.

**Critical limitation**: `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block or a PL/pgSQL function. The worker must execute it as a top-level statement from a client connection. `pg_cron` runs inside a transaction, so it cannot directly execute `CONCURRENTLY` — an external worker or `pg_background` is required.

```python
# Pseudocode for the background worker
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT

conn = psycopg2.connect(dsn)
conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)

# Step 1: Select candidates (inside a transaction)
with conn.cursor() as cur:
    cur.execute("""
        WITH candidates AS (...)
        INSERT INTO index_registry (...)
        SELECT ... ON CONFLICT DO NOTHING
        RETURNING id, index_name, workflow_definition_id, field_key, field_type;
    """)
    rows = cur.fetchall()

# Step 2: For each candidate, create the index outside any transaction
for (reg_id, idx_name, wf_id, field_key, field_type) in rows:
    # Sanitize field_key (mandatory)
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]{0,62}

**Key design decisions in the worker:**

| Decision | Rationale |
| :--- | :--- |
| `AUTOCOMMIT` isolation level | Required because `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. |
| `IF NOT EXISTS` | Prevents errors if the index already exists. **But note**: `IF NOT EXISTS` matches on name only, not definition. If you change the index definition but keep the name, PostgreSQL will silently skip it. Version your index names if definitions change. |
| `ANALYZE` after creation | Gives the planner statistics immediately. Without this, the planner may not use the new index. |
| Registry update on failure | Logs the error message so the failure can be diagnosed without re-running. |
| Partial index predicate (`WHERE workflow_definition_id = ... AND custom_values ? ...`) | Reduces index size and write cost. Only rows that actually contain the field are indexed. The query planner will only use the partial index if the query's `WHERE` clause logically implies the index's predicate. |

### 4.5 Advisory Locks for Additional Safety

While the `UNIQUE` constraint on `index_registry` prevents duplicate registration, it does not prevent two workers from both selecting the same candidate and both attempting `CREATE INDEX CONCURRENTLY`. If two workers try to create the same index simultaneously, PostgreSQL will serialize them (only one index creation can run per table at a time), but the second will fail with an error. Advisory locks can prevent wasted work:

```sql
-- At the start of processing each candidate:
SELECT pg_advisory_xact_lock(hashtext('idx_txn_wf' || wf_id || '_' || field_key));
```

However, there is a subtle danger. A transaction blocked on `pg_advisory_lock` still holds a snapshot, and `CREATE INDEX CONCURRENTLY` must wait for all transactions with snapshots predating the second scan to terminate. A blocked `SELECT pg_advisory_lock` can therefore **block the concurrent index creation** in another session, creating a deadlock that PostgreSQL cannot detect. The safer approach is to use the `UPDATE ... RETURNING` atomic claim pattern described in Section 4.6, which does not hold a snapshot while waiting.

### 4.6 Atomic Claiming Without Advisory Locks

Instead of advisory locks, use an atomic `UPDATE ... RETURNING` to claim a candidate before processing:

```sql
UPDATE index_registry
SET status = 'in_progress', claimed_at = now()
WHERE id = <reg_id>
  AND status = 'pending'
RETURNING id;
```

If the `UPDATE` returns 0 rows, another worker already claimed it. This is atomic, does not hold a snapshot while waiting, and is immune to the `CREATE INDEX CONCURRENTLY` snapshot-wait problem.

---

## 5. Guardrails: Automatic Policies Without an Admin

Without a human administrator, the system must enforce its own limits. The following policies should be hard-coded into the worker and the candidate selection query.

### 5.1 Index Count Cap

```sql
-- Before creating any index, check the total count
SELECT count(*) FROM pg_indexes WHERE tablename = 'transactions';
```

If the count is at or above the cap (e.g., 30), the worker should **drop the least-used index** before creating a new one, rather than simply refusing to create. The least-used index is identified by querying `pg_stat_user_indexes` for the lowest `idx_scan` value.

### 5.2 Minimum Table Size

```sql
SELECT reltuples FROM pg_class WHERE relname = 'transactions';
```

If the table has fewer than 10,000 rows, a sequential scan is likely faster than an index. Do not create indexes on small tables. The `reltuples` value is an estimate maintained by `ANALYZE` and autovacuum, so it is approximate but sufficient for this purpose.

### 5.3 Minimum Query Threshold

The candidate selection query uses `HAVING count(*) >= 50` over a 24-hour window. This threshold is configurable. A higher threshold (e.g., 100) reduces the number of indexes created; a lower threshold (e.g., 20) creates indexes more aggressively.

### 5.4 Automatic Cleanup of Unused Indexes

Run a monthly cleanup job that queries `pg_stat_user_indexes` for indexes with `idx_scan = 0`:

```sql
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE relname = 'transactions'
  AND idx_scan = 0
  AND indexrelname NOT LIKE 'transactions_pkey%'
  AND indexrelname NOT IN (
      SELECT index_name FROM protected_indexes
  );
```

The `pg_stat_user_indexes` view provides the `idx_scan` column, which shows how many times each index has been used since the last statistics reset. An index with `idx_scan = 0` over 90 days is a candidate for dropping.

**Important caveat**: `idx_scan` is reset by `pg_stat_reset()` and on server restart. Do not drop an index immediately after a restart; wait at least 30 days to allow usage statistics to accumulate.

### 5.5 Log Cleanup

The `field_query_log` table grows indefinitely. Either delete old entries after processing:

```sql
DELETE FROM field_query_log WHERE queried_at < now() - interval '7 days';
```

Or use the `last_indexed_at` column in `index_registry` to count only queries since the last index creation:

```sql
WHERE fql.queried_at > COALESCE(ir.last_indexed_at, '-infinity'::timestamptz)
```

The second approach is better because it preserves the full audit trail of query activity.

---

## 6. Index Type Selection for JSONB

The worker must choose the correct index type based on the field's data type. The decision is driven by the query operator that will be used.

### 6.1 B-Tree Expression Index (for Known Fields)

When a field is filtered by **equality or range** (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`), a B-tree expression index is the correct choice. The expression extracts the value from JSONB and casts it to the appropriate type:

```sql
-- Integer field
CREATE INDEX idx_txn_wf123_height
ON transactions (((custom_values->>'height')::int))
WHERE workflow_definition_id = 123;

-- Text field
CREATE INDEX idx_txn_wf123_contractor
ON transactions ((custom_values->>'contractor_name'))
WHERE workflow_definition_id = 123;

-- Date field
CREATE INDEX idx_txn_wf123_completion
ON transactions (((custom_values->>'completion_date')::date))
WHERE workflow_definition_id = 123;
```

The cast must be **inside** the parentheses that define the index column. The `WHERE` clause makes this a partial index, which reduces size and write cost.

### 6.2 GIN Index (for Containment and Existence)

When a field is queried with **containment** (`@>`) or **key existence** (`?`), a GIN index is the correct choice:

```sql
CREATE INDEX idx_transactions_custom_gin
ON transactions USING GIN (custom_values);
```

This index supports:
- Containment: `custom_values @> '{"height": 25}'`
- Key existence: `custom_values ? 'height'`
- JSONPath: `custom_values @? '$.height ? (@ > 10)'`

The `jsonb_path_ops` operator class produces a smaller index that is faster for containment-only workloads. The default `jsonb_ops` supports a broader set of operators.

### 6.3 When to Use Which

| Query Pattern | Index Type | Example |
| :--- | :--- | :--- |
| `WHERE custom_values->>'field' = 'value'` | B-tree expression | `CREATE INDEX ... ((custom_values->>'field'))` |
| `WHERE (custom_values->>'field')::int > 10` | B-tree expression with cast | `CREATE INDEX ... (((custom_values->>'field')::int))` |
| `WHERE custom_values @> '{"field": "value"}'` | GIN | `CREATE INDEX ... USING GIN (custom_values)` |
| `WHERE custom_values ? 'field'` | GIN | `CREATE INDEX ... USING GIN (custom_values)` |

A broad GIN index is often a poor match for scalar range queries. A B-tree expression index is typically the better choice when filtering or sorting on a known scalar value.

---

## 7. Write Amplification and TOAST Considerations

JSONB columns larger than approximately 2 KB are stored in TOAST (The Oversized-Attribute Storage Technique). Updating even a small part of a large JSONB value requires that the **entire thing is read and written**, causing significant data churn. This has two important implications for the auto-indexing architecture:

**1. Expression indexes prevent HOT updates**: PostgreSQL's Heap-Only Tuples (HOT) optimization allows updates to avoid updating indexes when the indexed columns are unchanged. However, expression indexes on JSONB prevent HOT updates, because the expression must be re-evaluated to determine whether the indexed value changed. For JSONB-heavy applications, this can cause significant bloat.

**2. Hot keys should be promoted to typed columns**: If a JSONB field is frequently updated or filtered, extracting it into a dedicated relational column is often better than indexing it in place. The SitePoint JSONB performance guide explicitly recommends promoting frequently updated or filtered JSONB keys to typed columns when access patterns stabilize.

The auto-indexing system should therefore have a **promotion policy**: if a field crosses both the index usage threshold and a write-frequency threshold, the worker should recommend (or automatically perform) promotion to a typed column rather than creating an expression index. This is a more invasive change and may require application-layer coordination, but it avoids the HOT-update penalty in the long run.

---

## 8. Comparison with Alternatives

| Approach | When to Use | Advantages | Disadvantages |
| :--- | :--- | :--- | :--- |
| **Usage-driven auto-indexing** (this report) | No admin; unpredictable query patterns | Self-governing; only indexes what is used; automatic cleanup | Complex; requires background worker; DDL risk |
| **Predefined index slots** | Known set of searchable fields; static schema | Zero DDL; predictable; auditable | Bounded to fixed number of slots; requires app mapping |
| **Generated columns** | Top 5–10 hot fields per workflow | Cleaner than expression indexes; B-tree compatible | Requires `ALTER TABLE`; not dynamic |
| **GIN index on entire JSONB** | Containment and existence queries | One index covers all keys; no per-field DDL | Cannot handle range queries; large index; write overhead |
| **Controlled EAV search table** | Cross-workflow search on dynamic fields | Typed value columns; proper indexes; cross-workflow | Reintroduces some EAV complexity; application must write to both places |

The usage-driven approach is the most flexible but also the most complex. For government systems with **stable, predictable workflows**, predefined index slots are safer and more auditable. For systems with **many evolving workflows**, usage-driven auto-indexing is the only approach that scales without human intervention.

---

## 9. Monitoring and Observability

### 9.1 Index Usage

```sql
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname = 'transactions'
ORDER BY idx_scan DESC;
```

This query shows how often each index has been used. Indexes with `idx_scan = 0` are candidates for dropping. Indexes with high `idx_scan` but low `idx_tup_read` may be poorly selective.

### 9.2 Index Size

```sql
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE relname = 'transactions'
ORDER BY pg_relation_size(indexrelid) DESC;
```

If the total index size exceeds the table size, the index set is likely bloated. AWS recommends controlling the number of indexes and dropping unused ones to reduce autovacuum overhead.

### 9.3 Registry Status

```sql
SELECT status, count(*)
FROM index_registry
GROUP BY status;
```

This shows how many indexes are in each state. A high number of `failed` indexes indicates a problem with the worker or with field key sanitization.

### 9.4 Query Log Volume

```sql
SELECT date_trunc('hour', queried_at) AS hour, count(*)
FROM field_query_log
WHERE queried_at > now() - interval '7 days'
GROUP BY 1 ORDER BY 1;
```

This shows query activity over time. Spikes may indicate a new workflow that needs indexing.

---

## 10. Failure Modes and Recovery

### 10.1 Invalid Index After Failed Creation

If `CREATE INDEX CONCURRENTLY` fails midway (e.g., due to a uniqueness violation or a deadlock), the index is left in an **invalid** state. It remains in the catalog but is ignored by the planner. The worker must detect this and drop it before retrying:

```sql
SELECT indexrelname, indisvalid
FROM pg_index i
JOIN pg_class c ON c.oid = i.indexrelid
WHERE i.indrelid = 'transactions'::regclass
  AND NOT i.indisvalid;
```

After a failed deploy, check `pg_index.indisvalid` and `DROP INDEX` before retrying.

### 10.2 Worker Crash

If the worker crashes between `UPDATE index_registry SET status = 'in_progress'` and the actual `CREATE INDEX`, the registry row remains in `in_progress` forever. A recovery job should run periodically:

```sql
UPDATE index_registry
SET status = 'pending'
WHERE status = 'in_progress'
  AND claimed_at < now() - interval '1 hour';
```

### 10.3 Race Condition with `ON CONFLICT DO NOTHING`

If two workers both pass the `NOT EXISTS` check and both attempt `INSERT ... ON CONFLICT DO NOTHING`, one will succeed and the other will silently do nothing. The worker that gets the `RETURNING` row will process it; the other will see no row and move on. This is the correct behavior.

---

## 11. Conclusion

Usage-driven auto-indexing is a viable architecture for PostgreSQL systems that need dynamic indexing without a human administrator. The key design principles are:

1. **Decouple decision from execution**: The candidate selection query runs inside a transaction; `CREATE INDEX CONCURRENTLY` runs outside any transaction in a background worker.
2. **Make the process idempotent**: The `UNIQUE (workflow_definition_id, field_key)` constraint on `index_registry` is the ultimate safeguard against duplicates.
3. **Use atomic claiming**: `UPDATE ... RETURNING` prevents multiple workers from processing the same candidate without the snapshot-holding problems of advisory locks.
4. **Enforce hard guardrails**: Index count caps, minimum table size, and usage thresholds prevent index explosion and wasted work.
5. **Clean up automatically**: Drop indexes with `idx_scan = 0` after a grace period.
6. **Choose the right index type**: B-tree expression indexes for scalar equality and range queries; GIN for containment and existence.
7. **Monitor and recover**: Detect invalid indexes, recover stuck registry rows, and log every action for auditability.

For government work, the system should be designed with **auditability** in mind: every index creation, drop, and failure should be logged in `index_registry`, and the registry should be queryable by auditors. The absence of a human administrator does not mean the absence of governance — it means the governance is encoded in policies rather than in approvals.

---

## Appendix A: Complete `index_registry` Schema

```sql
CREATE TABLE index_registry (
    id serial PRIMARY KEY,
    workflow_definition_id int NOT NULL REFERENCES workflow_definition(id),
    field_key text NOT NULL,
    field_type text NOT NULL,
    index_name text NOT NULL UNIQUE,
    status text NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'in_progress', 'active', 'failed', 'dropped')),
    created_at timestamptz DEFAULT now(),
    claimed_at timestamptz,
    activated_at timestamptz,
    last_indexed_at timestamptz,
    last_used_at timestamptz,
    drop_after timestamptz,
    error_message text,
    UNIQUE (workflow_definition_id, field_key)
);

CREATE INDEX idx_ir_status ON index_registry (status);
CREATE INDEX idx_ir_last_used ON index_registry (last_used_at);
CREATE INDEX idx_ir_workflow ON index_registry (workflow_definition_id);
```

## Appendix B: Field Key Sanitization Regex

```sql
-- Only allow alphanumeric and underscore, starting with a letter or underscore
-- Maximum 63 characters (PostgreSQL identifier limit)
IF p_field_key !~ '^[a-zA-Z_][a-zA-Z0-9_]{0,62}

This regex must be applied **before** any dynamic SQL is constructed. Without it, a field key like `height); DROP TABLE transactions; --` would execute arbitrary SQL.

## Appendix C: Index Type Decision Tree

```
Is the field queried with @> or ? operators?
├── Yes → GIN index on custom_values (or jsonb_path_ops for containment-only)
└── No (scalar equality or range)
    ├── Is the field type integer/numeric?
    │   └── B-tree expression index: ((custom_values->>'key')::numeric)
    ├── Is the field type date?
    │   └── B-tree expression index: ((custom_values->>'key')::date)
    └── Is the field type text?
        └── B-tree expression index: (custom_values->>'key')
```, field_key):
        update_registry(reg_id, 'failed', 'Invalid field key')
        continue

    # Build the index definition based on type
    if field_type == 'integer':
        expr = f"((custom_values->>'{field_key}')::int)"
    elif field_type == 'numeric':
        expr = f"((custom_values->>'{field_key}')::numeric)"
    elif field_type == 'date':
        expr = f"((custom_values->>'{field_key}')::date)"
    else:
        expr = f"(custom_values->>'{field_key}')"

    sql = f"""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS {idx_name}
        ON transactions {expr}
        WHERE workflow_definition_id = {wf_id}
          AND custom_values ? '{field_key}';
    """
    try:
        with conn.cursor() as cur:
            cur.execute(sql)
        with conn.cursor() as cur:
            cur.execute("ANALYZE transactions;")
        update_registry(reg_id, 'active', None)
    except Exception as e:
        update_registry(reg_id, 'failed', str(e))
```

**Key design decisions in the worker:**

| Decision | Rationale |
| :--- | :--- |
| `AUTOCOMMIT` isolation level | Required because `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. |
| `IF NOT EXISTS` | Prevents errors if the index already exists. **But note**: `IF NOT EXISTS` matches on name only, not definition. If you change the index definition but keep the name, PostgreSQL will silently skip it. Version your index names if definitions change. |
| `ANALYZE` after creation | Gives the planner statistics immediately. Without this, the planner may not use the new index. |
| Registry update on failure | Logs the error message so the failure can be diagnosed without re-running. |
| Partial index predicate (`WHERE workflow_definition_id = ... AND custom_values ? ...`) | Reduces index size and write cost. Only rows that actually contain the field are indexed. The query planner will only use the partial index if the query's `WHERE` clause logically implies the index's predicate. |

### 4.5 Advisory Locks for Additional Safety

While the `UNIQUE` constraint on `index_registry` prevents duplicate registration, it does not prevent two workers from both selecting the same candidate and both attempting `CREATE INDEX CONCURRENTLY`. If two workers try to create the same index simultaneously, PostgreSQL will serialize them (only one index creation can run per table at a time), but the second will fail with an error. Advisory locks can prevent wasted work:

{{CODE_BLOCK_5}}

However, there is a subtle danger. A transaction blocked on `pg_advisory_lock` still holds a snapshot, and `CREATE INDEX CONCURRENTLY` must wait for all transactions with snapshots predating the second scan to terminate. A blocked `SELECT pg_advisory_lock` can therefore **block the concurrent index creation** in another session, creating a deadlock that PostgreSQL cannot detect. The safer approach is to use the `UPDATE ... RETURNING` atomic claim pattern described in Section 4.6, which does not hold a snapshot while waiting.

### 4.6 Atomic Claiming Without Advisory Locks

Instead of advisory locks, use an atomic `UPDATE ... RETURNING` to claim a candidate before processing:

{{CODE_BLOCK_6}}

If the `UPDATE` returns 0 rows, another worker already claimed it. This is atomic, does not hold a snapshot while waiting, and is immune to the `CREATE INDEX CONCURRENTLY` snapshot-wait problem.

---

## 5. Guardrails: Automatic Policies Without an Admin

Without a human administrator, the system must enforce its own limits. The following policies should be hard-coded into the worker and the candidate selection query.

### 5.1 Index Count Cap

{{CODE_BLOCK_7}}

If the count is at or above the cap (e.g., 30), the worker should **drop the least-used index** before creating a new one, rather than simply refusing to create. The least-used index is identified by querying `pg_stat_user_indexes` for the lowest `idx_scan` value.

### 5.2 Minimum Table Size

{{CODE_BLOCK_8}}

If the table has fewer than 10,000 rows, a sequential scan is likely faster than an index. Do not create indexes on small tables. The `reltuples` value is an estimate maintained by `ANALYZE` and autovacuum, so it is approximate but sufficient for this purpose.

### 5.3 Minimum Query Threshold

The candidate selection query uses `HAVING count(*) >= 50` over a 24-hour window. This threshold is configurable. A higher threshold (e.g., 100) reduces the number of indexes created; a lower threshold (e.g., 20) creates indexes more aggressively.

### 5.4 Automatic Cleanup of Unused Indexes

Run a monthly cleanup job that queries `pg_stat_user_indexes` for indexes with `idx_scan = 0`:

{{CODE_BLOCK_9}}

The `pg_stat_user_indexes` view provides the `idx_scan` column, which shows how many times each index has been used since the last statistics reset. An index with `idx_scan = 0` over 90 days is a candidate for dropping.

**Important caveat**: `idx_scan` is reset by `pg_stat_reset()` and on server restart. Do not drop an index immediately after a restart; wait at least 30 days to allow usage statistics to accumulate.

### 5.5 Log Cleanup

The `field_query_log` table grows indefinitely. Either delete old entries after processing:

{{CODE_BLOCK_10}}

Or use the `last_indexed_at` column in `index_registry` to count only queries since the last index creation:

{{CODE_BLOCK_11}}

The second approach is better because it preserves the full audit trail of query activity.

---

## 6. Index Type Selection for JSONB

The worker must choose the correct index type based on the field's data type. The decision is driven by the query operator that will be used.

### 6.1 B-Tree Expression Index (for Known Fields)

When a field is filtered by **equality or range** (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`), a B-tree expression index is the correct choice. The expression extracts the value from JSONB and casts it to the appropriate type:

{{CODE_BLOCK_12}}

The cast must be **inside** the parentheses that define the index column. The `WHERE` clause makes this a partial index, which reduces size and write cost.

### 6.2 GIN Index (for Containment and Existence)

When a field is queried with **containment** (`@>`) or **key existence** (`?`), a GIN index is the correct choice:

{{CODE_BLOCK_13}}

This index supports:
- Containment: `custom_values @> '{"height": 25}'`
- Key existence: `custom_values ? 'height'`
- JSONPath: `custom_values @? '$.height ? (@ > 10)'`

The `jsonb_path_ops` operator class produces a smaller index that is faster for containment-only workloads. The default `jsonb_ops` supports a broader set of operators.

### 6.3 When to Use Which

| Query Pattern | Index Type | Example |
| :--- | :--- | :--- |
| `WHERE custom_values->>'field' = 'value'` | B-tree expression | `CREATE INDEX ... ((custom_values->>'field'))` |
| `WHERE (custom_values->>'field')::int > 10` | B-tree expression with cast | `CREATE INDEX ... (((custom_values->>'field')::int))` |
| `WHERE custom_values @> '{"field": "value"}'` | GIN | `CREATE INDEX ... USING GIN (custom_values)` |
| `WHERE custom_values ? 'field'` | GIN | `CREATE INDEX ... USING GIN (custom_values)` |

A broad GIN index is often a poor match for scalar range queries. A B-tree expression index is typically the better choice when filtering or sorting on a known scalar value.

---

## 7. Write Amplification and TOAST Considerations

JSONB columns larger than approximately 2 KB are stored in TOAST (The Oversized-Attribute Storage Technique). Updating even a small part of a large JSONB value requires that the **entire thing is read and written**, causing significant data churn. This has two important implications for the auto-indexing architecture:

**1. Expression indexes prevent HOT updates**: PostgreSQL's Heap-Only Tuples (HOT) optimization allows updates to avoid updating indexes when the indexed columns are unchanged. However, expression indexes on JSONB prevent HOT updates, because the expression must be re-evaluated to determine whether the indexed value changed. For JSONB-heavy applications, this can cause significant bloat.

**2. Hot keys should be promoted to typed columns**: If a JSONB field is frequently updated or filtered, extracting it into a dedicated relational column is often better than indexing it in place. The SitePoint JSONB performance guide explicitly recommends promoting frequently updated or filtered JSONB keys to typed columns when access patterns stabilize.

The auto-indexing system should therefore have a **promotion policy**: if a field crosses both the index usage threshold and a write-frequency threshold, the worker should recommend (or automatically perform) promotion to a typed column rather than creating an expression index. This is a more invasive change and may require application-layer coordination, but it avoids the HOT-update penalty in the long run.

---

## 8. Comparison with Alternatives

| Approach | When to Use | Advantages | Disadvantages |
| :--- | :--- | :--- | :--- |
| **Usage-driven auto-indexing** (this report) | No admin; unpredictable query patterns | Self-governing; only indexes what is used; automatic cleanup | Complex; requires background worker; DDL risk |
| **Predefined index slots** | Known set of searchable fields; static schema | Zero DDL; predictable; auditable | Bounded to fixed number of slots; requires app mapping |
| **Generated columns** | Top 5–10 hot fields per workflow | Cleaner than expression indexes; B-tree compatible | Requires `ALTER TABLE`; not dynamic |
| **GIN index on entire JSONB** | Containment and existence queries | One index covers all keys; no per-field DDL | Cannot handle range queries; large index; write overhead |
| **Controlled EAV search table** | Cross-workflow search on dynamic fields | Typed value columns; proper indexes; cross-workflow | Reintroduces some EAV complexity; application must write to both places |

The usage-driven approach is the most flexible but also the most complex. For government systems with **stable, predictable workflows**, predefined index slots are safer and more auditable. For systems with **many evolving workflows**, usage-driven auto-indexing is the only approach that scales without human intervention.

---

## 9. Monitoring and Observability

### 9.1 Index Usage

{{CODE_BLOCK_14}}

This query shows how often each index has been used. Indexes with `idx_scan = 0` are candidates for dropping. Indexes with high `idx_scan` but low `idx_tup_read` may be poorly selective.

### 9.2 Index Size

{{CODE_BLOCK_15}}

If the total index size exceeds the table size, the index set is likely bloated. AWS recommends controlling the number of indexes and dropping unused ones to reduce autovacuum overhead.

### 9.3 Registry Status

{{CODE_BLOCK_16}}

This shows how many indexes are in each state. A high number of `failed` indexes indicates a problem with the worker or with field key sanitization.

### 9.4 Query Log Volume

{{CODE_BLOCK_17}}

This shows query activity over time. Spikes may indicate a new workflow that needs indexing.

---

## 10. Failure Modes and Recovery

### 10.1 Invalid Index After Failed Creation

If `CREATE INDEX CONCURRENTLY` fails midway (e.g., due to a uniqueness violation or a deadlock), the index is left in an **invalid** state. It remains in the catalog but is ignored by the planner. The worker must detect this and drop it before retrying:

{{CODE_BLOCK_18}}

After a failed deploy, check `pg_index.indisvalid` and `DROP INDEX` before retrying.

### 10.2 Worker Crash

If the worker crashes between `UPDATE index_registry SET status = 'in_progress'` and the actual `CREATE INDEX`, the registry row remains in `in_progress` forever. A recovery job should run periodically:

{{CODE_BLOCK_19}}

### 10.3 Race Condition with `ON CONFLICT DO NOTHING`

If two workers both pass the `NOT EXISTS` check and both attempt `INSERT ... ON CONFLICT DO NOTHING`, one will succeed and the other will silently do nothing. The worker that gets the `RETURNING` row will process it; the other will see no row and move on. This is the correct behavior.

---

## 11. Conclusion

Usage-driven auto-indexing is a viable architecture for PostgreSQL systems that need dynamic indexing without a human administrator. The key design principles are:

1. **Decouple decision from execution**: The candidate selection query runs inside a transaction; `CREATE INDEX CONCURRENTLY` runs outside any transaction in a background worker.
2. **Make the process idempotent**: The `UNIQUE (workflow_definition_id, field_key)` constraint on `index_registry` is the ultimate safeguard against duplicates.
3. **Use atomic claiming**: `UPDATE ... RETURNING` prevents multiple workers from processing the same candidate without the snapshot-holding problems of advisory locks.
4. **Enforce hard guardrails**: Index count caps, minimum table size, and usage thresholds prevent index explosion and wasted work.
5. **Clean up automatically**: Drop indexes with `idx_scan = 0` after a grace period.
6. **Choose the right index type**: B-tree expression indexes for scalar equality and range queries; GIN for containment and existence.
7. **Monitor and recover**: Detect invalid indexes, recover stuck registry rows, and log every action for auditability.

For government work, the system should be designed with **auditability** in mind: every index creation, drop, and failure should be logged in `index_registry`, and the registry should be queryable by auditors. The absence of a human administrator does not mean the absence of governance — it means the governance is encoded in policies rather than in approvals.

---

## Appendix A: Complete `index_registry` Schema

{{CODE_BLOCK_20}}

## Appendix B: Field Key Sanitization Regex

{{CODE_BLOCK_21}}

This regex must be applied **before** any dynamic SQL is constructed. Without it, a field key like `height); DROP TABLE transactions; --` would execute arbitrary SQL.

## Appendix C: Index Type Decision Tree

{{CODE_BLOCK_22}} THEN
    RAISE EXCEPTION 'Invalid field key: %', p_field_key;
END IF;
```

This regex must be applied **before** any dynamic SQL is constructed. Without it, a field key like `height); DROP TABLE transactions; --` would execute arbitrary SQL.

## Appendix C: Index Type Decision Tree

{{CODE_BLOCK_22}}, field_key):
        update_registry(reg_id, 'failed', 'Invalid field key')
        continue

    # Build the index definition based on type
    if field_type == 'integer':
        expr = f"((custom_values->>'{field_key}')::int)"
    elif field_type == 'numeric':
        expr = f"((custom_values->>'{field_key}')::numeric)"
    elif field_type == 'date':
        expr = f"((custom_values->>'{field_key}')::date)"
    else:
        expr = f"(custom_values->>'{field_key}')"

    sql = f"""
        CREATE INDEX CONCURRENTLY IF NOT EXISTS {idx_name}
        ON transactions {expr}
        WHERE workflow_definition_id = {wf_id}
          AND custom_values ? '{field_key}';
    """
    try:
        with conn.cursor() as cur:
            cur.execute(sql)
        with conn.cursor() as cur:
            cur.execute("ANALYZE transactions;")
        update_registry(reg_id, 'active', None)
    except Exception as e:
        update_registry(reg_id, 'failed', str(e))
```

**Key design decisions in the worker:**

| Decision | Rationale |
| :--- | :--- |
| `AUTOCOMMIT` isolation level | Required because `CREATE INDEX CONCURRENTLY` cannot run inside a transaction. |
| `IF NOT EXISTS` | Prevents errors if the index already exists. **But note**: `IF NOT EXISTS` matches on name only, not definition. If you change the index definition but keep the name, PostgreSQL will silently skip it. Version your index names if definitions change. |
| `ANALYZE` after creation | Gives the planner statistics immediately. Without this, the planner may not use the new index. |
| Registry update on failure | Logs the error message so the failure can be diagnosed without re-running. |
| Partial index predicate (`WHERE workflow_definition_id = ... AND custom_values ? ...`) | Reduces index size and write cost. Only rows that actually contain the field are indexed. The query planner will only use the partial index if the query's `WHERE` clause logically implies the index's predicate. |

### 4.5 Advisory Locks for Additional Safety

While the `UNIQUE` constraint on `index_registry` prevents duplicate registration, it does not prevent two workers from both selecting the same candidate and both attempting `CREATE INDEX CONCURRENTLY`. If two workers try to create the same index simultaneously, PostgreSQL will serialize them (only one index creation can run per table at a time), but the second will fail with an error. Advisory locks can prevent wasted work:

{{CODE_BLOCK_5}}

However, there is a subtle danger. A transaction blocked on `pg_advisory_lock` still holds a snapshot, and `CREATE INDEX CONCURRENTLY` must wait for all transactions with snapshots predating the second scan to terminate. A blocked `SELECT pg_advisory_lock` can therefore **block the concurrent index creation** in another session, creating a deadlock that PostgreSQL cannot detect. The safer approach is to use the `UPDATE ... RETURNING` atomic claim pattern described in Section 4.6, which does not hold a snapshot while waiting.

### 4.6 Atomic Claiming Without Advisory Locks

Instead of advisory locks, use an atomic `UPDATE ... RETURNING` to claim a candidate before processing:

{{CODE_BLOCK_6}}

If the `UPDATE` returns 0 rows, another worker already claimed it. This is atomic, does not hold a snapshot while waiting, and is immune to the `CREATE INDEX CONCURRENTLY` snapshot-wait problem.

---

## 5. Guardrails: Automatic Policies Without an Admin

Without a human administrator, the system must enforce its own limits. The following policies should be hard-coded into the worker and the candidate selection query.

### 5.1 Index Count Cap

{{CODE_BLOCK_7}}

If the count is at or above the cap (e.g., 30), the worker should **drop the least-used index** before creating a new one, rather than simply refusing to create. The least-used index is identified by querying `pg_stat_user_indexes` for the lowest `idx_scan` value.

### 5.2 Minimum Table Size

{{CODE_BLOCK_8}}

If the table has fewer than 10,000 rows, a sequential scan is likely faster than an index. Do not create indexes on small tables. The `reltuples` value is an estimate maintained by `ANALYZE` and autovacuum, so it is approximate but sufficient for this purpose.

### 5.3 Minimum Query Threshold

The candidate selection query uses `HAVING count(*) >= 50` over a 24-hour window. This threshold is configurable. A higher threshold (e.g., 100) reduces the number of indexes created; a lower threshold (e.g., 20) creates indexes more aggressively.

### 5.4 Automatic Cleanup of Unused Indexes

Run a monthly cleanup job that queries `pg_stat_user_indexes` for indexes with `idx_scan = 0`:

{{CODE_BLOCK_9}}

The `pg_stat_user_indexes` view provides the `idx_scan` column, which shows how many times each index has been used since the last statistics reset. An index with `idx_scan = 0` over 90 days is a candidate for dropping.

**Important caveat**: `idx_scan` is reset by `pg_stat_reset()` and on server restart. Do not drop an index immediately after a restart; wait at least 30 days to allow usage statistics to accumulate.

### 5.5 Log Cleanup

The `field_query_log` table grows indefinitely. Either delete old entries after processing:

{{CODE_BLOCK_10}}

Or use the `last_indexed_at` column in `index_registry` to count only queries since the last index creation:

{{CODE_BLOCK_11}}

The second approach is better because it preserves the full audit trail of query activity.

---

## 6. Index Type Selection for JSONB

The worker must choose the correct index type based on the field's data type. The decision is driven by the query operator that will be used.

### 6.1 B-Tree Expression Index (for Known Fields)

When a field is filtered by **equality or range** (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`), a B-tree expression index is the correct choice. The expression extracts the value from JSONB and casts it to the appropriate type:

{{CODE_BLOCK_12}}

The cast must be **inside** the parentheses that define the index column. The `WHERE` clause makes this a partial index, which reduces size and write cost.

### 6.2 GIN Index (for Containment and Existence)

When a field is queried with **containment** (`@>`) or **key existence** (`?`), a GIN index is the correct choice:

{{CODE_BLOCK_13}}

This index supports:
- Containment: `custom_values @> '{"height": 25}'`
- Key existence: `custom_values ? 'height'`
- JSONPath: `custom_values @? '$.height ? (@ > 10)'`

The `jsonb_path_ops` operator class produces a smaller index that is faster for containment-only workloads. The default `jsonb_ops` supports a broader set of operators.

### 6.3 When to Use Which

| Query Pattern | Index Type | Example |
| :--- | :--- | :--- |
| `WHERE custom_values->>'field' = 'value'` | B-tree expression | `CREATE INDEX ... ((custom_values->>'field'))` |
| `WHERE (custom_values->>'field')::int > 10` | B-tree expression with cast | `CREATE INDEX ... (((custom_values->>'field')::int))` |
| `WHERE custom_values @> '{"field": "value"}'` | GIN | `CREATE INDEX ... USING GIN (custom_values)` |
| `WHERE custom_values ? 'field'` | GIN | `CREATE INDEX ... USING GIN (custom_values)` |

A broad GIN index is often a poor match for scalar range queries. A B-tree expression index is typically the better choice when filtering or sorting on a known scalar value.

---

## 7. Write Amplification and TOAST Considerations

JSONB columns larger than approximately 2 KB are stored in TOAST (The Oversized-Attribute Storage Technique). Updating even a small part of a large JSONB value requires that the **entire thing is read and written**, causing significant data churn. This has two important implications for the auto-indexing architecture:

**1. Expression indexes prevent HOT updates**: PostgreSQL's Heap-Only Tuples (HOT) optimization allows updates to avoid updating indexes when the indexed columns are unchanged. However, expression indexes on JSONB prevent HOT updates, because the expression must be re-evaluated to determine whether the indexed value changed. For JSONB-heavy applications, this can cause significant bloat.

**2. Hot keys should be promoted to typed columns**: If a JSONB field is frequently updated or filtered, extracting it into a dedicated relational column is often better than indexing it in place. The SitePoint JSONB performance guide explicitly recommends promoting frequently updated or filtered JSONB keys to typed columns when access patterns stabilize.

The auto-indexing system should therefore have a **promotion policy**: if a field crosses both the index usage threshold and a write-frequency threshold, the worker should recommend (or automatically perform) promotion to a typed column rather than creating an expression index. This is a more invasive change and may require application-layer coordination, but it avoids the HOT-update penalty in the long run.

---

## 8. Comparison with Alternatives

| Approach | When to Use | Advantages | Disadvantages |
| :--- | :--- | :--- | :--- |
| **Usage-driven auto-indexing** (this report) | No admin; unpredictable query patterns | Self-governing; only indexes what is used; automatic cleanup | Complex; requires background worker; DDL risk |
| **Predefined index slots** | Known set of searchable fields; static schema | Zero DDL; predictable; auditable | Bounded to fixed number of slots; requires app mapping |
| **Generated columns** | Top 5–10 hot fields per workflow | Cleaner than expression indexes; B-tree compatible | Requires `ALTER TABLE`; not dynamic |
| **GIN index on entire JSONB** | Containment and existence queries | One index covers all keys; no per-field DDL | Cannot handle range queries; large index; write overhead |
| **Controlled EAV search table** | Cross-workflow search on dynamic fields | Typed value columns; proper indexes; cross-workflow | Reintroduces some EAV complexity; application must write to both places |

The usage-driven approach is the most flexible but also the most complex. For government systems with **stable, predictable workflows**, predefined index slots are safer and more auditable. For systems with **many evolving workflows**, usage-driven auto-indexing is the only approach that scales without human intervention.

---

## 9. Monitoring and Observability

### 9.1 Index Usage

{{CODE_BLOCK_14}}

This query shows how often each index has been used. Indexes with `idx_scan = 0` are candidates for dropping. Indexes with high `idx_scan` but low `idx_tup_read` may be poorly selective.

### 9.2 Index Size

{{CODE_BLOCK_15}}

If the total index size exceeds the table size, the index set is likely bloated. AWS recommends controlling the number of indexes and dropping unused ones to reduce autovacuum overhead.

### 9.3 Registry Status

{{CODE_BLOCK_16}}

This shows how many indexes are in each state. A high number of `failed` indexes indicates a problem with the worker or with field key sanitization.

### 9.4 Query Log Volume

{{CODE_BLOCK_17}}

This shows query activity over time. Spikes may indicate a new workflow that needs indexing.

---

## 10. Failure Modes and Recovery

### 10.1 Invalid Index After Failed Creation

If `CREATE INDEX CONCURRENTLY` fails midway (e.g., due to a uniqueness violation or a deadlock), the index is left in an **invalid** state. It remains in the catalog but is ignored by the planner. The worker must detect this and drop it before retrying:

{{CODE_BLOCK_18}}

After a failed deploy, check `pg_index.indisvalid` and `DROP INDEX` before retrying.

### 10.2 Worker Crash

If the worker crashes between `UPDATE index_registry SET status = 'in_progress'` and the actual `CREATE INDEX`, the registry row remains in `in_progress` forever. A recovery job should run periodically:

{{CODE_BLOCK_19}}

### 10.3 Race Condition with `ON CONFLICT DO NOTHING`

If two workers both pass the `NOT EXISTS` check and both attempt `INSERT ... ON CONFLICT DO NOTHING`, one will succeed and the other will silently do nothing. The worker that gets the `RETURNING` row will process it; the other will see no row and move on. This is the correct behavior.

---

## 11. Conclusion

Usage-driven auto-indexing is a viable architecture for PostgreSQL systems that need dynamic indexing without a human administrator. The key design principles are:

1. **Decouple decision from execution**: The candidate selection query runs inside a transaction; `CREATE INDEX CONCURRENTLY` runs outside any transaction in a background worker.
2. **Make the process idempotent**: The `UNIQUE (workflow_definition_id, field_key)` constraint on `index_registry` is the ultimate safeguard against duplicates.
3. **Use atomic claiming**: `UPDATE ... RETURNING` prevents multiple workers from processing the same candidate without the snapshot-holding problems of advisory locks.
4. **Enforce hard guardrails**: Index count caps, minimum table size, and usage thresholds prevent index explosion and wasted work.
5. **Clean up automatically**: Drop indexes with `idx_scan = 0` after a grace period.
6. **Choose the right index type**: B-tree expression indexes for scalar equality and range queries; GIN for containment and existence.
7. **Monitor and recover**: Detect invalid indexes, recover stuck registry rows, and log every action for auditability.

For government work, the system should be designed with **auditability** in mind: every index creation, drop, and failure should be logged in `index_registry`, and the registry should be queryable by auditors. The absence of a human administrator does not mean the absence of governance — it means the governance is encoded in policies rather than in approvals.

---

## Appendix A: Complete `index_registry` Schema

{{CODE_BLOCK_20}}

## Appendix B: Field Key Sanitization Regex

{{CODE_BLOCK_21}}

This regex must be applied **before** any dynamic SQL is constructed. Without it, a field key like `height); DROP TABLE transactions; --` would execute arbitrary SQL.

## Appendix C: Index Type Decision Tree

{{CODE_BLOCK_22}}