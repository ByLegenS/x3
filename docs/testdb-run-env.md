# The variables the wrapped command is handed

[The pages](INDEX.md) - [what x3 is](../README.md)

## The variables the wrapped command is handed

A setup step could always be handed the DSN under the application's own variable
name; the **wrapped command** could not. It got exactly one name, `dsnEnv`, and
if the code underneath it reads its connection string from another one — and a
real application does — then a fresh database is created, the code inside the
wrapper never sees it, and the suite runs against the **shared** database it
would have used anyway. Nothing is red. Every project that hit this wrote the
bridge by hand in its own script, and a project that did not know to look never
found out.

`runEnv` closes it, with the vocabulary a step's `env` already uses:

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN", "template": "app_test_template",
    "runEnv": { "APP_DATABASE_URL": "x3:dsn", "APP_ENV": "test" } } }
```

```
x3 testdb run -- go test ./...
```

The command — and everything it starts — now finds the fresh database under the
name its own code reads. The rules are the step's rules, checked at the same
door and with the same exit `2`: `"x3:dsn"` is the only reference the engine
answers, any other `x3:` value is a refused typo rather than text passed on, and
**`dsnEnv` itself may not be written** — that would point the command at another
database while the rest of the run talks to this one. Written empty
(`"runEnv": {}`) is exit `2` as well: a run that sets nothing does not write the
field.

It reaches further than it looks. `x3 testdb run -- x3 case ./internal/...`
gives the example runner, which has no environment seam of its own, the same
fresh database — the variables are inherited by every process under the wrapper.

### A database of its own, cloned inside the run

One fresh copy per run is shared by everything the wrapped command starts. An
example that needs a database **nobody else touches** — one that takes locks, or
rewrites the schema — has to clone its own, and it cannot clone the run copy: the
run copy has sessions, and PostgreSQL refuses to copy a database with another
session connected (SQLSTATE `55006`). The template has none, but its name comes
from its content (`<prefix>tpl_<digest>`), so the code under the wrapper could
not know it. Every project that needed this carried its own create-and-drop
mechanism.

`templateEnv` names the variable the template's name is handed in:

```json
{ "testdb": { "adminDsnEnv": "APP_ADMIN_DSN", "prefix": "apptest_",
    "template": { "name": "apptest_tpl", "from": ["migrations/**"] },
    "dsnEnv": "APP_TEST_DSN", "templateEnv": "APP_TEST_TEMPLATE" } }
```

```
x3 testdb run -- go test ./...
```

Inside, `CREATE DATABASE <own> TEMPLATE $APP_TEST_TEMPLATE` (over any connection
whose role may create databases, the run copy's DSN included) gives the example
a database with the schema already in it. The run **holds the template for as
long as the command runs**: the template's shared lock is taken before the
clone and released after the run copy is dropped, so a sweeper cannot drop the
template while the command is still cloning from it. Clones from the template
do not wait for one another.

Measured (2026-11-15, PostgreSQL 18, a session left open on the run copy):

| | Result |
|---|---|
| four parallel clones from `$templateEnv`, each dropped after | all four created and dropped |
| a clone from the run copy itself, with another session on it | `55006` |
| `templateEnv` not written | the variable is empty; the command has no template to clone |
| `templateEnv` written without `template` | exit `2`, named: there is no template name to hand over |

The door is the step's door: `templateEnv` may not be `dsnEnv`, may not also be
written in `runEnv` (two writers of one variable let the last one win), and a
name holding `=` is refused — all exit `2`.

To leave the sweeping to the engine as well, **name the clone the way the engine
names its own**: `x3 testdb create` clones the same template and prints the new
name under the prefix on stdout, and `x3 testdb drop -name <name>` drops it,
retrying through an autovacuum session and answering `42501` with a plain drop.
A clone left behind under that name is collected by `x3 testdb drop -stale`
like any other; a clone under a name of your own is swept by nothing.

<!-- x3-dist version=v0.380.0 capabilities=451c7c6ab59714f7ae80efc82ab6029b3a61b21aa0e436344a7fb54d479d80af template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
