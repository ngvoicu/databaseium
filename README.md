# Nautilith

**Write changes the way you always have. Read one schema that's always current.**

Nautilith is a small schema-migration tool in the spirit of Liquibase and Flyway, with one twist: every
change you write is *folded* into a single, always-current `schema.sql`, and then it gets out of the way.
No ever-growing folder of `V1__init.sql … V412__add_index.sql`. One file, the whole truth.

> **Status: design stage.** There is nothing to install yet. This README and
> [`docs/design.md`](docs/design.md) describe what we're building. The name is a proposal; the options
> and how they were vetted are in [`docs/naming.md`](docs/naming.md).

## The problem

Liquibase and Flyway record *how the database got here*. What people and AI agents need most of the time
is *what it looks like now*. With a classic migration chain, "now" exists only in a live database, or in
your head after mentally replaying hundreds of files.

An agent that reads `V1__init.sql` sees a `users` table without the fourteen columns added since, and
writes code against it. An agent that reads every migration spends its context on history instead of on
your task, and still has to reconstruct the result correctly.

> "To think is to forget differences, generalize, make abstractions."
> — Jorge Luis Borges, *Funes the Memorious*

## The idea: imperative in, declarative out

1. You (or your agent) write a normal change: `ALTER TABLE`, `CREATE TABLE`, a backfill, plus its rollback.
2. Nautilith runs it in a throwaway **shadow database** that holds the current schema. It proves the
   rollback restores the previous schema, then dumps the result back into `schema.sql`.
3. The change leaves the working tree. It moves to a **ledger** that only the tool reads, so environments
   that are behind can still catch up and databases can still roll back.
4. Once every environment has the change, its ledger entry is **compacted** away. Git keeps the archive.

The database engine does the merging. There's no hand-written SQL rewriting, so anything your database
understands folds correctly: renames, constraints, enums, views, functions, triggers.

## What it looks like

```text
db/
├── schema.sql     ← the golden source: complete, current, generated. The only file you read.
├── changes/       ← the inbox: changes waiting to be folded (usually empty on main)
│   └── 20260925T1412-add-user-email.sql
└── .ledger/       ← folded changes some environment still needs. Tool territory.
nautilith.toml     ← dialect, environments, shadow database
```

A change:

```sql
-- db/changes/20260925T1412-add-user-email.sql
-- @up
ALTER TABLE users ADD COLUMN email text;
CREATE UNIQUE INDEX users_email_key ON users (email);
COMMENT ON COLUMN users.email IS 'Login e-mail, lowercase. Unique when present.';

-- @down
DROP INDEX users_email_key;
ALTER TABLE users DROP COLUMN email;
```

After `nautilith fold`, the change file is gone and `schema.sql` reads:

```sql
-- nautilith · v42 · sha256:3f9a…c1 · generated: change it with `nautilith new`, never by hand

CREATE TABLE users (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name        text NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now(),
  email       text
);
COMMENT ON COLUMN users.email IS 'Login e-mail, lowercase. Unique when present.';
CREATE UNIQUE INDEX users_email_key ON users (email);
```

No `ALTER` anywhere. The table reads as if it had always been this way.

## Commands

| Command | What it does |
|---|---|
| `nautilith new <name>` | Scaffold a change in `db/changes/` |
| `nautilith check` | Fold pending changes in the shadow database, prove their rollbacks, print the resulting schema diff. For CI and agents. |
| `nautilith fold` | `check`, then rewrite `schema.sql` and move the changes into the ledger |
| `nautilith apply --env prod` | Bring an environment to the current version. An empty database loads `schema.sql` directly, with no replay. |
| `nautilith rollback --env prod --steps 1` | Undo the last change in that database, using the down script it stored when the change was applied |
| `nautilith revert <change>` | Fold a new change that undoes an old one, so every environment rolls forward |
| `nautilith status` · `history` · `drift --env prod` | Where each environment is, what ran there, and whether someone changed it by hand |
| `nautilith compact` | Drop ledger entries that every environment already has |
| `nautilith init --from-migrations db/migration` | Adopt it: replay an existing Flyway, Liquibase or plain-SQL history once and fold it into `schema.sql` v1 |

## Environments and rollbacks

- Every database records what ran in a `nautilith_history` table, including the SQL to undo it, so it can
  roll itself back even after the repo has moved on.
- Environments live in `nautilith.toml`. `local` may apply unfolded changes while you iterate; protected
  environments like `prod` only ever receive folded, versioned changes, and can require confirmation.
- A change can be scoped to some environments (`-- @env dev, test`) for fixtures and seeds. Scoped
  changes may not change the schema: there is one golden source for every environment.
- Folding is separate from deploying, the way committing is separate from shipping. `schema.sql` is the
  intended state; the ledger carries each environment from where it is to there.

## Built for AI agents

- One file to read: deterministic, sorted, each table grouped with its indexes, triggers and comments.
- The *why* lives in `COMMENT ON`, which survives folding and sits right next to the column it explains.
- `nautilith show users` prints a single table for small context windows. Every command has `--json`.
- `nautilith agents` writes the `AGENTS.md` / `CLAUDE.md` section: read `schema.sql`, add changes with
  `nautilith new`, never edit `schema.sql`, ignore `.ledger/`.
- `check` fails loudly, and says what to do instead, if `schema.sql` was edited by hand.

## How it compares

| Tool | Approach | Difference |
|---|---|---|
| Liquibase, Flyway | Ordered changesets, a history table, rollbacks, per-environment targeting | The chain only grows; the current schema exists only in a live database |
| Rails `schema.rb` / `structure.sql`, dbmate | A schema dump regenerated next to the migrations; new databases can load it | The closest in spirit. But the migrations stay in the tree until someone deletes them by hand, and rollbacks aren't verified. |
| Django `squashmigrations` | Squash a range of migrations on demand | Manual and Django-only; the result is still a chain |
| Atlas, Skeema, sqldef, Prisma, Supabase declarative schemas | Edit the desired state; the tool generates the diff | A diff can't reliably tell a rename from a drop-and-add, or express a backfill or a multi-step zero-downtime change. Nautilith keeps you in control of the change and keeps the desired state for you. |
| graphile-migrate | Iterate on `current.sql`, then commit it | The committed history is still a chain |

## Why "Nautilith"

The chambered nautilus lives only in the newest, largest chamber of its shell. When it outgrows a chamber
it seals it behind a wall and moves forward. The shell stays one piece, and a thin tube, the siphuncle,
still runs through every sealed chamber.

Your application and your agent live in the newest chamber: `schema.sql`. Old changes are sealed in the
ledger, out of sight but still connected, so a lagging environment can catch up and a database can still
roll back. *Lithos* is Greek for stone: the history is set in stone, the schema stays alive.

Jacob Bernoulli was so taken with the logarithmic spiral, the curve of the nautilus shell, that he asked
for it on his tombstone with the words *Eadem mutata resurgo*: "though changed, I rise again the same."
(The mason carved the wrong spiral. The motto survived.)

## Roadmap

- **v0.1** PostgreSQL: `init`, `new`, `check`, `fold`, `apply`, `rollback`, `status`, `history`
- **v0.2** `drift`, `revert`, `unfold`, `rebase`, `compact`, `show`, `agents`, `--json`; SQLite
- **v0.3** MySQL/MariaDB; `init --from-migrations`; reference data; rehearsals against staging

Planned as a single static binary (Go) that you can drop into any repository, whatever its stack.
