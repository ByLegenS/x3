# The comment diet

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 comments`

**What it catches:** a comment block that grew past reading length — not
documentation, but a document in the wrong place.

```
x3 comments [-config <file>] [-out <file>] [-cache <file>] [-no-cache] [dir]
```

**Block length is red.** Consecutive comment lines form a block; a blank line or
a line of code closes it.

```
BLOCK internal/source/glob.go:10: block_too_long
        a comment block runs 11 lines, the limit is 10; what needs this many
        lines belongs in a document
```

**The ratio only warns.** When a file carries more comment lines than code
lines, the run says so and stays green. The measure is *necessity*, not count,
and a gate that failed on a ratio would make people delete comments that were
needed. It speaks only above `ratioFloor`, because a three-line file with four
comment lines is not a finding.

```json
{ "comments": { "block": 10, "doc": 20, "ratio": "warn", "ratioFloor": 30,
                "docByExtension": { ".go": 20 }, "openers": { ".ts": "//" } } }
```

The **opening block** — everything before the first line of code — has its own
limit, twice the ordinary one by default: it is read once and describes the
whole file. "The first block" would have been the wrong rule, because the first
block *inside* the code is an ordinary block. Not every language earns the same
allowance, so `docByExtension` writes it per language.

Lines that talk to a tool rather than a reader are neither prose nor code —
`//go:...`, `// Deprecated:`, `//nolint`, `#!`, `// +build`, and x3's own
directives. They close a block and count for nothing.

Ten languages are built in (`.go`, `.js`, `.java`, `.cpp`, `.py`, `.ps1`,
`.yaml`, `.yml`, `.sql`, `.lua`); an extension that is not among them is skipped
rather than guessed at, and a project adds its own with `openers`.

An exemption carries a reason and a dead one is red. It may sit **above or
below** the block it covers: above is natural, but `gofmt` moves directives to
the end of a Go doc comment, and a rule accepting only one side would break
itself on the next format.

```go
//x3:allow:comments: the glob syntax table is the contract itself
```

<!-- x3-dist version=v0.70.0 capabilities=d6bca49b1ee20fb16cf56855193fb72748bc6213792c4f4e81682cf9ef31d4b0 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
