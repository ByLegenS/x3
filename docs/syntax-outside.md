# Where the pattern does not look

[The pages](INDEX.md) - [what x3 is](../README.md)

## Where the pattern does not look

An [`ignore`](syntax-ignore.md#the-word-another-language-owns) reads the **text** a pattern
matched. Some rules are not about that text at all: a manual entry is allowed
when it sits inside a container that declares itself the fallback, and the line
that matched says nothing about where that container begins or ends. `outside`
names the container, and the check does not look inside it.

```json
{ "name": "an-identity-box-nobody-declared", "sources": ["ui/**/*.html"],
  "deny": "<(?:input|textarea)\\b[^>]*\\b(?:account_id|tenant_id)\\b[^>]*>",
  "reads": "file",
  "outside": { "open": "<(\\w+)\\b[^>]*data-manual-fallback[^>]*>",
               "close": "</${1}>", "nest": "<${1}\\b" },
  "reason": "an identity the system learns is not one to type by hand; a fallback is declared, not assumed" }
```

| Field | Meaning |
|---|---|
| `open` | the pattern a container begins with |
| `close` | the pattern it ends with; `${1}`..`${9}` carry `open`'s captures as literal text |
| `nest` | optional - what raises the depth, so an inner container does not close the outer one |

**Not an exclusion, and that is the point.** A dead `ignore` is red, because a
stale exclusion is how a gate goes blind. A container nobody has declared yet is
normal: the rule is written before the first fallback exists. So `outside`
claims nothing about the tree and carries no dead-declaration law - what it
carries is a count, `summary.outside`, saying how many matches fell inside one.

**An opening that never closes is only itself.** Taking the rest of the file
would turn one unclosed tag into a way to silence the whole check; it also lets
a self-contained element carry the marker itself
(`<input data-manual-fallback ...>`).

### What the pattern reads at once

`reads` is `line` by default: the pattern is searched line by line and a line is
at most one finding.

`"reads": "file"` searches the whole text. Every match is a finding (two tags on
one line are two violations) and the line reported is the one the match begins
on. The default does not change: every gate written so far was measured at one
finding per line.

### The match that crosses a line end

**A line-by-line search does not make a pattern a line pattern.** `\s` matches a
newline, so `Waive\(\s*""\s*\)` accepts a line break between the paren and the
empty reason. Split the call and the line reading never sees it - the rule was
there, the pattern was right, and the answer was green. **That green is the
harm:** a missing rule is distrusted, a silent one is believed. So a pattern
whose own text can reach past a line end (`\s`, `[\s\S]`, `(?s).`, a literal
`
`; `.` alone cannot) is searched over the whole file as well, and every match
that **contains** a newline is a finding. `summary.crossed` counts them: how
many a line-by-line reading could not structurally have seen. The counting is
unchanged - a crossing match whose first line already carries a finding is
dropped - and on a live tree of 9657 files under 35 checks the report before and
after was byte-identical.

⛔ **An anchor is a declaration, so an anchored pattern is never widened.** `^`
and `$` mean *line* end to a reading that splits lines and *text* end to one that
does not; running the pattern in both modes would silently change what its own
anchors say. The one that gets widened is the one that went narrow in silence.

### Inside a loop body, and counted per file

Two questions a forbidden pattern cannot answer alone: *is this call inside a
loop?* and *how many of these does the file make?* Both ride on `deny`.

```yaml
syntax:
  checks:
    - name: no-query-inside-a-loop
      sources: ['**/*.go']
      deny: '\.(Query|QueryRow|Exec)\('
      within: go-loop
      reason: a query inside a loop is one round trip per row; ask once for the whole set
    - name: a-screen-asks-each-address-once
      sources: ['ui/**/*.js']
      deny: 'api[(]''GET'', ''([^'']+)'''
      count: { max: 12, distinct: true }
      reason: every request on opening is a round trip; merge them and ask each address once
```

```
BLOCK store.go:9 (no-query-inside-a-loop): forbidden_pattern
BLOCK screen.js:5 (a-screen-asks-each-address-once): repeated_value
	"/api/orders" is asked again; the first is on line 3 - ask once and share the answer
BLOCK screen.js:6 (a-screen-asks-each-address-once): over_ceiling
	the file makes 4 of these, the ceiling is 3 - merge them into fewer
```

**`within: go-loop`** looks for the pattern only inside the body of a Go `for`
or `range`, nested ones included; the body is drawn by Go's own parser, not by
the text. The same call outside a loop is right, so a pattern alone would be red
on every query in the tree. A file that does not parse as Go is red: with no
body drawn, "nothing found" would be a lie. Bound: a query inside a function the
loop *calls* is not seen; what is measured is the call written in the body.

**`count`** turns the matches from findings into a tally of the file: `max` is
the ceiling (red on the match that crosses it), `distinct` reds the second time
the pattern's first capture group - the address - is written. One line is not
wrong on its own; the sum and the repeat are. Bound: the count is per file, so a
screen's imported component counts in its own file.

Both keep what `deny` has: an exempting line (`x3:allow:syntax:<check>` with its
why) one line above, the finding baseline (`-update-baseline`, shrinks only), and
the per-file cache. Neither enters the cache key of a check that does not write
it. **Control arms** (`internal/syntax/testdata/loopcall`, `.../requests`): the
same work as a loop of queries and as one query, the same screen asking a list
twice and once.

<!-- x3-dist version=v0.388.0 capabilities=0e20b5613ce4c8c3986b7359f0f251003201957dffdee335ca335b095d8538e0 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
