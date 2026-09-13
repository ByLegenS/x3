# testdb reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.170.0`**

## testdb settings

| Field | Required | Meaning |
|---|---|---|
| `adminDsnEnv` | yes | **name** of the variable holding the maintenance DSN. Point it at a maintenance database, never at the template: a template with an open connection cannot be cloned |
| `driver` | no | defaults to `pgx`; an unregistered name is a configuration error (exit `2`) |
| `prefix` | no | defaults to `x3test_`, and it is the **authority boundary** — nothing outside it is listed or dropped, so an empty prefix is rejected |
| `template` | no | without it an empty database is created and the setup steps do the work |
| `dsnEnv` | no | the variable the new DSN is exported as; defaults to `X3_TESTDB_DSN` |
| `maxAgeMinutes` | no | age past which a leftover is stale; defaults to `120` |
| `setup` | no | the steps run after creation, **in order**; each one `command`, `args`, `env`, `timeoutMs` (§ [A ready database is more than one command](testdb-setup.md#a-ready-database-is-more-than-one-command)) |
| `runEnv` | no | the variables the **wrapped command** of `testdb run` is handed, on top of `dsnEnv`; same vocabulary as a step's `env` (§ [The variables the wrapped command is handed](testdb-run-env.md#the-variables-the-wrapped-command-is-handed)) |

<!-- x3-dist version=v0.170.0 capabilities=a9b75718df0998f4fdf50ebad2bd46683894cb2c1a425b50125b3ce994e4cb5b template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
