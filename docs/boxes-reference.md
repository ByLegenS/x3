# boxes reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.185.0`**

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

## boxes criteria written in prose

| `when` | The rest of the line is read as |
|---|---|
| `file`, `gone` | a path |
| `pattern`, `absent` | a place, then the expression; the place matches the file **and** everything under it |
| `sql` | the query, `==`, the value it must give |
| `command` | arguments appended to `prefix`; `argument: "word"` demands exactly one |
| `manual` | who looks, the separator, what they must see |

<!-- x3-dist version=v0.185.0 capabilities=1ee338e5c8d1ec7040cc5325fc6cbca863abdaec69e021877be96075e8ad3a4a template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
