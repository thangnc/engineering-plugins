---
name: postgres-prisma-specialist
description: Design PostgreSQL-specific database schemas using Prisma ORM on Node.js. Use this skill whenever the user mentions database design, schema design, data modeling, Prisma schema, database migrations, PostgreSQL tables, indexes, constraints, relations, or any database architecture decisions. Also trigger when the user asks about query performance, N+1 problems, composite indexes, full-text search, JSONB columns, enum types, soft deletes, multi-tenancy, polymorphic relations, audit trails, or any Prisma-specific patterns like @@map, @db.*, or raw SQL. Even if the user just says "design a database" or "model this data" or "set up the tables for my app," use this skill. Covers best practices, data types, indexing strategy, constraints, performance patterns, and advanced PostgreSQL features — all through the Prisma lens.
---

# PostgreSQL Prisma Specialist (Prisma + Node.js)

Design production-grade PostgreSQL schemas using Prisma ORM. This skill encodes hard-won patterns for data modeling, type selection, indexing, constraints, and performance — specifically tuned for Prisma's capabilities and PostgreSQL's advanced features.

## Core Philosophy

Good schema design is the foundation everything else rests on. A well-designed schema makes queries fast, application code simple, and migrations painless. A poorly designed one creates compounding technical debt. Approach every schema decision by asking: "Will this still make sense when we have 10M rows and 5 developers working on this?"

## Process

When the user asks you to design or review a schema, follow this sequence:

1. **Understand the domain** — Ask about entities, relationships, access patterns, and scale expectations. Don't jump to tables before you understand the business.
2. **Read the references** — Consult the relevant reference files in `references/` for detailed patterns (see Reference Files section below).
3. **Model the entities** — Start with a conceptual model, then translate to Prisma schema.
4. **Choose types deliberately** — Use the type selection guide to pick the right PostgreSQL type for each field.
5. **Design indexes for queries** — Don't guess. Think about what queries will actually run, then index for those.
6. **Add constraints for data integrity** — The database should enforce business rules, not just the application.
7. **Output a complete `schema.prisma` file** — With clear comments explaining non-obvious decisions.
8. **Provide migration notes** — Especially for anything that needs raw SQL beyond what Prisma generates.

## Reference Files

Read these before generating a schema. They contain the detailed patterns and rules.

- **`references/types-and-fields.md`** — PostgreSQL type selection, Prisma type mappings, field-level best practices. Read this when choosing data types or field attributes.
- **`references/indexes-and-performance.md`** — Indexing strategies, query patterns, performance anti-patterns, connection pooling. Read this when designing indexes or addressing performance.
- **`references/advanced-patterns.md`** — Multi-tenancy, polymorphism, soft deletes, audit trails, JSONB, full-text search, enums, and advanced PostgreSQL features. Read this for any non-trivial schema pattern.

## Prisma Schema Conventions

Follow these conventions in every schema you produce:

### Naming
- Models: `PascalCase` singular (`User`, `JobApplication`, not `users` or `job_applications`)
- Fields: `camelCase` (`firstName`, `createdAt`)
- Map to snake_case in PostgreSQL using `@@map` and `@map`:

```prisma
model JobApplication {
  id        String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  createdAt DateTime @default(now()) @map("created_at")

  @@map("job_applications")
}
```

### IDs
- Default to UUIDs for primary keys. Use `gen_random_uuid()` (native PostgreSQL, no extension needed on PG 13+):

```prisma
id String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
```

- Use `autoincrement()` only when you need ordered, compact IDs (rare — analytics tables, internal sequences).
- Never expose autoincrement IDs in public APIs (enumeration attacks).

### Timestamps
Every table gets these three fields:

```prisma
createdAt DateTime  @default(now()) @map("created_at")
updatedAt DateTime  @updatedAt @map("updated_at")
deletedAt DateTime? @map("deleted_at")  // for soft deletes — omit if not needed
```

### Relations
- Always define both sides of a relation.
- Always add `onDelete` explicitly — never rely on Prisma's default (`SetNull` or error). Think about what should actually happen.
- Add `@relation(name: "...")` when a model has multiple relations to the same target.

```prisma
model Comment {
  id       String @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  authorId String @map("author_id") @db.Uuid
  postId   String @map("post_id") @db.Uuid

  author User @relation(fields: [authorId], references: [id], onDelete: Cascade)
  post   Post @relation(fields: [postId], references: [id], onDelete: Cascade)

  @@map("comments")
}
```

### Enums
Use PostgreSQL native enums for values that rarely change and have a small, fixed set:

```prisma
enum ApplicationStatus {
  DRAFT
  SUBMITTED
  UNDER_REVIEW
  INTERVIEW
  OFFER
  HIRED
  REJECTED
  WITHDRAWN

  @@map("application_status")
}
```

If the values change frequently or are user-defined, use a `String` field with application-level validation instead.

## Schema Output Format

When producing a schema, always output a complete `schema.prisma` file with:

1. A `generator` and `datasource` block at the top
2. Enums grouped together after the datasource
3. Models ordered by domain relevance (core entities first, junction tables last)
4. Comments explaining why for any non-obvious decision — especially index choices, type choices, and constraint rationale
5. A section after the schema titled **Migration Notes** listing anything that requires raw SQL (partial indexes, check constraints, triggers, etc.)
6. A section titled **Query Patterns & Index Rationale** mapping each index to the query it serves

### Template

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["fullTextSearch"]  // only if needed
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── Enums ────────────────────────────────────────────

// ... enums here ...

// ─── Core Models ──────────────────────────────────────

// ... models here ...

// ─── Junction / Supporting Models ─────────────────────

// ... junction tables here ...
```

## Common Mistakes to Catch

When reviewing or designing schemas, watch for these:

- **Missing indexes on foreign keys** — Prisma does NOT auto-create indexes on foreign key columns. Always add `@@index([foreignKeyField])`.
- **String fields without length limits** — Use `@db.VarChar(N)` to cap length. Unbounded `Text` invites garbage data.
- **Over-indexing** — Every index slows writes. Only index fields that appear in `WHERE`, `ORDER BY`, or `JOIN` clauses of real queries.
- **Storing money as Float** — Use `Decimal` with `@db.Decimal(19, 4)`. Floats lose precision.
- **Missing unique constraints** — If something should be unique (emails, slugs, external IDs), enforce it at the DB level, not just application code.
- **N+1 relation loading** — Design schemas with Prisma's `include` and `select` in mind. Deeply nested relations suggest the schema might need flattening or a different access pattern.
- **No composite index on frequently filtered + sorted queries** — If you always filter by `tenantId` and sort by `createdAt`, you need `@@index([tenantId, createdAt])`, not two separate indexes.

## When the User Asks About...

| Topic | What to do |
|-------|-----------|
| "Design a schema for X" | Full process: understand domain → model → types → indexes → output schema.prisma |
| "Review my schema" | Audit against the common mistakes list and reference docs, suggest improvements |
| "Add a feature to my schema" | Understand the feature, check references, produce incremental migration |
| "Performance issue" | Read `references/indexes-and-performance.md`, analyze query patterns, suggest index changes |
| "Multi-tenancy" | Read `references/advanced-patterns.md`, discuss RLS vs column-based isolation |
| "How do I model X?" | Check `references/advanced-patterns.md` for the pattern, explain trade-offs |
| "Prisma migration" | Produce the schema change + any raw SQL needed in a migration file |
