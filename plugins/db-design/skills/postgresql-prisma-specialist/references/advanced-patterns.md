# Advanced Patterns Reference

## Multi-Tenancy

### Column-Based Isolation (Recommended for Most Apps)

Add `tenantId` to every tenant-scoped table. Enforce with composite indexes and unique constraints.

```prisma
model Organization {
  id   String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name String @db.VarChar(255)

  users User[]
  jobs  Job[]

  @@map("organizations")
}

model Job {
  id             String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  organizationId String @map("organization_id") @db.Uuid
  title          String @db.VarChar(255)

  organization Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)

  @@index([organizationId])
  @@map("jobs")
}
```

Enforce tenant isolation with Prisma middleware or extension:

```typescript
// Prisma extension for automatic tenant scoping
const prisma = new PrismaClient().$extends({
  query: {
    $allModels: {
      async findMany({ args, query }) {
        args.where = { ...args.where, organizationId: getCurrentTenantId() };
        return query(args);
      },
    },
  },
});
```

### Row-Level Security (Maximum Isolation)

For stricter isolation, use PostgreSQL RLS:

```sql
-- Enable RLS on the table
ALTER TABLE jobs ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see their organization's data
CREATE POLICY tenant_isolation ON jobs
  USING (organization_id = current_setting('app.current_tenant')::uuid);

-- Set tenant context before each request (in your Node.js middleware)
-- await prisma.$executeRaw`SET LOCAL app.current_tenant = ${tenantId}`;
```

RLS trade-offs:
- Stronger: Can't accidentally leak cross-tenant data, even with raw SQL
- Harder: Requires managing session variables, complicates migrations
- Slower: Adds overhead per query (usually negligible, but measure)

## Soft Deletes

### Standard Pattern

```prisma
model User {
  id        String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  email     String    @db.VarChar(320)
  deletedAt DateTime? @map("deleted_at")

  // Unique on email, but only for non-deleted users (needs partial index)
  @@map("users")
}
```

Pair with a partial unique index in raw SQL:

```sql
CREATE UNIQUE INDEX idx_users_email_active
  ON users (email)
  WHERE deleted_at IS NULL;
```

Why not `@@unique([email])` in Prisma? Because that makes the email unique across ALL rows including soft-deleted ones, preventing a user from re-registering with an email that was previously soft-deleted.

### Prisma Middleware for Automatic Filtering

```typescript
// Automatically exclude soft-deleted records
const prisma = new PrismaClient().$extends({
  query: {
    $allModels: {
      async findMany({ args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
      async findFirst({ args, query }) {
        args.where = { ...args.where, deletedAt: null };
        return query(args);
      },
    },
  },
});
```

## Audit Trails

### Separate Audit Table

```prisma
model AuditLog {
  id         String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  tableName  String   @map("table_name") @db.VarChar(100)
  recordId   String   @map("record_id") @db.Uuid
  action     AuditAction
  changes    Json?    @db.JsonB  // { field: { old: ..., new: ... } }
  performedBy String? @map("performed_by") @db.Uuid
  performedAt DateTime @default(now()) @map("performed_at")
  ipAddress  String?  @map("ip_address") @db.VarChar(45)

  @@index([tableName, recordId])
  @@index([performedBy])
  @@index([performedAt])
  @@map("audit_logs")
}

enum AuditAction {
  CREATE
  UPDATE
  DELETE

  @@map("audit_action")
}
```

### PostgreSQL Trigger Approach (More Reliable)

For critical audit requirements, use a database trigger so that even raw SQL changes are captured:

```sql
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_logs (table_name, record_id, action, changes, performed_at)
  VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    CASE TG_OP
      WHEN 'UPDATE' THEN jsonb_build_object(
        'old', to_jsonb(OLD),
        'new', to_jsonb(NEW)
      )
      WHEN 'DELETE' THEN to_jsonb(OLD)
      WHEN 'INSERT' THEN to_jsonb(NEW)
    END,
    NOW()
  );
  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

-- Apply to a table
CREATE TRIGGER audit_users
  AFTER INSERT OR UPDATE OR DELETE ON users
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

## Polymorphic Relations

### Discriminated Union (Recommended)

Instead of a generic "commentable_type" + "commentable_id" pattern, use explicit foreign keys:

```prisma
model Comment {
  id     String  @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  body   String  @db.Text
  
  // Explicit nullable FKs for each target type
  postId    String? @map("post_id") @db.Uuid
  articleId String? @map("article_id") @db.Uuid

  post    Post?    @relation(fields: [postId], references: [id], onDelete: Cascade)
  article Article? @relation(fields: [articleId], references: [id], onDelete: Cascade)

  @@index([postId])
  @@index([articleId])
  @@map("comments")
}
```

Add a check constraint to ensure exactly one FK is set:

```sql
ALTER TABLE comments
  ADD CONSTRAINT check_one_parent
  CHECK (num_nonnulls(post_id, article_id) = 1);
```

This approach gives you referential integrity (foreign keys work), type safety, and queryability — none of which you get with the generic "type + id" pattern.

### When You Have Many Target Types

If you genuinely have 5+ target types, consider a shared base table:

```prisma
model Commentable {
  id       String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  type     CommentableType
  comments Comment[]

  @@map("commentables")
}

model Post {
  id            String      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  commentableId String      @unique @map("commentable_id") @db.Uuid
  commentable   Commentable @relation(fields: [commentableId], references: [id])

  @@map("posts")
}

model Comment {
  id            String      @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  commentableId String      @map("commentable_id") @db.Uuid
  body          String      @db.Text
  commentable   Commentable @relation(fields: [commentableId], references: [id], onDelete: Cascade)

  @@index([commentableId])
  @@map("comments")
}
```

## JSONB Patterns

### When to Use JSONB

Use JSONB for:
- User preferences / settings with varying structure
- Metadata that differs per record
- Data from external APIs with unstable schemas
- Feature flags and configuration
- Form builder responses (dynamic fields)

Do NOT use JSONB for:
- Data you regularly filter, sort, or join on — make it a proper column
- Core domain fields — these deserve their own columns with types and constraints
- Anything where you need referential integrity

### JSONB with Type Safety

Define a TypeScript type alongside the Prisma model:

```prisma
model UserSettings {
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId      String @unique @map("user_id") @db.Uuid
  preferences Json   @default("{}") @db.JsonB

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@map("user_settings")
}
```

```typescript
// In your application code
interface UserPreferences {
  theme: 'light' | 'dark';
  language: string;
  notifications: {
    email: boolean;
    push: boolean;
  };
}

// Use Zod or similar for runtime validation
const preferencesSchema = z.object({
  theme: z.enum(['light', 'dark']).default('light'),
  language: z.string().default('en'),
  notifications: z.object({
    email: z.boolean().default(true),
    push: z.boolean().default(true),
  }).default({}),
});
```

### Indexing JSONB

```sql
-- GIN index for containment queries (@>, ?, ?|, ?&)
CREATE INDEX idx_settings_prefs ON user_settings USING GIN (preferences);

-- Specific path index for frequent equality checks
CREATE INDEX idx_settings_theme ON user_settings ((preferences->>'theme'));
```

## Full-Text Search

### Basic Setup

```prisma
// Enable in schema.prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch"]
}

model Job {
  id          String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  title       String @db.VarChar(255)
  description String @db.Text

  // search_vector column added via raw SQL (see below)

  @@map("jobs")
}
```

```sql
-- Add generated tsvector column
ALTER TABLE jobs ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
  ) STORED;

-- GIN index for fast search
CREATE INDEX idx_jobs_search ON jobs USING GIN (search_vector);
```

Query with Prisma raw SQL:

```typescript
const results = await prisma.$queryRaw`
  SELECT id, title, ts_rank(search_vector, query) AS rank
  FROM jobs, plainto_tsquery('english', ${searchTerm}) query
  WHERE search_vector @@ query
  ORDER BY rank DESC
  LIMIT 20
`;
```

### For More Advanced Search

If you need fuzzy matching, typo tolerance, faceted search, or relevance tuning beyond what PostgreSQL's built-in FTS offers, consider pg_trgm extension or an external search engine (Elasticsearch, Typesense, Meilisearch). PostgreSQL FTS is great for exact and stemmed matches but limited for "Google-like" search UX.

## Enums: PostgreSQL Native vs String

### Use Native Enum When:
- Values are stable and well-defined (status codes, roles, categories)
- Set is small (< 20 values)
- You want DB-level validation
- You won't need to remove values (PG enums can add values but removing is painful)

### Use String When:
- Values are user-defined or change frequently
- You need to remove values
- The set might grow large
- You need to deploy enum changes without a migration

### Renaming or Removing Enum Values

Adding a new value is simple. Removing or renaming requires a full migration:

```sql
-- Adding a value (easy)
ALTER TYPE application_status ADD VALUE 'ON_HOLD';

-- Renaming a value (PG 10+)
ALTER TYPE application_status RENAME VALUE 'UNDER_REVIEW' TO 'IN_REVIEW';

-- Removing a value (hard — requires recreating the type)
-- 1. Create new type without the value
-- 2. Alter all columns to use the new type
-- 3. Drop old type
-- 4. Rename new type
-- This is why you should be conservative about what you put in enums.
```

## Table Partitioning

For very large tables (100M+ rows), consider partitioning. Common strategies:

### Range Partitioning (Time-Series Data)

```sql
-- Partition audit_logs by month
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  performed_at TIMESTAMPTZ NOT NULL,
  -- ... other columns
) PARTITION BY RANGE (performed_at);

-- Create monthly partitions
CREATE TABLE audit_logs_2024_01 PARTITION OF audit_logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE audit_logs_2024_02 PARTITION OF audit_logs
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

Prisma doesn't natively support partitioned tables. Define them in raw SQL migrations. Prisma can still query the parent table normally.

### List Partitioning (Multi-Tenant)

```sql
-- Partition by tenant for large multi-tenant tables
CREATE TABLE events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  -- ... other columns
) PARTITION BY LIST (tenant_id);
```

## Database Extensions Worth Knowing

| Extension | Purpose | Enable With |
|-----------|---------|-------------|
| `uuid-ossp` | UUID generation (legacy, prefer `gen_random_uuid()`) | `CREATE EXTENSION IF NOT EXISTS "uuid-ossp";` |
| `citext` | Case-insensitive text type | `CREATE EXTENSION IF NOT EXISTS citext;` |
| `pg_trgm` | Trigram similarity / fuzzy search | `CREATE EXTENSION IF NOT EXISTS pg_trgm;` |
| `pg_stat_statements` | Query performance tracking | `CREATE EXTENSION IF NOT EXISTS pg_stat_statements;` |
| `btree_gin` | B-tree operators in GIN indexes | `CREATE EXTENSION IF NOT EXISTS btree_gin;` |
| `postgis` | Geospatial data and queries | `CREATE EXTENSION IF NOT EXISTS postgis;` |
| `pgcrypto` | Cryptographic functions | `CREATE EXTENSION IF NOT EXISTS pgcrypto;` |

Enable extensions in a Prisma migration:

```sql
-- In prisma/migrations/YYYYMMDD_extensions/migration.sql
CREATE EXTENSION IF NOT EXISTS citext;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
```
