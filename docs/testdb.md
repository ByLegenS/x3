# A fresh database for this run

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 testdb`

**What it catches:** tests that serialise on one shared database, and tests that
pay for the migrations every time. `x3 testdb` gives a run its own PostgreSQL
database by **cloning a prepared template**, hands the command a DSN through the
environment, and drops the database when the command is done.

```
x3 testdb create [-config <file>]
x3 testdb drop   [-config <file>] (-name <database> | -stale)
x3 testdb list   [-config <file>] [-stale]
x3 testdb run    [-config <file>] [-keep] -- <command> [args...]
```

`create` prints its **DSN on stdout**, one line and nothing else, so a shell can
capture it. `run` is the shape most projects want: one process, a fresh
database, automatic cleanup; `-keep` leaves it behind, for the sweep to collect.

`run` returns the command's own exit code, with one exception: a runner that
announces it found nothing to run does not make a green. The sentences and the
command lines they are looked for in are the `live.blind` block `x3 guard`
documents, read from the same configuration file, and the reason is the same - `go test -run <pattern>` exits **0** when the pattern
matches nothing.

**Every `create` and `run` sweeps first.** The leak is not in the cleanup a run
does — it is where that cleanup never happens: a killed process (a cancelled
gate, a closed worker tree) says nothing more, and only the *next* run can see
what it left. So the next run drops everything past `maxAgeMinutes` and prints
how many. There is no switch: a sweep that can be turned off goes back to piling
up on the day it is. It costs one `pg_database` query, about 50 ms against the
~3 s a create takes.

**The cleanup belongs to the run, not to one exit path.** Once the database is
up, every way out carries it: the command passed, the command failed, the
command never started at all. Written per path, the obligation is forgotten by
the path added next — and the database that stays is invisible until someone
looks. The line the run prints is not read off the error either, because a run
can report a failure and still have closed cleanly:

| The run says | What happened |
|---|---|
| `created and dropped` | the database is gone, whatever the command's exit code was |
| `kept` | `-keep` was given, and `drop -stale` collects it later |
| `left behind` | the drop itself failed — this one really is still on the server |

`list` without `-stale` shows **every** database carrying the prefix, with its
age, so a leftover is visible immediately; `maxAgeMinutes` only decides which of
them `-stale` will collect.

### Two invariants

**Speed.** Measured on PostgreSQL 18.2 over loopback, five runs of a 40-table,
40-index schema: **238–263 ms** to clone the template against **275–292 ms** to
create an empty database and replay the same DDL; end to end `x3 testdb create`
took **409 ms**. The gap widens with the schema — cloning is one directory copy
whatever the migration count.

**Safety.** Every name this command touches has to be one x3 made, checked
before a byte reaches the server: the name matches `^[a-z_][a-z0-9_]{0,62}$`
(`CREATE DATABASE` takes no bound parameters, so this is the injection gate, not
a style rule), and it carries the configured `prefix` **and** the creation stamp
x3 writes into it. A name that fails either rule exits **1** — the gate refused
it — while a name that passes and then cannot be reached exits **2**. That
difference is what makes the gate observable from outside. The stamp is weighed
too: base36 accepts letters, so a hand-written `apptest_backup_v2` would read as
born in 1970 and look infinitely stale. A stamp outside **2025-01-01 … now +
24 h** is not a stamp, and the name is not ours.

### `testdb` in `x3.yaml`

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN",
    "template": { "name": "app_test_template", "from": ["migrations/**"] },
    "prefix": "apptest_", "dsnEnv": "APP_TEST_DSN", "maxAgeMinutes": 120,
    "setup": [{ "command": "./migrate", "args": ["up"], "timeoutMs": 60000 }] } }
```

See **testdb settings** in the [testdb reference](testdb-reference.md#testdb-settings).

The DSN handed to the command is the maintenance DSN with **only the database
name changed**, so credentials and options carry over; both PostgreSQL spellings
are understood. **The creation time is in the name** — PostgreSQL does not record
it — which is what lets `-stale` work on any server with no extra table and no
privileges, and it is also why a database x3 did not name has no age and is
never touched. **If a setup step fails, the database is dropped**: a half-built
schema is worse than none.

### The setup runs once, not once per database

Measured in a production application (2026-09-15): five gate steps each built
their own throwaway database, and **26.7 s of every one of them** was the same
migration applied again — around 130 s of work to produce one schema five times.

```json
{ "testdb": { "template": { "name": "app_test_template", "from": ["migrations/**"] } } }
```

With a template, the setup steps run **into the template**, once, and every
database after that is `CREATE DATABASE ... TEMPLATE` — a copy, not a migration.

**Freshness is in the name, not in a stamp table.** The template's real name
carries a digest of the files `from` names (`app_test_template_a1b2c3d4`). A
changed migration changes the name, so the next run builds a new template and the
old one ages out. A stamp table would put the question "is this template stale?"
*inside* the template — and answering it would mean connecting, while a database
with an open connection cannot be cloned.

⛔ **`from` is required.** A template with no declared sources cannot be known to
be fresh: the day a migration changes it would quietly hand out the old schema,
and every test would pass against the wrong database.

⛔ **Two steps racing to build it is normal, and it is settled in the database.**
The gate runs its steps in parallel, so several can see "no template" at once. A
PostgreSQL advisory lock keyed on the template name makes one build it while the
others wait — a file lock would only work on one machine, and nothing else in the
engine assumes there is only one. The build lands under a temporary name and is
renamed only when the setup finished: a half-built template standing under the
real name would be cloned as if it were ready.

### One database per item, not one per suite

```
x3 testdb fan -list "go list ./..." -- go test {item} -v -count=1
```

`fan` runs a **list command**, takes each output line as an item, and gives every
item its own database — items in parallel, output written one block per item.

Measured in a production application (2026-09-15): a suite of 111 packages shared
a single throwaway database. The packages ran in parallel, saw each other's rows,
and five of the ten red steps were that contamination. Putting the packages in a
line would have fixed it by making the suite single-file. Giving each one its own
database costs, with a template, about a second and a half — which is what makes
isolation affordable in the first place. The same suite: **98 s → 66 s**, and the
contamination is gone.

**The engine knows neither `go test` nor packages.** The items come from whatever
command is given, and `{item}` in the command line is replaced with each one; the
same mechanism serves another language's suite, or a sequence of migrations.
Where the placeholder is absent the item is appended, which is the common case.

⛔ **Blindness is asked of the run, not of the item.** A package with no test
files measures nothing, and that is normal; a *run* that measured nothing is not.
Asked per item, one empty package would turn the whole suite red — measured, it
did, on the first package of the 111. The count is still reported (`0 red,
65 measured nothing`), because the number itself says where the suite is not
looking.

⛔ **The default worker count is `GOMAXPROCS`, not the processor count.**
Measured: a suite that `fan` finished in 66 s on its own took **193 s** inside a
gate running 24 steps at the same time — two layers of parallelism again, 24
steps each fanning 32 items across 32 processors. A step's share is already
written in the gate's environment (`gate.env`, `{cores}`); `fan` reads it instead
of inventing a number, and a step that deserves more says so in its own `env`.
Outside a gate `GOMAXPROCS` is the processor count, so a suite run on its own
loses nothing.

**The template is built once, before the fan opens.** Otherwise the whole first
wave of workers finds no template, one builds it and the rest wait on the lock —
the parallelism would be spent waiting.

### Secrets and errors

The maintenance DSN is named by environment variable only, and its value is
stripped out of every error message before it is printed. The DSN of the
*created* database is deliberately printed by `create` — that is the point of the
subcommand — but under `run` it is never printed, only passed through the
environment.

<!-- x3-dist version=v0.272.0 capabilities=46500cb8ad080cc4b9df6cfb91418347928fab317327288705dd32ff203740a5 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
