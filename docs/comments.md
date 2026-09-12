# The comment diet

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 comments`

**What it catches:** a comment block that grew past reading length — not
documentation, but a document in the wrong place.

```
x3 comments [-config <file>] [-out <file>] [-cache <file>] [-no-cache] [-baseline <file>] [-update-baseline] [dir]
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

**Two blocks in one file are two debts.** When a finding is frozen into a
[baseline](baseline.md#the-finding-baseline), its identity is the rule, the file **and the
block itself** — the comment text with indentation and line breaks normalised
away, so re-wrapping or re-indenting a frozen block keeps it frozen while a *new*
over-long block in the same file does not inherit its pardon. The identity used
to be the file alone, and that was a hole of exactly the shape this gate exists
to close: in a file the baseline already knew, a newly written over-long block
was absorbed in silence — the held count went up by one and nothing turned red,
while the identical fault in a file with no baseline entry was blocked. The
digest each finding carries is that identity, so a baseline line can be traced
back to the block that put it there.

**An older baseline has to be re-recorded, and the order matters.** Every
identity in a `comments` baseline written before this changed, so the old file
reads as debt the run no longer finds. **Delete the file, then record it once
more** — an update run against the old file sees every entry dead and no entry
held, and writes an empty baseline, which is a declaration of no debt at all. The
diff of the new file is worth reading: whatever the old identity was hiding
appears in it.

<!-- x3-dist version=v0.147.0 capabilities=ec7474ac3ae437e0e7b4c481e6241021d5e7b6a9a998088c3a6178cf6d8f00af template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
