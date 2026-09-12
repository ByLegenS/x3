# scan reference

[The reference index](../REFERENCE.md) - [the pages](INDEX.md) - [what x3 is](../README.md)

**Current version: `v0.161.0`**

## scan exit codes

| Code | Meaning |
|---|---|
| `0` | green — every directive is well formed and in a legal scope |
| `1` | red — at least one directive failed; each is printed with `file:line` |
| `2` | usage error, or the run could not complete |

## the directive dictionary

| Directive | Valid scopes | Requires |
|---|---|---|
| `//x3:rule:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:guard:<type>[:<subtype>...]` | `decl`, `file`, `pkg` | at least one sub-type |
| `//x3:case: <payload>` | `decl` only | a payload that parses: `in=(...) out=...` |
| `//x3:import: [<name> ]<path>` | `file` only | an import path, and a name that is a plain identifier if one is written |
| `//x3:tags: <tag>[ <tag>...]` | `file` only | one or more build tags; a constraint expression is not a tag |
| `//x3:type: <declaration>` | `file` only | a type declaration, or a method on a type the same file declares |
| `//x3:live` | `decl`, `file`, `pkg` | nothing |
| `//x3:skip:<reason>` | `decl`, `file`, `pkg` | a reason |
| `//x3:allow:<type>:<reason>` | `decl`, `file`, `pkg`, `line` | a type **and** a reason |

## scan error codes

| Code | Turns red when |
|---|---|
| `unknown_category` | the type is not in the dictionary — no verifier exists for it |
| `malformed` | a required sub-type or reason is missing, a doubled colon left an empty sub-type, or a `case` payload does not parse |
| `scope_not_allowed` | the type is known and well formed, but not legal in this scope |
| `unattached` | the type needs a declaration and there is none to bind to |

## the scan report fields

| Field | Notes |
|---|---|
| `version` | schema version; it goes up when a field changes meaning |
| `file`, `line` | relative to the scan root, always `/`-separated |
| `raw` | the directive line exactly as written |
| `category`, `segments`, `payload` | the parsed line; omitted when empty |
| `scope`, `target` | resolved binding; `target` only for `decl` |
| `status`, `code`, `message` | `ok` or `error`; the last two only on `error` |

## expectation fields

| Field | Meaning |
|---|---|
| `name` | required; the red names the expectation that was not met |
| `min` | required, at least 1 — an expectation of zero verifies nothing |
| `paths` | globs **relative to the configuration**; absent means the whole scan |
| `category` | `guard`, `rule`, `case`, ...; absent means any |
| `kind` | the first segment after the category; absent means any |

<!-- x3-dist version=v0.161.0 capabilities=dc3e9670ba9497349615241cc730ebb0d3de55e98954f4e755b74dcc5cc9ebff template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
