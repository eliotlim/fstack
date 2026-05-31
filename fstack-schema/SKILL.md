---
name: fstack-schema
version: 0.1.0
description: Design a data model and migration honoring chosen ORM and DB. Tunes normalization, indexes, constraints to architecture_care and scope_appetite.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Glob
  - Grep
  - AskUserQuestion
triggers:
  - new table
  - new model
  - data model
  - schema design
  - migration
  - add column
---

# /fstack-schema

1. Get the change in one line.
2. Detect ORM + DB.
3. Calibrate from profile.
4. Propose.
5. Write. Update generated types. Log.

## Preamble

```bash
_FSTACK_BIN=""
_skill_md="$HOME/.claude/skills/fstack-schema/SKILL.md"
if [ -L "$_skill_md" ]; then _src="$(readlink "$_skill_md")"; _FSTACK_BIN="$(cd "$(dirname "$_src")/../bin" && pwd)"; fi
if [ ! -x "$_FSTACK_BIN/fstack-config" ]; then
  for cand in "$HOME/Workspaces/fstack/bin" "$HOME/fstack/bin"; do [ -x "$cand/fstack-config" ] && _FSTACK_BIN="$cand" && break; done
fi
"$_FSTACK_BIN/fstack-config" exists || { echo "uninitialized. run /fstack"; exit 0; }
echo "PROFILE:"
"$_FSTACK_BIN/fstack-profile" dimensions
echo "STACK:"
"$_FSTACK_BIN/fstack-config" get stack.database
"$_FSTACK_BIN/fstack-config" get stack.orm
```

## 1. What's the change?

Free text. Examples:

- `Add teams table with name, slug, owner_id, created_at.`
- `Add verified_at timestamp to users.`
- `Soft-delete posts with deleted_at + index.`

## 2. Detect ORM + DB

First match wins:

| Signal | ORM |
|--------|-----|
| `drizzle.config.ts` / `drizzle/` / drizzle imports | Drizzle |
| `prisma/schema.prisma` | Prisma |
| `kysely` import + `db.ts` | Kysely |
| `alembic.ini` + `env.py` | SQLAlchemy + Alembic |
| `models.py` with `declarative_base()` | SQLAlchemy |
| `db/migrate/` | ActiveRecord |
| plain `.sql` migrations | raw SQL |

DB: check `DATABASE_URL`, `docker-compose.yml`, ORM config.

Conflict with declared stack? Ask: declared / detected / tell me.

## 3. Calibrate

| Dim | Effect |
|-----|--------|
| `architecture_care` low | one table, PK only, no indexes |
| `architecture_care` mid | FKs, NOT NULL on important cols, basic indexes |
| `architecture_care` high | normalized, ON DELETE policies, partial + composite indexes, check constraints, timestamps, soft-delete |
| `scope_appetite` low | only what was asked |
| `scope_appetite` high | add helpers: `created_at`, `updated_at`, `deleted_at`, slug |
| `risk_tolerance` low | reversible migrations only |
| `risk_tolerance` high | one-way OK |
| `test_rigor` high | add seed/fixture |

"Just add X" beats declared profile.

## 4. Propose

```
Change: <one line>
ORM: <orm>  DB: <db>
Tables: <list>
Columns:
  - <col>: <type> [constraints]
Indexes:
  - <name> ON <table>(<cols>) [partial|unique]
FKs:
  - <table>.<col> → <other>.<col> [ON DELETE <policy>]
Migration: <path>
Reversibility: <fully|one-way>
```

AskUserQuestion: apply / change / cancel.

## 5. Write

Match the ORM:

**Drizzle:**

```ts
export const <table> = pgTable('<table>', {
  id: uuid('id').primaryKey().defaultRandom(),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
})
```

Then: `drizzle-kit generate`.

**Prisma:**

```prisma
model <Table> {
  id        String   @id @default(cuid())
  createdAt DateTime @default(now())
}
```

Then: `npx prisma migrate dev --name <slug>`.

**Alembic:** write `alembic/versions/<rev>_<slug>.py` with `op.create_table(...)` and a real `downgrade()`.

**Raw SQL:** write `migrations/<NNN>_<slug>.up.sql` + matching `.down.sql`. Skip down only if `risk_tolerance` is high AND the user opted in.

## 6. Propagate

Codegen after schema changes:

- Drizzle: `drizzle-kit generate`
- Prisma: `prisma generate`
- sqlc / kysely-codegen: regen bindings

Grep for typed queries on the changed table. List them. Ask before updating.

## 7. Log

```bash
"$_FSTACK_BIN/fstack-observe" log architecture_care <signal> --skill fstack-schema --context "<choices>"
"$_FSTACK_BIN/fstack-observe" log scope_appetite <signal> --skill fstack-schema --context "<extras>"
[ "$reversible" = "yes" ] && "$_FSTACK_BIN/fstack-observe" log risk_tolerance 0.25 --skill fstack-schema --context "reversible"
[ "$reversible" = "no" ] && "$_FSTACK_BIN/fstack-observe" log risk_tolerance 0.75 --skill fstack-schema --context "one-way"
```

## 8. Report

```
✓ Schema change applied
  ORM: <orm>  DB: <db>
  Migration: <path>
  Reversibility: <fully|one-way>
  Next: <run migrate | review diff>
```

Touches prod data (rename, drop)? Flag it.

## Voice

Concrete column lists, not "the data model". Reversibility line always. Match ORM idioms.
