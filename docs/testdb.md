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
database, automatic cleanup even when the command fails; `-keep` leaves it
behind, which is exactly what `drop -stale` later collects.

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
difference is what makes the gate observable from outside.

### `testdb` in `x3.json`

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN", "template": "app_test_template",
    "prefix": "apptest_", "dsnEnv": "APP_TEST_DSN", "maxAgeMinutes": 120,
    "migrate": { "command": "./migrate", "args": ["up"], "timeoutMs": 60000 } } }
```

See **testdb settings** in [REFERENCE.md](../REFERENCE.md#testdb-settings).

The DSN handed to the command is the maintenance DSN with **only the database
name changed**, so credentials and options carry over; both PostgreSQL spellings
are understood. **The creation time is in the name** — PostgreSQL does not record
it — which is what lets `-stale` work on any server with no extra table and no
privileges, and it is also why a database x3 did not name has no age and is
never touched. **If the migration hook fails, the database is dropped**: a
half-built schema is worse than none.

### Secrets and errors

The maintenance DSN is named by environment variable only, and its value is
stripped out of every error message before it is printed. The DSN of the
*created* database is deliberately printed by `create` — that is the point of the
subcommand — but under `run` it is never printed, only passed through the
environment.

<!-- x3-dist version=v0.70.0 capabilities=d6bca49b1ee20fb16cf56855193fb72748bc6213792c4f4e81682cf9ef31d4b0 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
