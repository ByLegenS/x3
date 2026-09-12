# Where the container ends

[The pages](INDEX.md) - [what x3 is](../README.md)

## Where the container ends

**What it catches:** the value that is *outside* a block and gets read as
though it were inside. A region runs to the next boundary, so a block that
opens only once in a file has no end at all — the next field, the next
declaration, the rest of the file are all still "inside" it.

```go
UI: UIDecl{
	Dir: "call", Screens: []string{
		"home", "leads",
	},
	Nav: NavDecl{Icon: "phone"},
},
```

A rule asking *"which screens does this manifest declare"* reads `home` and
`leads` — and `phone`, which is an icon. `until` closes the region:

```json
{ "left": { "from": "regex", "sources": ["apps/*/core/manifest.go"],
            "region": "Screens: \[\]string\{",
            "until":  "\n\t\t\t\}," } }
```

Measured on a real production Go application, on the two applications that
carry such a manifest, against the screen files on disk:

| The same rule | Subjects | Violations |
|---|---|---|
| `region` alone — the block never ends | 32 and 17 | 8 and 7 |
| `region` with `until` | 24 and 10 | **0 and 0** |

The eight and the seven are icon names, menu keys and search paths: every
quoted word written below the block.

### It does not count, and it says so when it would have to

The closing is the **first** match of `until` after the opening. That is what a
hand-written gate does — *find the opening, then find the next `}`* — and it is
all a text reading can honestly do; telling an inner closing from an outer one
needs a parser.

So the engine refuses the two shapes it cannot read, by name and with a line
number, rather than reading them wrongly:

| The tree | What happens |
|---|---|
| a second opening before the closing | exit `2`, naming both lines: the inner container cannot be told from the outer |
| an opening whose closing never comes | exit `2`, naming the opening: a container with no end |

The second one fires on real trees and is worth expecting. A list written on
one line (`Screens: []string{"a", "b"},`) never matches a closing anchored to
the start of a line; the answer is a pattern that matches both shapes (`\},`),
or a narrower `sources`.

### The laws

- `until` belongs to `from: "regex"` and **wants `region`** — a closing with
  nothing to close is a declaration that does nothing.
- It takes **no capture group**: a closing draws a boundary and names nothing.
- Beside `until` the region's capture group is **optional**, exactly as it is
  beside `holds`: with a group `join` still glues the name on, without one the
  region is a pure scope.
- A region begins **at its boundary**, so a value written on the opening line
  is inside it. `until` changes where a region *ends*, not where it begins.
- **Outside is an answer, not an error.** With no closing and no `holds`, a
  value above the first boundary stops the run, because it has no name to
  carry. With a closing it is simply not in the set — the container was
  declared and the value is not in it. Nothing is lost quietly: the other side
  of the comparison names it, and a closing that empties the side is
  `empty_scope` and red.
- `until` and `holds` may be written **together**: the container must both end
  where it is said to end and hold what it is said to hold.

<!-- x3-dist version=v0.158.0 capabilities=e84674ef730d4c01bd28143e856bf5a927aed61a6b4eead5472b98cb6d4baaa2 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
