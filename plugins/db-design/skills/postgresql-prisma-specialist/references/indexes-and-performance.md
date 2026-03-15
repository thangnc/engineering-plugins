# Indexes & Performance Reference

## Indexing Strategy

### The Golden Rule

Only create indexes for queries you actually run. Every index speeds up reads but slows down writes and consumes storage. Start lean, add indexes when you have evidence (slow queries, EXPLAIN output).

### Index Types in PostgreSQL (via Prisma)

| Index Type | Prisma Syntax | Use Case | Notes |
|-----------|--------------|----------|-------|
| B-tree (default) | `@@index([field])` | Equality, range, sorting | Default. Handles `=`, `<`, `>`, `BETWEEN`, `ORDER BY`. |
| Composite B-tree | `@@index([field1, field2])` | Multi-column filters/sorts | Column order matters! Leftmost prefix rule applies. |
| Unique | `@@unique([field])` or `@unique` | Uniqueness enforcement | Also creates an index. Don't add a separate `@@index`. |
| Hash | `@@index([field], type: Hash)` | Equality-only lookups | Faster than B-tree for `=`, but doesn't support range or sorting. |
| GIN | `@@index([field], type: Gin)` | JSONB, arrays, full-text | Required for JSONB field queries and `tsvector` search. |
| GiST | Needs raw SQL | Geospatial, range types | PostGIS, IP ranges, temporal ranges. |
| BRIN | Needs raw SQL | Large, naturally ordered tables | Tiny index for big tables where data is physically sorted (time-series). |

### Composite Index Column Ordering

The order of columns in a composite index is critical. Follow this rule:

**Equality columns first, then range/sort columns.**

```
-- Query: WHERE tenant_id = ? AND status = ? ORDER BY created_at DESC
-- Good:  @@index([tenantId, status, createdAt(sort: Desc)])
-- Bad:   @@index([createdAt, tenantId, status])
```

A composite index on `[A, B, C]` can serve queries on:
- `A` alone
- `A, B` together
- `A, B, C` together

It CANNOT efficiently serve queries on:
- `B` alone
- `C` alone
- `B, C` together

### Partial Indexes (Raw SQL)

Partial indexes only index rows matching a condition. Essential for soft-delete patterns and status-based queries.

```sql
-- Only index non-deleted rows (most queries filter out deleted)
CREATE INDEX idx_users_email_active
  ON users (email)
  WHERE deleted_at IS NULL;

-- Only index pending applications (hot path)
CREATE INDEX idx_applications_pending
  ON job_applications (created_at DESC)
  WHERE status = 'SUBMITTED';
```

Prisma cannot define partial indexes natively. Add them in migration SQL files and document them in schema comments.

### Expression Indexes (Raw SQL)

Index on computed values:

```sql
-- Case-insensitive email lookup without citext
CREATE INDEX idx_users_email_lower
  ON users (LOWER(email));

-- Index on JSONB field extraction
CREATE INDEX idx_profiles_country
  ON profiles ((metadata->>'country'));

-- Index on date part of timestamp
CREATE INDEX idx_events_date
  ON events (DATE(created_at));
```

## Query Pattern → Index Mapping

Always document which query each index serves. Here are common patterns:

### Pattern: Filter + Sort

```prisma
// Query: Find user's applications, newest first
// prisma.jobApplication.findMany({ where: { userId }, orderBy: { createdAt: 'desc' } })

@@index([userId, createdAt(sort: Desc)])
```

### Pattern: Multi-tenant Lookups

```prisma
// Query: Find tenant's active users by email
// prisma.user.findFirst({ where: { tenantId, email, deletedAt: null } })

@@unique([tenantId, email])
// + partial index WHERE deleted_at IS NULL (raw SQL)
```

### Pattern: Status Dashboard

```prisma
// Query: Count applications by status for a job
// prisma.jobApplication.groupBy({ by: ['status'], where: { jobId }, _count: true })

@@index([jobId, status])
```

### Pattern: Full-text Search

```sql
-- Requires tsvector column + GIN index (raw SQL)
ALTER TABLE jobs ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;

CREATE INDEX idx_jobs_search ON jobs USING GIN (search_vector);
```

### Pattern: JSONB Field Queries

```sql
-- Index specific JSONB paths you query
CREATE INDEX idx_settings_theme
  ON user_settings USING GIN (preferences jsonb_path_ops);

-- Or index a specific extracted key
CREATE INDEX idx_settings_lang
  ON user_settings ((preferences->>'language'));
```

## Performance Anti-Patterns

### 1. Missing Foreign Key Indexes

Prisma does NOT auto-create indexes on foreign key columns. This is the #1 performance problem in Prisma apps.

```prisma
// BAD — no index on userId, full table scan on JOINs
model Post {
  userId String @map("user_id") @db.Uuid
  user   User   @relation(fields: [userId], references: [id])
}

// GOOD — explicit index
model Post {
  userId String @map("user_id") @db.Uuid
  user   User   @relation(fields: [userId], references: [id])

  @@index([userId])
}
```

### 2. N+1 Queries

Schema design can help prevent N+1 problems:

- **Denormalize counts** when you frequently display them (e.g., `commentCount` on Post).
- **Use composite fields** instead of separate lookups when data is always accessed together.
- **Design for `include`** — keep relation depth shallow (max 2–3 levels).

### 3. Over-Indexing

Signs you have too many indexes:
- Write-heavy table with 10+ indexes
- Indexes that are subsets of other indexes (`[A]` when `[A, B]` already exists)
- Indexes on rarely-queried columns
- Low index usage in `pg_stat_user_indexes`

### 4. Unbounded Queries

Always design schemas to support pagination. Add a cursor-friendly index:

```prisma
// Cursor-based pagination: ORDER BY createdAt DESC, id DESC
@@index([createdAt(sort: Desc), id(sort: Desc)])
```

### 5. Large Row Sizes

Keep rows under 8KB (PostgreSQL's page size) when possible:
- Move large text to a separate table
- Store files in object storage, keep only URLs
- Use JSONB sparingly — large JSONB fields bloat rows

## Connection Pooling

### Prisma with PgBouncer

For production, always use a connection pooler. Configure in `schema.prisma`:

```prisma
datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")  // for migrations (bypasses pooler)
}
```

Set `DATABASE_URL` to your PgBouncer URL with `?pgbouncer=true&connection_limit=1` appended.
Set `DIRECT_URL` to your direct PostgreSQL connection for running migrations.

### Connection Limits

- **Serverless (Vercel, Lambda):** Use Prisma Accelerate or PgBouncer in transaction mode. Set connection limit to 1.
- **Long-running server:** Set pool size to `(cpu_cores * 2) + disk_spindles`. For most apps, 10–20 is fine.
- **Multiple replicas:** Use `@prisma/extension-read-replicas` for read/write splitting.

## Monitoring Queries to Add

Run these periodically to find performance issues:

```sql
-- Find missing indexes (sequential scans on large tables)
SELECT relname, seq_scan, idx_scan, n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND n_live_tup > 10000
ORDER BY seq_scan - idx_scan DESC;

-- Find unused indexes (candidates for removal)
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find slow queries (requires pg_stat_statements extension)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```
