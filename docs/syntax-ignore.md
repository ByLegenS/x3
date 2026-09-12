# The word another language owns

[The pages](INDEX.md) - [what x3 is](../README.md)

## The word another language owns

A denied word is often legitimate somewhere: `debug.Module` is the standard
library's type, `type="module"` is the browser's attribute, `go.mod` names a
file. The pattern cannot tell them apart — Go's regexp engine is RE2 and carries
no lookaround — so a [`deny`](syntax.md#x3-syntax) check may exclude those lines, in the
same grammar [`x3 secrets`](secrets.md#x3-secrets) uses. One shape for one question: a
second grammar would have to learn the same laws again.

```json
{ "name": "the-old-term-cannot-come-back", "sources": ["**/*.go"],
  "deny": "[Mm]odule", "reason": "the rename is done; the old term may not return",
  "ignore": [ { "match": "debug[.]Module|go[.]mod", "on": "line",
                "reason": "another language's own word, and not ours to rename" } ] }
```

Red without the exclusion, green with it — and the real violation
(`type Module struct{}`) stays red under both, which is the direction that says
the exclusion is not simply switching the check off.

| Field | Meaning |
|---|---|
| `match` | a second pattern; the line is excluded when it matches |
| `value` | an exact value instead of a pattern; exactly one of the two is written |
| `on` | what is read: `value` (default) is the text the denied pattern matched, `line` the whole line it sits on |
| `reason` | required — an exclusion nobody explained is never questioned again |

An exclusion that excluded nothing in the whole run is `dead_ignore`, red: a
stale exclusion is how a gate goes blind without saying so. On an `as` or `run`
check it is a configuration error, because a parser answers for the whole file
and there is no line to exclude.

<!-- x3-dist version=v0.145.0 capabilities=053cb5bda96cabedacc8ad6dc3d302d827ec2d9dcde5c77e648e817df0e093a8 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
