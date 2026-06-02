**Model:** Codex 5.5 Medium

# Prompt 1:
## Role:
You are an expert software engineer specialized in database management.

## Context:
I need to update the DB of the current repository with new entities.
The goal is to generate a .sql to migrate the DB based on an ERD in mermaid.
The complete instructions are present in exercise-instructions.md file and some useful information is present in course-info.md file.
Ensure to read these 2 files.
These documents contain some images that were not copied. If you need any of them to verify specific information, ask me.
I do NOT run PostgreSQL databases on the standard 5432 port because it conflicts with other databases I already have. Consider using port 5452.

## Task:
Do NOT modify any file.
I want you to give me a complete prompt to run in an AI agent to modify this repository and comply with all the exercise instructions in exercise-instructions.md.
Do not make any assumptions. If any information is unclear or missing, ask me.
Consider using coding best practices and senior software engineering experience to deliver high-quality work.


---
**Model:** Claude Code auto mode

# Prompt 2:

## 1. Initial Prompt

```
Act as a Senior Software Engineer specialized in PostgreSQL, relational modeling,
normalization, indexes, and Prisma migrations.

Work directly on this repository. Implement and verify the full exercise.

Context: extend the current LTI data model to support the full hiring workflow
described in the ERD. Existing models (Candidate, Education, WorkExperience, Resume)
must NOT be removed or degraded.

Delivery: update backend/prisma/schema.prisma, generate a reproducible Prisma
migration at backend/prisma/migrations/<timestamp>_add_hiring_workflow/migration.sql,
and create this file at prompts/prompts-LL.md.

Local PostgreSQL runs on port 5452 (via Docker Compose). Temporarily update .env to
DB_PORT=5452, run all work, then restore .env exactly.

Before editing any file, present the implementation plan and proposed schema changes,
then wait for explicit approval.
```

---

## 2. Follow-up Prompts

**User approval:** "Yes, proceed."

No additional prompts were required. The implementation was fully autonomous after
the user approved the plan.

---

## 3. Summary of Technical Decisions

### 3.1 Preserved existing models without modification

The original four models (`Candidate`, `Education`, `WorkExperience`, `Resume`) were
kept byte-for-byte identical in terms of columns and types. The only addition was a
virtual Prisma relation field (`applications Application[]`) on `Candidate`, which
does not add any column to the database table.

### 3.2 String fields use VarChar with bounded lengths

All short text fields use `@db.VarChar(N)` to enforce reasonable bounds at the
database level. Long descriptive fields (`jobDescription`, `requirements`, etc.) use
`@db.Text` (PostgreSQL TEXT, unbounded). This is consistent with the existing schema
style.

### 3.3 Date vs DateTime

- `Position.applicationDeadline` → `@db.Date` (only date matters, no time)
- `Application.applicationDate` → `@db.Date` (submission date, no time)
- `Interview.interviewDate` → `DateTime` / `TIMESTAMP(3)` (time of interview matters)

### 3.4 No enums used

Fields like `status`, `result`, `employmentType`, and `role` are modeled as
`VarChar` rather than PostgreSQL enums. Reasoning: enum types in PostgreSQL require
DDL changes to add new values, making future evolution painful. VarChar strings are
easier to extend and validate at the application layer.

### 3.5 Salary as DECIMAL(10,2)

`salaryMin` / `salaryMax` use `Decimal` (`DECIMAL(10,2)`) for exact fixed-point
arithmetic. Using `Float` would introduce floating-point rounding errors for
monetary amounts.

### 3.6 All FK columns are indexed

Every foreign key column in every new table has at least a B-tree index, either via a
dedicated `@@index`, a `@@unique` constraint that covers it, or a compound index. This
prevents sequential scans on parent-to-child lookups.

---

## 4. Justified ERD Corrections

The ERD as provided contained three 1:1 relationships that are semantically
many-to-one and were corrected accordingly.

### 4.1 POSITION ↔ INTERVIEW_FLOW

| Attribute | Value |
|---|---|
| ERD cardinality | `\|\|--\|\|` (1:1) — one flow belongs to exactly one position |
| Implemented | Many-to-one (N:1) — many positions can share one flow |
| Prisma | `Position.interviewFlowId Int` + FK to `InterviewFlow` |

**Justification:** An `InterviewFlow` represents a reusable hiring process template
(e.g., "Standard Engineering Process"). Multiple positions with the same profile
(e.g., "Senior Engineer — Team A" and "Senior Engineer — Team B") should share the
same flow. Forcing 1:1 would require duplicating flow records for every position,
violating 3NF (the flow's steps would be duplicated facts). The composite index
`@@index([companyId, status])` on `Position` and `@@index([interviewFlowId])` support
efficient lookups without adding a redundant back-pointer on `InterviewFlow`.

### 4.2 INTERVIEW_STEP ↔ INTERVIEW_TYPE

| Attribute | Value |
|---|---|
| ERD cardinality | `\|\|--\|\|` (1:1) — one type is used by exactly one step |
| Implemented | Many-to-one (N:1) — many steps can share one type |
| Prisma | `InterviewStep.interviewTypeId Int` + FK to `InterviewType` |

**Justification:** `InterviewType` is a catalog entity (e.g., "Technical Interview",
"HR Screen", "Cultural Fit"). Steps in different flows — or even different steps in
the same flow — may legitimately share the same type. A 1:1 constraint would make it
impossible to reuse type definitions, again violating normalization. The
`@@index([interviewTypeId])` on `InterviewStep` supports fast lookup of all steps
using a given type.

### 4.3 INTERVIEW ↔ INTERVIEW_STEP

| Attribute | Value |
|---|---|
| ERD cardinality | `\|\|--\|\|` (1:1) — one step belongs to exactly one interview |
| Implemented | Many-to-one (N:1) — many interviews (for different applications) can reference the same step |
| Prisma | `Interview.interviewStepId Int` + FK to `InterviewStep` |

**Justification:** `InterviewStep` is the *template* (e.g., "Step 2: Technical
Assessment"). The concrete `Interview` record is the *instance* — it records that a
specific candidate, via a specific application, went through that step. Many
candidates will all pass through the same step. A 1:1 would mean a step could only
ever be used once, making the step model meaningless as a reusable construct. The
`@@unique([applicationId, interviewStepId])` constraint enforces the correct domain
rule: one interview per step per application.

---

## 5. Indexes and Constraints Reference

| Index / Constraint | Table | Columns | Type | Reason |
|---|---|---|---|---|
| `Candidate_email_key` | Candidate | email | UNIQUE | Existing, identity uniqueness |
| `Employee_email_key` | Employee | email | UNIQUE | Identity uniqueness |
| `Employee_companyId_idx` | Employee | companyId | B-tree | FK lookup: employees of a company |
| `InterviewStep_interviewFlowId_orderIndex_key` | InterviewStep | (interviewFlowId, orderIndex) | UNIQUE | No duplicate step order in a flow; also serves as FK index on interviewFlowId |
| `InterviewStep_interviewTypeId_idx` | InterviewStep | interviewTypeId | B-tree | FK lookup: steps using a given type |
| `Position_companyId_status_idx` | Position | (companyId, status) | Compound B-tree | Core dashboard: open positions per company |
| `Position_interviewFlowId_idx` | Position | interviewFlowId | B-tree | FK lookup: positions using a given flow |
| `Application_positionId_candidateId_key` | Application | (positionId, candidateId) | UNIQUE | One application per candidate per position; covers FK on positionId |
| `Application_candidateId_idx` | Application | candidateId | B-tree | FK lookup: all applications by a candidate |
| `Interview_applicationId_interviewStepId_key` | Interview | (applicationId, interviewStepId) | UNIQUE | One interview per step per application; covers FK on applicationId |
| `Interview_interviewStepId_idx` | Interview | interviewStepId | B-tree | FK lookup: all interviews at a given step |
| `Interview_employeeId_idx` | Interview | employeeId | B-tree | FK lookup: all interviews conducted by an employee |

---

## 6. Verification Commands Executed

```bash
# 1. Format schema
./node_modules/.bin/prisma format

# 2. Validate schema
set -a && source ../.env && set +a
./node_modules/.bin/prisma validate

# 3. Generate migration (--create-only to review before applying)
./node_modules/.bin/prisma migrate dev --create-only --name add_hiring_workflow

# 4. Apply migration
./node_modules/.bin/prisma migrate deploy

# 5. Check migration status
./node_modules/.bin/prisma migrate status

# 6. Verify tables
docker exec ai4devs-db-2603-db-1 psql -U LTIdbUser -d LTIdb -c \
  "SELECT tablename FROM pg_tables WHERE schemaname = 'public' ORDER BY tablename;"

# 7. Verify foreign keys
docker exec ai4devs-db-2603-db-1 psql -U LTIdbUser -d LTIdb -c \
  "SELECT tc.table_name, kcu.column_name, ccu.table_name AS foreign_table_name ...;"

# 8. Verify indexes
docker exec ai4devs-db-2603-db-1 psql -U LTIdbUser -d LTIdb -c \
  "SELECT indexname, tablename, indexdef FROM pg_indexes WHERE schemaname='public'...;"

# 9. End-to-end test data + full JOIN query (all data deleted after verification)

# 10. Generate Prisma client
./node_modules/.bin/prisma generate

# 11. Build backend TypeScript
npm run build
```

Results:
- `prisma format`: OK
- `prisma validate`: schema valid
- `prisma migrate status`: "Database schema is up to date!"
- `npm run build`: compiled with no TypeScript errors
- 12 user tables created, 13 FK constraints, 24 indexes/constraints

---

## 7. Manual DBeaver Verification Checklist

Use DBeaver to connect to the local PostgreSQL container and inspect the schema.

### Connection details

| Parameter | Value |
|---|---|
| Host | localhost |
| Port | **5452** (container still running on this port) |
| Database | LTIdb |
| User | LTIdbUser |
| Password | (read from `.env` — not shown here) |
| SSL | disabled (sslmode=disable) |

### Steps

1. **Connect**: open DBeaver, create a new PostgreSQL connection with the details above.

2. **Review tables** — expand `LTIdb > Schemas > public > Tables` and verify these 12 tables are present:
   - `Application`, `Candidate`, `Company`, `Education`, `Employee`,
     `Interview`, `InterviewFlow`, `InterviewStep`, `InterviewType`,
     `Position`, `Resume`, `WorkExperience`

3. **Review foreign keys** — for each table, open the "Foreign Keys" tab and confirm the FKs listed in section 4 of this file are present.

4. **Review indexes** — for each table, open the "Indexes" tab and confirm the indexes from section 5 above are present.

5. **Execute a representative JOIN query** — open a SQL editor on `LTIdb` and run:

```sql
SELECT
  co.name              AS company,
  p.title              AS position,
  p.status             AS position_status,
  iflow.description    AS interview_flow,
  istep.name           AS step_name,
  istep."orderIndex"   AS step_order,
  itype.name           AS step_type
FROM "Company"       co
JOIN "Position"      p     ON p."companyId"       = co.id
JOIN "InterviewFlow" iflow ON iflow.id             = p."interviewFlowId"
JOIN "InterviewStep" istep ON istep."interviewFlowId" = iflow.id
JOIN "InterviewType" itype ON itype.id             = istep."interviewTypeId"
ORDER BY co.name, p.title, istep."orderIndex";
```

   This query returns zero rows on an empty database (as expected after cleanup),
   but it validates that all table references and column names are correctly resolved.

6. **Verify `_prisma_migrations` table** — confirm one row exists with migration name
   `20260602190559_add_hiring_workflow` and `finished_at` is populated (not NULL).

---

## 8. Limitations and Risks

- **No soft-delete**: the schema does not implement soft-delete (no `deletedAt`
  column). Deleting a `Company` is blocked by FK constraints while employees or
  positions exist (ON DELETE RESTRICT), which is safe but may require application-level
  handling.

- **`status` fields are unconstrained strings**: `Position.status`,
  `Application.status`, and `Interview.result` accept any string value. A CHECK
  constraint or application-layer enum would prevent invalid values. A future
  migration can add `ALTER TABLE ... ADD CONSTRAINT ... CHECK (status IN (...))`.

- **Salary currency not modeled**: `salaryMin`/`salaryMax` have no currency column.
  International positions would need a `salaryCurrency VarChar(3)` field (ISO 4217).

- **Port mismatch note**: the `.env` file has been restored to `DB_PORT=5432` (the
  original value). The Docker container remains running on port **5452** for DBeaver
  inspection. Restarting the container with `docker-compose up -d` (after `.env`
  restoration) would map it back to 5432.
