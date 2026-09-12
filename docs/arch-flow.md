# Where a restricted value may appear

[The pages](INDEX.md) - [what x3 is](../README.md)

## `flow` — where a value may appear

**Catches:** a restricted handle escaping the one place allowed to hold it.

```json
{ "kind": "flow", "sources": ["internal/**/*.go"],
  "value": { "field": "Module.pool" }, "allow": ["receiver"] }
```

The places are `receiver`, `argument`, `result`, `comparison`, `assignment` and
`other` — the last so an unrecognised position is refused rather than skipped.

`comparison` is its own place because it is its own **verb**: asking whether a
handle is `nil` is not handing it over, and a rule that reads `if m.pool == nil`
as an escape turns every file that writes a guard into a violation. Only the
comparison operators count (`==`, `!=`, `<`, `<=`, `>`, `>=`); arithmetic on the
same value is still `other`, because computing with a handle is not a question
about it. Allowing it narrows nothing else: on a fixture where one method asks,
one hands the value to a callback and one puts it into a sum, `allow:
["receiver"]` reports **3** and `allow: ["receiver", "comparison"]` reports
**2** — the callback and the sum. **`allow` lists
what is permitted; everything else is red**, because a deny list would leave a
place added later silently free. **It reads names, not types**: the type in
`value.field` proves only that the field is declared somewhere the rule reads,
and if it is not the rule is `empty_scope` — a renamed field must not leave a
green rule behind.

<!-- x3-dist version=v0.121.0 capabilities=5306ec23e7cabdf80270578237935895351c102537d5fe39aeb8684a98f26cd2 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
