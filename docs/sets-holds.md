# Is this call inside that condition

[The pages](INDEX.md) - [what x3 is](../README.md)

## Is this call inside that condition

**What it catches:** the call that was supposed to be guarded and is not. A
page added to a menu without asking whether the user may see it; a write done
without the check above it. Both sides of the comparison read the same text, so
a set of names cannot tell them apart — the difference is **where** the call
stands, not what it says.

```json
{ "left":  { "from": "regex", "sources": ["ui/**"],
             "select": "items\.push\(\{ href: '([^']+)'" },
  "right": { "from": "regex", "sources": ["ui/**"],
             "select": "items\.push\(\{ href: '([^']+)'",
             "region": "if \(",
             "holds":  "can\('CORE', '[^']+'\)" },
  "compare": "left-subset-of-right" }
```

The left side is **every** page put on the menu. The right side is the pages
put on it from inside a condition that asks for a capability, and the rule says
the first set may not be larger. A page nobody asked about is named.

### What `holds` measures

`region` already means *the nearest boundary above the value*. `holds` asks
whether the text **between that boundary and the value** contains a pattern. So
the question is answered against the condition closest to the call, which is
the one that governs it.

It is not brace matching, and does not pretend to be: a language's block
structure needs a parser, and a set extractor reads text. What it reads is
exactly what a hand-written gate reads when it compares *"where was the last
`if (`"* with *"where was the last `can(`"* — the same measure, declared
instead of written.

### Two ways to read a region

| Written | The region | The value |
|---|---|---|
| `region` + `join` | **names** the container | carries the name, glued |
| `region` + `holds` | **asks** about the container | enters the set unchanged, or not at all |
| all three | asks **and** names | carries the name, if the container answers |

Beside `holds` the region's capture group is **optional**: a boundary with no
group has no name to give, and then `join` has nothing to bind. With a group,
the name is still carried.

### Outside is an answer, not an error

Without `holds`, a value above the first boundary stops the run with exit `2`:
it has no container, so there is no name to write next to it. With `holds` the
same value is simply **not in the set** — *"is it inside such a container?"*
has the answer *no*, and that answer is the point of the reading.

Nothing is lost quietly, because the other side of the comparison still carries
it: a value dropped here is reported as missing by name. And a `holds` that
nothing in the tree answers empties the side, which is `empty_scope` and red —
a misspelled pattern shouts rather than agreeing with everything.

<!-- x3-dist version=v0.149.0 capabilities=aa2b3d5359a52c0465529a4d78500da0ece5c1d342d9261b163f39c08cf09ce1 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
