# The container a value sits in

[The pages](INDEX.md) - [what x3 is](../README.md)

## The container a value sits in

**What it catches:** a value that is right on its own and wrong where it stands
— a figure measured across the whole company, drawn on the card of a single
branch; a rule declared for one layer, written inside another layer's section.
The container is not a file, so `sources` cannot name it: it is a **region
inside** a file.

```json
{ "left": { "from": "regex", "sources": ["dash/**"],
            "region": "^\\[card (\\w+)\\]$",
            "select": "^metric = (\\w+)$",
            "join": ":" } }
```

`region` describes the **boundary** of a region and captures its **name**; every
value found inside that region enters the set written next to that name. The set
above reads

```text
branch:firm_health   branch:firm_revenue   firm:firm_total
```

and a rule can now hold it against a catalogue that says at which scope each
figure is measured. Without the container both sides speak bare names, the
question cannot be put, and the gate is **green on the wrong drawing** — that is
the direction the control experiment measures.

### What `region` is not

| Field | The question it answers |
|---|---|
| `sources` | which **files** are read |
| `parts` | which **pieces** one value is written in |
| `region` | which **container** a value was found inside |

`parts` with `join` already pairs a container with its value when a region holds
**one** — the block is captured whole and the two pieces are glued. It breaks on
the ordinary case: a region holding several values glues *all* of them into one
name (`branch:a:b`) that no set will ever carry, and `each` drops the container
instead. Measured before this was written; that gap is the whole reason for it.

### The laws

- `region` belongs to `from: "regex"`, like `parts`, `join` and `each`.
- It **wants `join`** — the name and the value are two pieces of one value, and
  a separator is never guessed.
- It wants **exactly one capture group**: the name of the region.
- Beside `region`, `join` and `each` may be written **together**: `join` binds
  the name to the value, `each` says the pieces are separate values. Without a
  region `join` has only the pieces to glue, so there the two still conflict.
- A value found **above the first boundary** stops the run with exit `2`, naming
  the file, the line and the value. Dropping it would shrink the set from a
  place nobody is looking at, and a shrunken set turns green quietly.

### Where a region ends, and what a comment does to it

A region runs to the next boundary — **not** to the end of the function it names
— so the doc comment of the *next* declaration sits inside it, and an inline
example written there reads as production code:

```go
func (m *App) ConnectionTypes() []Field { return []Field{{Key: "phone"}} }
//x3:case: given=(q := `SELECT v FROM t WHERE Key: "ghost"`) in=(q) out=…
```

`ConnectionTypes.ghost` enters the set and the rule goes red on a value no
running code names; in one real production Go application that surface is 2038
`//x3:` lines across 245 non-test Go files. The answer is `comments: "exempt"`,
the field the vocabulary rule has carried since v0.11.0 and defaults to.
Comments are blanked, not deleted, so positions hold and **region boundaries are
drawn on the same blanked text** — a heading inside a comment cannot open a
region either. The real fields are still read, so the green is measured rather
than empty; both directions run over one tree.

<!-- x3-dist version=v0.108.0 capabilities=6a7f8a4816aae0ad758d93568b8bc6011c91903ca5dd7c8ba29bdd601fedd3df template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
