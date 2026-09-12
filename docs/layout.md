# Where a directive may sit

[The pages](INDEX.md) - [what x3 is](../README.md)

## Directive placement

**What it catches:** a `//x3:` line written where the Go formatter would not
leave it. The directive works, the scan is green, and the next formatting run
quietly rewrites the file — so the debt grows one migration at a time, until a
formatting step can no longer be added to the gate at all.

The formatter treats any `//[a-z0-9]+:[a-z0-9]` line as a **directive**: it
lifts every such line out of a doc comment, prints the prose, and writes the
directives back **at the end**, separated from the text by one blank `//` line.
Every `//x3:` line matches that shape, so this covers `case`, `rule`, `guard`,
`live`, `skip` and `allow` alike — not one of them in particular.

```go
// Add returns the sum.
//x3:case: in=(1, 2) out=3
func Add(a, b int) int { return a + b }   // red: layout_no_separator

// Add returns the sum.
//
//x3:case: in=(1, 2) out=3
func Add(a, b int) int { return a + b }   // green
```

### What a run says

| Code | What it means |
|---|---|
| `layout_no_separator` | text stands directly above the directive; one blank `//` line belongs between them |
| `layout_not_last` | text — or a blank `//` — stands **below** a directive; the formatter gathers directives at the end |
| `layout_extra_separator` | more than one blank `//` above the directive block; the formatter keeps exactly one |

```
WARN  layout_no_separator wallet.go:41: the directive block starts on line 41
	with no blank "//" line above it; add one between line 40 and line 41
	found: //x3:case: in=(1, 2) out=3
x3 scan: 112 file(s) - 48 directive(s) - 0 red - 1 layout warning(s) - 0 layout block(s)
```

A finding is reported **once per doc comment**, on the first directive in it,
because a doc comment is the unit the formatter rewrites.

### Where the rule applies, and where it does not

The formatter only reformats a comment group that begins in **column 1** and is
followed immediately by the next token — a doc comment on a top-level
declaration, or on the `package` clause. Everywhere else placement is free, and
the engine stays silent:

- an indented comment: inside a function body, a struct, a parenthesized `var`
- a comment held away from the declaration by a blank line
- a `/* */` block comment — the formatter lifts no directive out of one
- a doc comment on an **`import`** declaration, which the formatter leaves alone

A group holding **only** directives needs no separator and is never a finding.

### Settings

```json
{ "scan": { "layout": "block" } }
```

`warn` is the default and keeps the run green; `block` makes the finding red.
The default is deliberate: a formatting rule that turned every existing tree red
on upgrade would be hostile, and this engine cannot know what debt a tree
already carries. Adopt it by cleaning the tree once, then writing `block`. Any
other value is a configuration error, exit `2` — a policy that silently falls
back to the default leaves the writer believing a rule is in force.

### The engine does not run the formatter

The verdict comes from x3's own parser, never from an external tool. A check
that shells out reports green on every machine where the tool is missing, which
is the silent-green class this engine exists to catch. The engine also **writes
no file**: it says the line is going to move, it does not move it.

<!-- x3-dist version=v0.124.0 capabilities=2a78d3c8bbcbbd5a748b56e37baa25f6bc5b5c75586cf4c71127501f6048d067 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
