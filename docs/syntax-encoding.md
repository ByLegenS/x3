# The file a shell decodes before it runs it

[The pages](INDEX.md) - [what x3 is](../README.md)

## The file the shell has to decode first

A parser reads a file by decoding it **itself**, and answers green because it
could. The shell that will actually run that file may decide differently and
read the same bytes as different letters. Measured: a gate script lost its UTF-8
byte order mark, Windows PowerShell 5.1 then read it in its own code page, the
file stopped parsing — **53 errors against 0 for the same file with the mark** —
and the gate that ran it went out green, having run no step at all, for seven
commits.

```json
{ "syntax": { "checks": [
    { "name": "every-gate-script-decodes-as-utf8",
      "sources": ["*.ps1", "tools/*.ps1"],
      "encoding": { "as": "utf-8", "bom": "beyond-ascii" } } ] } }
```

| Field | Meaning |
|---|---|
| `as` | the encoding the bytes must be valid in. `utf-8` is the only decoder built in, and the default |
| `bom` | `any` (default) does not ask · `required` wants the mark in every file · `forbidden` wants it in none · `beyond-ascii` wants it only where the file carries a byte outside ascii |

`beyond-ascii` is the contract a script language usually wants, and it says why:
a pure ascii file reads the same in every code page, so asking it for a mark
would be noise; the moment a byte outside ascii appears, a reader has to choose
an encoding, and without the mark it chooses its own. `forbidden` is the same
contract from the other side, for the readers that take the mark for content.

The two questions are **ordered**, not parallel: the bytes are checked against
the encoding first, and when they are not valid in it the mark is not asked
about at all — no mark makes invalid bytes decode, and one cause written as two
findings sends the reader looking for two repairs. Codes: `not_this_encoding`,
`byte_order_mark_missing`, `byte_order_mark_present`.

This engine runs the check over its own gate scripts. Two of the three were
missing the mark while carrying hundreds of bytes beyond ascii.

A check reads the files its `sources` match. Which files a check is **about**
can also be asked of what a file holds: [Which files the check is
about](syntax-subjects.md#which-files-the-check-is-about). And the tree a glob takes is often one
path too wide, while a comment explaining a rule is not a breach of it: [What a
check does not read](syntax-scope.md#what-a-syntax-check-does-not-read).

<!-- x3-dist version=v0.159.0 capabilities=7f149416d4d1ff326e5dfdfc03d02ef69f7f13faf78170206823bb1ba5536ceb template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
