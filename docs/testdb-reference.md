# testdb reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.329.0`**

## testdb settings

| Field | Required | Meaning |
|---|---|---|
| `adminDsnEnv` | yes | **name** of the variable holding the maintenance DSN. Point it at a maintenance database, never at the template: a template with an open connection cannot be cloned |
| `driver` | no | defaults to `pgx`; an unregistered name is a configuration error (exit `2`) |
| `prefix` | no | defaults to `x3test_`, and it is the **authority boundary** — nothing outside it is listed or dropped, so an empty prefix is rejected |
| `template` | no | `name` + `from`: the setup runs **once** into a template and every database after that is a clone of it. Without it an empty database is created and the setup steps run every time |
| `template.keep` | no | how many templates of that family may stand; 1 by default, and the current one's slot is always one of them |
| `dsnEnv` | no | the variable the new DSN is exported as; defaults to `X3_TESTDB_DSN` |
| `maxAgeMinutes` | no | age past which a leftover is stale; defaults to `120` |
| `setup` | no | the steps run after creation, **in order**; each one `command`, `args`, `env`, `timeoutMs` (§ [A ready database is more than one command](testdb-setup.md#a-ready-database-is-more-than-one-command)) |
| `runEnv` | no | the variables the **wrapped command** of `testdb run` is handed, on top of `dsnEnv`; same vocabulary as a step's `env` (§ [The variables the wrapped command is handed](testdb-run-env.md#the-variables-the-wrapped-command-is-handed)) |

<!-- x3-dist version=v0.329.0 capabilities=ddd397f73e95d73df9375896c5faf5e2695af824c87b0800b22ff5994a4410ff template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
