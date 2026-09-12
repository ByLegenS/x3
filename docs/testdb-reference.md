# testdb reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.139.1`**

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

<!-- x3-dist version=v0.139.1 capabilities=57752bf7cc6540dd7c5294a3fe1382b06d87295e2a0ddc13b7e6cc4b27eb6c02 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
