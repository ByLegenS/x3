# boxes reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.346.0`**

## boxes criteria fields

| `when` | Fields | Holds when |
|---|---|---|
| `file` | `path` | `path` exists |
| `gone` | `path` | `path` is **not** there |
| `pattern` | `sources`, `match`, `directives`, `reading` | `match` is found in the **code** under `sources` |
| `absent` | `sources`, `match`, `directives`, `reading` | `match` is found **nowhere** in the **code** under `sources` |
| `sql` | `dsnEnv`, `query`, `equals`, `driver`, `timeoutMs` | the query's first cell equals `equals` |
| `command` | `command`, `args`, `output`, `timeoutMs` | it exits `0` **and** its output meets `output` |
| `manual` | `by`, `seen`, `signed` | `signed` is written |

<!-- x3-dist version=v0.346.0 capabilities=ebff88b993f54d41ae92403bdf8a7a221356a8922c267d463ff950ebc20db649 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
