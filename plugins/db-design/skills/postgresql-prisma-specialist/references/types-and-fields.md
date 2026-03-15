# Types & Fields Reference

## PostgreSQL ↔ Prisma Type Mapping

Use this table when choosing field types. Always prefer the most specific type that fits.

### Identifiers

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| Primary key (default) | `String` | `uuid` | `@db.Uuid` | Use `gen_random_uuid()`. PG 13+ native. |
| Primary key (ordered) | `Int` | `serial` / `bigserial` | `@default(autoincrement())` | Only for internal IDs, analytics. Never expose publicly. |
| External ref / slug | `String` | `varchar(N)` | `@db.VarChar(N)` | Always cap length. |
| CUID / ULID | `String` | `varchar(26)` / `varchar(32)` | `@db.VarChar(26)` | Use `@default(cuid())` or generate in app. |

### Text

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| Short string (name, title) | `String` | `varchar(N)` | `@db.VarChar(255)` | Always set a max length. |
| Email | `String` | `citext` | `@db.VarChar(320)` | Use `citext` extension for case-insensitive. Max email length is 320. |
| URL | `String` | `varchar(2048)` | `@db.VarChar(2048)` | Standard max URL length. |
| Long text (description) | `String` | `text` | `@db.Text` | For user-generated content, rich text, markdown. |
| Searchable text | `String` | `tsvector` | Needs raw SQL | See full-text search in advanced-patterns.md. |

### Numbers

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| Integer count | `Int` | `integer` | (default) | Fits up to ~2.1B. |
| Large counter | `BigInt` | `bigint` | `@db.BigInt` | Use for anything that might exceed 2.1B (event counts, IDs from external systems). |
| Money / precise decimal | `Decimal` | `numeric(19,4)` | `@db.Decimal(19, 4)` | NEVER use Float for money. |
| Percentage / ratio | `Decimal` | `numeric(5,4)` | `@db.Decimal(5, 4)` | Stores 0.0000–1.0000. |
| Approximate number | `Float` | `double precision` | `@db.DoublePrecision` | Only for scientific data, coordinates, scores where precision loss is OK. |
| Small integer / flag | `Int` | `smallint` | `@db.SmallInt` | Use for status codes, small ranges. |

### Dates & Times

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| Timestamp with TZ | `DateTime` | `timestamptz` | `@db.Timestamptz()` | Default for all timestamps. Always use timezone-aware. |
| Date only | `DateTime` | `date` | `@db.Date` | Birthdays, deadlines. No time component. |
| Time only | `DateTime` | `time` | `@db.Time()` | Rare. Business hours, schedules. |
| Duration / interval | `String` | `interval` | Needs raw SQL | Store as ISO 8601 duration string if using Prisma. |

### Boolean & Enums

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| True/false | `Boolean` | `boolean` | (default) | Always set `@default(false)` or `@default(true)` — avoid nullable booleans. |
| Fixed set of values | `enum` | Native PG enum | Define as Prisma `enum` | Use when values are stable and small (<20 values). |
| Flexible set of values | `String` | `varchar(N)` | `@db.VarChar(50)` | When values change often or are user-defined. Validate in app. |

### JSON & Structured Data

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| Flexible metadata | `Json` | `jsonb` | `@db.JsonB` | JSONB, not JSON. Always JSONB — it's indexable and faster. |
| Configuration / settings | `Json` | `jsonb` | `@db.JsonB` | Good for user preferences, feature flags. |
| Typed JSON (known shape) | `Json` | `jsonb` | `@db.JsonB` | Define a TypeScript interface alongside for type safety. |

### Binary & Special

| Use Case | Prisma Type | PostgreSQL Native | Prisma Attribute | Notes |
|----------|-------------|-------------------|------------------|-------|
| File content | `Bytes` | `bytea` | `@db.ByteA` | Prefer object storage (S3) for large files; store URL instead. |
| IP address | `String` | `inet` | Needs raw SQL | PG `inet` type supports both IPv4 and IPv6. |
| Network range | `String` | `cidr` | Needs raw SQL | For IP range storage. |
| Geolocation | `Float[]` | `point` or PostGIS | Needs raw SQL / extension | For basic lat/lng, two Float fields suffice. For spatial queries, use PostGIS. |

## Field-Level Best Practices

### Nullable vs Required

- **Default to required** (`NOT NULL`). Make fields nullable only when `null` has a meaningful business interpretation (e.g., "not yet provided", "not applicable").
- **Never use nullable booleans.** A nullable boolean has three states (true, false, null), which creates ambiguity. Use an enum if you need three states.
- Nullable fields with `@default` are almost always wrong — if it has a default, it should be required.

### Default Values

- `@default(now())` for `createdAt`
- `@updatedAt` for `updatedAt` (Prisma handles this automatically)
- `@default(false)` for boolean flags
- `@default(dbgenerated("gen_random_uuid()"))` for UUID primary keys
- `@default(0)` for counters
- Avoid application-generated defaults for anything the DB can handle — let the database be the source of truth.

### Unique Constraints

- Always add `@unique` for natural keys: emails, usernames, slugs, external IDs.
- Use compound unique constraints (`@@unique([tenantId, email])`) for multi-tenant uniqueness.
- Unique constraints automatically create indexes — no need to add a separate `@@index`.

### Check Constraints (Raw SQL)

Prisma doesn't support `CHECK` constraints natively. Add them in migrations:

```sql
-- In a Prisma migration SQL file
ALTER TABLE job_applications
  ADD CONSTRAINT check_salary_range
  CHECK (min_salary >= 0 AND (max_salary IS NULL OR max_salary >= min_salary));
```

Document every check constraint in comments in your schema.prisma file.
