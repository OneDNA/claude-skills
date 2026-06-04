---
name: dbml-logical-to-physical
description: Convert a DBML logical schema (engine-agnostic types) to a physical DBML schema for a target database engine (Databricks SQL, PostgreSQL, MySQL, etc.). Use when the user has a logical DBML and wants a physical implementation, or asks to migrate/translate a schema between abstraction layers or engines.
---

# DBML: Logical → Physical Schema Conversion

A logical schema uses engine-agnostic types and omits engine-specific features. A physical schema maps those to the target engine's native types and adds performance, constraint, and identity constructs.

## The Two Layers

### Logical DBML
- Table names in PascalCase (e.g. `Employee`, `WorkRelation`)
- Types: `integer`, `string`, `boolean`, `date`, `timestamp`, `text`, `decimal`
- PKs: `[pk]` only — no identity/autoincrement strategy specified
- No indexes unless logically meaningful (unique constraints)
- No engine-specific notes or settings
- Reflects the Logical ERD; audience is data architect

### Physical DBML
- Table names in snake_case (e.g. `employees`, `work_relations`)
- Types mapped to engine natives (see tables below)
- PKs get identity/surrogate strategy (`[pk, increment]` or `GENERATED ALWAYS AS IDENTITY`)
- Indexes added for all FK columns and common query patterns
- Engine-specific notes, constraints, clustering directives added
- Reflects the DDL; audience is engineer

## Type Mapping Tables

### → Databricks SQL
| Logical | Physical |
|---|---|
| `integer` (PK) | `BIGINT` — note `'GENERATED ALWAYS AS IDENTITY'` |
| `integer` (FK/regular) | `BIGINT` |
| `string` | `STRING` (no varchar — Databricks STRING is unbounded) |
| `boolean` | `BOOLEAN` |
| `date` | `DATE` |
| `timestamp` | `TIMESTAMP` |
| `text` | `STRING` (no text type in Databricks SQL) |
| `decimal(p,s)` | `DECIMAL(p,s)` |
| `uuid` | `BIGINT` surrogate key (Databricks uses integer surrogates, not UUIDs) |

**DBML-specific note:** DBML's `[pk, increment]` is a PostgreSQL convention and does not mean `GENERATED ALWAYS AS IDENTITY` in Databricks. Always use `[pk, note: 'GENERATED ALWAYS AS IDENTITY']` instead.

Databricks-specific additions:
- `CLUSTER BY (col1, col2)` — Liquid Clustering; add to every table via `note:` field in DBML
- `ANALYZE TABLE tbl COMPUTE STATISTICS` — add to `note:` field
- `TBLPROPERTIES ('delta.enableChangeDataFeed' = 'true')` for SCD tables
- `PRIMARY KEY` / `FOREIGN KEY` constraints are `NOT ENFORCED` but inform the query optimizer — document in `note:` since DBML refs already capture the relationship

**Correct Databricks PK pattern in DBML:**
```dbml
Table employees {
  id  BIGINT  [pk, note: 'GENERATED ALWAYS AS IDENTITY']
  ...
  note: 'CLUSTER BY (id) | ANALYZE TABLE employees COMPUTE STATISTICS'
}
```

**Incorrect (PostgreSQL convention, not Databricks):**
```dbml
Table employees {
  id  integer  [pk, increment]   // wrong: integer not BIGINT, increment not valid in Databricks
  id  uuid     [pk]              // wrong: Databricks uses BIGINT surrogates, not UUIDs
  id  varchar  [pk]              // wrong: no varchar in Databricks SQL, use STRING
}
```

### → PostgreSQL
| Logical | Physical |
|---|---|
| `integer` (PK) | `SERIAL` or `INTEGER GENERATED ALWAYS AS IDENTITY` |
| `integer` (FK/regular) | `INTEGER` |
| `string` | `VARCHAR(n)` — choose n based on context (100 for names, 255 for email, 50 for codes) |
| `boolean` | `BOOLEAN` |
| `date` | `DATE` |
| `timestamp` | `TIMESTAMPTZ` |
| `text` | `TEXT` |
| `decimal(p,s)` | `NUMERIC(p,s)` |
| `uuid` | `UUID` with `gen_random_uuid()` default |

PostgreSQL-specific additions:
- `CREATE INDEX ON tbl(col)` for all FK columns
- `CHECK` constraints inline
- `DEFAULT now()` for audit timestamps

### → MySQL / MariaDB
| Logical | Physical |
|---|---|
| `integer` (PK) | `INT AUTO_INCREMENT` |
| `integer` (FK/regular) | `INT` |
| `string` | `VARCHAR(n)` |
| `boolean` | `TINYINT(1)` |
| `date` | `DATE` |
| `timestamp` | `DATETIME` |
| `text` | `TEXT` |
| `decimal(p,s)` | `DECIMAL(p,s)` |

## Naming Conventions

| Logical (PascalCase) | Physical (snake_case) |
|---|---|
| `Employee` | `employees` |
| `WorkRelation` | `work_relations` |
| `PerformanceReview` | `performance_reviews` |
| `Contract` | `contracts` |
| `Department` | `departments` |
| `Leave` | `leaves` |

Rule: PascalCase → lowercase with underscores before each uppercase transition.

## Conversion Procedure

1. **Read the logical DBML** — identify all tables, columns, types, refs, and notes.
2. **Determine target engine** — ask if not specified.
3. **Map each table**:
   - Rename to snake_case plural
   - Map each column type using the table above
   - Upgrade PK to identity/surrogate strategy
   - Keep all `[not null]`, `[unique]`, `[default: ...]`, `[note: ...]` constraints
   - Keep all `[ref: ...]` relationships, updating table/column names to snake_case
4. **Add physical concerns**:
   - Indexes on all FK columns
   - Composite unique indexes where logically implied
   - Clustering / partitioning directives for the target engine
   - CHECK constraints for validated fields (e.g. `rating` 1–5)
5. **Write the output file** as `<name>-<engine>.dbml` or `<name>-physical.dbml`
6. **Add a header comment** clarifying this is the physical layer and referencing the logical file

## Example Conversion

**Logical input:**
```dbml
Table Employee {
  id          integer   [pk]
  first_name  string    [not null]
  email       string    [unique, not null]
  hire_date   date      [not null]
  is_active   boolean   [default: true]
  job_id      integer   [not null, ref: > Job.id]
}
```

**Physical output (Databricks SQL):**
```dbml
Table employees {
  id          BIGINT   [pk, note: 'GENERATED ALWAYS AS IDENTITY']
  first_name  STRING   [not null]
  email       STRING   [unique, not null]
  hire_date   DATE     [not null]
  is_active   BOOLEAN  [default: true]
  job_id      BIGINT   [not null, ref: > jobs.id]

  indexes {
    job_id
  }

  note: 'CLUSTER BY (id) | ANALYZE TABLE employees COMPUTE STATISTICS'
}
```

**Physical output (PostgreSQL):**
```dbml
Table employees {
  id          SERIAL        [pk]
  first_name  VARCHAR(100)  [not null]
  email       VARCHAR(255)  [unique, not null]
  hire_date   DATE          [not null]
  is_active   BOOLEAN       [default: true]
  job_id      INTEGER       [not null, ref: > jobs.id]

  indexes {
    job_id
  }
}
```

## Self-Referencing FKs

When a table has two FKs pointing to the same table (e.g. `reviewee_id` and `reviewer_id` both → `Employee`), both refs must be preserved in the physical schema. DBML supports multiple refs to the same table — just write them on separate columns as normal.

```dbml
// Logical
Table PerformanceReview {
  reviewee_id  integer [ref: > Employee.id]
  reviewer_id  integer [ref: > Employee.id]
}

// Physical (Databricks)
Table performance_reviews {
  reviewee_key  BIGINT [not null, ref: > employees.id]
  reviewer_key  BIGINT [not null, ref: > employees.id]

  indexes {
    reviewee_key
    reviewer_key
    (reviewee_key, reviewed_at)
  }
}
```

## Output Checklist

- [ ] All table names in snake_case plural
- [ ] All column names in snake_case
- [ ] All types mapped to engine natives
- [ ] All PKs have identity/surrogate strategy
- [ ] All FK columns have an index entry
- [ ] All `[not null]`, `[unique]`, `[default]` constraints preserved
- [ ] All `[ref: >]` relationships updated to snake_case names
- [ ] Self-referencing FKs both present
- [ ] Engine-specific clustering/partitioning added
- [ ] Header comment references the logical source file
