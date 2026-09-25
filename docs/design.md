# Nautilith: design

How Nautilith works and why. This is the spec the first implementation gets built against.

## Goals

- **One file tells the truth.** `schema.sql` is always the complete, current schema, readable top to
  bottom, with no `ALTER` history in it.
- **Write changes the way you already do.** Plain SQL, imperative, with an explicit rollback. No XML,
  YAML or DSL.
- **Keep what Liquibase and Flyway users rely on:** environments, a history table, checksums,
  rollbacks, drift detection, baselining.
- **Small.** One binary, three concepts (schema, changes, ledger), no server.
- **Agent-first.** Deterministic output, JSON everywhere, error messages that say what to do next.

## Non-goals, for now

- Generating changes from a desired state (declarative diffing). Other tools do this well; see
  [Later: capture](#later-capture) for how we might borrow it.
- Roles, grants and other cluster-level objects.
- Zero-downtime orchestration (expand/contract). The change format leaves room for it.

## Vocabulary

| Term | Meaning |
|---|---|
| **schema** | `db/schema.sql`, the golden source. Generated. The complete schema at the latest version (HEAD). |
| **change** | A SQL file with an `@up` section and usually a `@down` section. Lives in `db/changes/` until folded. |
| **fold** | Run pending changes in a shadow database that holds the schema, verify them, regenerate `schema.sql`, move the changes into the ledger. |
| **ledger** | `db/.ledger/`: folded changes, numbered by version, kept until every environment has them. Read by the tool, not by people or agents. |
| **version** | Integer assigned when a change is folded. HEAD is the latest. |
| **change id** | Stable identity of a change, taken from its file name (`20260925T1412-add-user-email`). Survives rebases; versions don't. |
| **history** | The `nautilith_history` table inside each database: what ran there, when, by whom, and the SQL to undo it. |
| **environment** | A named database target in `nautilith.toml`. |
| **shadow** | A throwaway database used for folding and verification. Never holds real data. |
| **horizon** | The lowest version any tracked environment is at. Ledger entries at or below it can be compacted. |

## Layout

```text
db/
├── schema.sql
├── changes/
│   └── 20260925T1412-add-user-email.sql
└── .ledger/
    ├── 0041-20260918T0930-create-invoices.sql
    └── 0042-20260925T1412-add-user-email.sql
nautilith.toml
```

```toml
# nautilith.toml
dialect = "postgres"
dir     = "db"
shadow  = { provider = "docker", image = "postgres:17" }  # or a scratch server URL, or "embedded"

[env.local]
url     = "postgres://localhost:5432/app_dev"
pending = true                  # may apply unfolded changes from db/changes/

[env.staging]
url   = "${STAGING_DATABASE_URL}"
track = true                    # counts toward the compaction horizon

[env.prod]
url     = "${DATABASE_URL}"
track   = true
confirm = true
```

`nautilith init` also writes a `.gitattributes` line so GitHub collapses ledger diffs:

```gitattributes
db/.ledger/** linguist-generated=true
```

`schema.sql` is deliberately *not* marked as generated: its diff is the most useful thing a reviewer can
read.

## Change files

```sql
-- @up
ALTER TABLE users ADD COLUMN email text;

-- @down
ALTER TABLE users DROP COLUMN email;
```

| Directive | Meaning |
|---|---|
| `-- @up` | Start of the change |
| `-- @down` | Start of the rollback |
| `-- @down irreversible` | There is no rollback; `rollback` stops here unless forced |
| `-- @down lossy` | The rollback restores an equivalent schema, not an identical one (for example, column order) |
| `-- @env dev, test` | Run only in these environments (default: all) |
| `-- @no-transaction` | Run outside a transaction, for example `CREATE INDEX CONCURRENTLY` |

- Directives are lines that start with `-- @`. Everything else is plain SQL, passed to the database as is.
- A change without `@down` is treated as irreversible, and `check` says so.
- File names are `<UTC timestamp>-<slug>.sql`, created by `nautilith new`. The timestamp orders pending
  changes; the whole stem is the change id.
- **Pending changes are mutable.** Edit one and `nautilith apply --env local` undoes the old version with
  the down SQL it stored, then applies the new one. That gives a tight edit-and-rerun loop.
- **Folded changes are immutable.** Their checksum is recorded in the ledger and in every database that
  runs them.

## The fold

For each pending change, in change-id order:

```text
shadow  ← empty database, engine and major version pinned in config
load schema.sql into shadow
assert render(shadow) == body(schema.sql)       # fixpoint: the file reproduces itself
before  ← render(shadow)

run change.up
after   ← render(shadow)

if change is reversible:
    run change.down
    assert render(shadow) == before              # the rollback is proven
    run change.up                                # continue from the new state

schema.sql                      ← header(v + 1, sha256(after)) + after
.ledger/<v + 1>-<change id>.sql ← change + metadata (before/after checksums, folded-at)
delete changes/<change id>.sql
```

Why it works:

- **The database is the merge engine.** There's no ALTER-to-CREATE rewriter to get wrong. Whatever the
  engine accepts (renames, type changes, constraints, enums, views, functions, triggers) folds correctly.
- **Deterministic.** The renderer's output depends only on the catalog, so two people folding the same
  change get byte-identical files, and CI can verify that `schema.sql` is exactly what folding produces.
- **Proof, not hope.** Every reversible change has had its rollback executed at least once before it
  reaches a real database.

What the fold can't prove: the shadow database has no rows. `ALTER TABLE users ADD COLUMN email text NOT
NULL` folds fine, then fails on a table with data. A later `nautilith rehearse --env staging` runs pending
changes inside a transaction on a staging database and rolls it back, on engines with transactional DDL.

Pure data changes (backfills, fixes) leave `after == before`. They fold to nothing: `schema.sql` doesn't
change, and the ledger still carries them for environments that need to run them. Fresh databases never
run them, which is correct: there's no data to migrate.

Some rollbacks restore an equivalent schema rather than an identical one. The usual case is column order
after a drop and re-add. The check reports the exact difference, and `-- @down lossy` accepts it.

## The renderer

The renderer turns a live catalog into `schema.sql`. It's where most of the per-dialect work lives.

- **Complete and executable.** Loading the file into an empty database recreates the schema. Order
  respects dependencies: extensions, schemas, types, sequences, tables, views, functions, triggers,
  policies.
- **Grouped for reading.** Each table is followed by its comments, indexes and triggers, so everything
  about `users` is in one place.
- **Inline where possible.** Primary keys, unique and check constraints, defaults and foreign keys go
  inside `CREATE TABLE`. Tables are ordered so referenced tables come first. Only a real foreign-key cycle
  falls back to an `ALTER TABLE … ADD CONSTRAINT` at the end, with a comment saying why.
- **Stable.** Objects are sorted by name within their group. Columns keep their real (ordinal) order,
  because `SELECT *` and `INSERT` without a column list depend on it. Formatting is fixed. The shadow
  engine's version is pinned, because catalogs print defaults and expressions slightly differently
  across versions.
- **Comments survive.** `--` comments in change files can't survive regeneration. `COMMENT ON` does, and
  the renderer puts it next to the object it describes. It's the recommended way to document a schema,
  for people and for agents.
- **Honest.** Objects the renderer doesn't support yet are never silently dropped: `check` fails and
  names them.
- **Header.** Version, checksum, and one line saying the file is generated and how to change it.

## Applying to an environment

`nautilith apply --env <name>`:

1. Take a lock (`pg_advisory_lock` on Postgres, `GET_LOCK` on MySQL, a lock row on SQLite) so two deploys
   can't race.
2. Read `nautilith_history`, then:
   - **Empty database:** run `schema.sql` and record a `baseline` row at HEAD. No history is replayed,
     which makes CI and test databases fast.
   - **Behind HEAD:** run ledger entries from the database's version + 1 up to HEAD, in order. Each change
     gets its own transaction when the engine has transactional DDL (Postgres, SQLite) and the change
     isn't marked `@no-transaction`. MySQL commits DDL implicitly, so there we recommend one statement
     per change.
   - **`pending = true` environments** (meant for `local`) also run `db/changes/*`, recorded by change id
     with no version. When those changes are folded later, the id match means they don't run again.
3. Record every step: version, change id, direction, checksum, the up and down SQL, who ran it, and when.

Protected environments (anything without `pending = true`) only ever receive folded changes. Everything
that reaches them has a version, a checksum and, unless marked otherwise, a proven rollback.

`--plan` prints exactly what would run, and stops.

If an environment is behind the start of the ledger because entries were compacted, `apply` says so and
offers two ways out: `--restore-ledger` pulls the missing entries back out of git history, and `--rebuild`
drops and recreates a disposable database from `schema.sql`.

History table, as created on Postgres:

```sql
CREATE TABLE nautilith_history (
  seq         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,  -- order of execution in this database
  version     integer,                                          -- NULL for pending changes
  change_id   text        NOT NULL,
  direction   text        NOT NULL CHECK (direction IN ('up', 'down', 'baseline')),
  checksum    text        NOT NULL,
  up_sql      text,
  down_sql    text,
  applied_at  timestamptz NOT NULL DEFAULT now(),
  applied_by  text        NOT NULL,
  tool        text        NOT NULL                              -- nautilith version
);
```

## Rollback, revert and unfold

Three different operations that people all call "rollback":

| Command | Changes the database | Changes `schema.sql` | Use it when |
|---|---|---|---|
| `rollback --env prod --steps 1` (or `--to 41`) | yes: runs stored down SQL | no | Incident: get prod back now |
| `revert <change id or version>` | not until the next `apply` | yes: folds a new change that undoes the old one | The change was wrong; every environment rolls forward |
| `unfold` | no | yes: pops the last fold back into `changes/` | You folded too early and the change hasn't left `local` |

- `rollback` uses the down SQL stored in the database's own history, not the repo's, so it still works
  after the repo moves on or the ledger is compacted.
- After a rollback the environment is simply behind HEAD. `status` shows that, and the next `apply` rolls
  it forward again unless you `revert` first.
- Irreversible changes stop a rollback unless you pass `--force`, which records what was skipped.
- No tool can bring back dropped data. For destructive changes, the recommended pattern is two steps:
  stop using the column (or rename it), then drop it in a later change.
- `revert` reads the original change from the ledger, or from git history if it was compacted.

## Scoped changes, data changes and reference data

- `@env dev, test` limits a change to those environments: fixtures, test users, fake data.
- A scoped change must not change the schema. `fold` checks that `after == before` and refuses otherwise,
  because there is one golden source for all environments.
- Reference data that every environment needs (roles, countries, plan tiers) behaves like schema: fresh
  databases need it too, and they don't replay history. Planned: list those tables in config, and the
  fold also regenerates `db/reference.sql` from their rows. It's loaded right after `schema.sql`.

## Drift

`nautilith drift --env prod` renders the live database with the same renderer and compares it with the
schema at that environment's version:

- **At HEAD:** compare with `schema.sql`.
- **Behind HEAD:** compare checksums with the ledger entry's `after` checksum. For a readable diff, rebuild
  the old schema on a shadow by loading `schema.sql` and running the down SQL back to that version. The
  down SQL was verified at fold time, so the rebuild is exact.

If someone hot-fixed prod by hand, you see exactly what changed, and can capture it as a change.

## Compaction and the horizon

The ledger only exists for environments that are behind. `nautilith compact` connects to every environment
marked `track = true`, finds the horizon (the lowest version among them) and deletes ledger entries at or
below it. Git keeps them. `--through <version>` does the same without connecting, for pipelines that
already know the answer.

In steady state the repository holds `schema.sql`, an empty `changes/`, and a ledger holding only what's
in flight.

## Working in teams

Folding assigns versions, so two branches that fold at the same time both claim v42. It's the same problem
as Rails' `schema.rb` or Django's conflicting migrations, and the fix is one command:

- **Fold in the pull request (default).** The author runs `nautilith fold` before pushing. The PR shows the
  real schema diff plus the ledger entry. CI runs `nautilith check --clean`: no pending changes left, and
  `schema.sql` is exactly what folding produces.
- **Main moved?** `nautilith rebase` takes main's `schema.sql` and ledger and re-folds the branch's entries
  on top as v43, v44 and so on. Change ids don't change, so local databases that already ran them are
  unaffected.
- A git merge driver for `schema.sql` and the ledger can run the same logic automatically.
- Teams that prefer automation can fold in a merge queue instead.

## The agent contract

`nautilith agents` writes this section into `AGENTS.md` or `CLAUDE.md`, and keeps it up to date:

```markdown
## Database schema
- The complete, current schema is `db/schema.sql`. It's the only schema file you need to read.
- Never edit `db/schema.sql` by hand. It is generated.
- To change the schema, run `nautilith new <what-it-does>`, put the SQL under `-- @up` and the undo
  under `-- @down`, then run `nautilith check`.
- Document tables and columns with `COMMENT ON`, not `--` comments.
- Ignore `db/.ledger/`. It's history for the tool, not for you.
```

Also:

- `nautilith show users` prints one table's block (definition, indexes, triggers, comments).
- `--json` on every command, with stable field names.
- Hand edits to `schema.sql` fail `check`, and the error message explains the fix.
- The ledger is a dot-directory. ripgrep, which many agents use to search, skips hidden directories by
  default.

## Adopting it

- **New project:** `nautilith init`, then write changes.
- **Existing database:** `nautilith init --from-db "$DATABASE_URL"` renders it as `schema.sql` v1. Then
  `nautilith baseline --env prod` marks each environment as v1 after a drift check.
- **Existing migrations:** `nautilith init --from-migrations db/migration --format flyway` (or
  `liquibase-sql`, `plain`) replays them once into a shadow, in their tool's order, and folds the lot into
  v1. The old directory can go; git remembers it.

## Implementation plan

- **Go, one static binary.** It drops into any repository whatever the stack. Distribution: release
  binaries, Homebrew, `go install`, a Docker image, later thin npm and Maven/Gradle wrappers.
- **Dialect interface:** connect, lock, history-table DDL, a transactional-DDL flag, statement splitting,
  and `Render(catalog) → schema.sql`. PostgreSQL first, then SQLite, then MySQL/MariaDB.
- **Shadow providers:** a temporary database on a server you already have
  (`CREATE DATABASE nautilith_shadow_…`), Docker, or an embedded Postgres downloaded on first use. SQLite
  folds in memory.
- **Tests that matter:**
  - golden tests for the renderer;
  - fixpoint: `render(load(schema.sql)) == schema.sql`;
  - equivalence: for generated sequences of changes, folding and then loading gives the same catalog as
    replaying the changes in order;
  - round trip: `down(up(s)) == s` for every reversible change in the suite.

### Milestones

- **v0.1** PostgreSQL: `init`, `new`, `check`, `fold`, `apply`, `rollback`, `status`, `history`;
  temporary-database and Docker shadows; `nautilith.toml`.
- **v0.2** `drift`, `revert`, `unfold`, `rebase`, `compact`, `show`, `agents`, `--json`; SQLite.
- **v0.3** MySQL/MariaDB; `init --from-migrations`; reference data; `rehearse`.

### Later: capture

Agents will edit `schema.sql` directly no matter what the instructions say. `nautilith capture` could turn
that edit into a proposed change file: load HEAD and the edited file into two shadows, diff them with an
existing diff engine, and write the result into `changes/` for a person or agent to refine (renames,
backfills, safety). Declarative when that's convenient, imperative when it matters.

## Risks and open questions

- **Renderer coverage** is the long pole, and it's per dialect.
- **Engine versions:** a shadow on a different major version than prod can render subtly different
  catalogs. Pin it in config and warn on mismatch.
- **MySQL** has no transactional DDL, so a failed multi-statement change can leave partial state.
- **Oracle and SQL Server** are common in Liquibase shops. Worth supporting? Decide after v0.3.
- **Several schemas or databases** in one project: probably "stacks" in config.
