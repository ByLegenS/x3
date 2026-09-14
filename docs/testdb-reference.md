# testdb reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.210.0`**

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

<!-- x3-dist version=v0.210.0 capabilities=e3305c71f849b117968238c071cae00adc610cb5ba3b82e7db2a75aaae129e4b template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
