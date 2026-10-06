# boxes reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.334.0`**

## boxes criteria fields

| `when` | Fields | Holds when |
|---|---|---|
| `file` | `path` | `path` exists |
| `gone` | `path` | `path` is **not** there |
| `pattern` | `sources`, `match`, `directives`, `reading` | `match` is found in the **code** under `sources` |
| `absent` | `sources`, `match`, `directives` | `match` is found **nowhere** under `sources` |
| `sql` | `dsnEnv`, `query`, `equals`, `driver`, `timeoutMs` | the query's first cell equals `equals` |
| `command` | `command`, `args`, `output`, `timeoutMs` | it exits `0` **and** its output meets `output` |
| `manual` | `by`, `seen`, `signed` | `signed` is written |

<!-- x3-dist version=v0.334.0 capabilities=05d32e130d9cedb16eec9c10f66b50c62f766394fce85acb384d8848e14d5f6d template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
