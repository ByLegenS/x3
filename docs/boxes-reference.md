# boxes reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.138.0`**

## boxes criteria fields

| `when` | Fields | Holds when |
|---|---|---|
| `file` | `path` | `path` exists |
| `pattern` | `sources`, `match`, `directives` | `match` is found under `sources` |
| `absent` | `sources`, `match`, `directives` | `match` is found **nowhere** under `sources` |
| `sql` | `dsnEnv`, `query`, `equals`, `driver`, `timeoutMs` | the query's first cell equals `equals` |
| `command` | `command`, `args`, `output`, `timeoutMs` | it exits `0` **and** its output meets `output` |
| `manual` | `by`, `seen`, `signed` | `signed` is written |

## boxes criteria written in prose

| `when` | The rest of the line is read as |
|---|---|
| `file` | a path |
| `pattern`, `absent` | a place, then the expression; the place matches the file **and** everything under it |
| `sql` | the query, `==`, the value it must give |
| `command` | arguments appended to `prefix` |
| `manual` | who looks, the separator, what they must see |

<!-- x3-dist version=v0.138.0 capabilities=f63b177d435cdd61e34235067c446339025b12c519a1cb36f46e1e0052e6dd2c template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
