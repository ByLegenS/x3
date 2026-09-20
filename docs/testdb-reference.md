# testdb reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.255.0`**

## testdb settings

| Field | Required | Meaning |
|---|---|---|
| `adminDsnEnv` | yes | **name** of the variable holding the maintenance DSN. Point it at a maintenance database, never at the template: a template with an open connection cannot be cloned |
| `driver` | no | defaults to `pgx`; an unregistered name is a configuration error (exit `2`) |
| `prefix` | no | defaults to `x3test_`, and it is the **authority boundary** — nothing outside it is listed or dropped, so an empty prefix is rejected |
| `template` | no | `name` + `from`: the setup runs **once** into a template and every database after that is a clone of it. Without it an empty database is created and the setup steps run every time |
| `dsnEnv` | no | the variable the new DSN is exported as; defaults to `X3_TESTDB_DSN` |
| `maxAgeMinutes` | no | age past which a leftover is stale; defaults to `120` |
| `setup` | no | the steps run after creation, **in order**; each one `command`, `args`, `env`, `timeoutMs` (§ [A ready database is more than one command](testdb-setup.md#a-ready-database-is-more-than-one-command)) |
| `runEnv` | no | the variables the **wrapped command** of `testdb run` is handed, on top of `dsnEnv`; same vocabulary as a step's `env` (§ [The variables the wrapped command is handed](testdb-run-env.md#the-variables-the-wrapped-command-is-handed)) |

<!-- x3-dist version=v0.255.0 capabilities=83aee35d143ce0c5a588733de19a5c5368735fe1b0101e6c869d606a6fd44a69 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
