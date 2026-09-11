# Where a pattern binds

[The pages](INDEX.md) - [what x3 is](../README.md)

## How a pattern is read

**What it catches:** a pattern written about lines but read against a whole
file, which silently measures almost nothing.

Most settings match a regular expression against a **file's whole text**, and
there `^` and `$` bind to a **line**:

| The pattern | Holds when |
|---|---|
| `^func main` | some line starts with `func main` |
| `\Apackage ` | the **file** starts with `package ` |
| `(?-m)^package ` | the same thing, with the mode turned off |

Line mode is the default because the cost of the other reading is not symmetric:

| The measurement wants | A collapsed pattern gives | How it shows |
|---|---|---|
| something **found** | nothing found | loud: red over an absence |
| something **absent** | nothing found | **silent: green without measuring** |
| a **number** (`count`, `cap`) | a number near zero | **silent: debt reads as repaid** |

Nobody looks at green, so the silent rows decide the default. Nothing is lost:
`\A` and `\z` always mean the ends of the file. Where the answer depends on the
reading, the finding says so rather than leaving the number unexplained:

```
"docs/list.md" counts 3, above the cap of 1; a cap takes no debt;
^ and $ read a line here, not the whole file - read the other way it would count 1
```

**Four patterns are not read this way**, because their subject is one line or
one value, not a file: `syntax` `deny`, `secrets` `patterns[].match` and
`ignore[].match`, `boxes` `markdown.moved.match`, and `arch` `literal`
`pattern`. The last is the one to read twice — a `literal` rule looks for a
**name**, so `^name$` means "the whole literal is this name".

### Where a line ends

**What it catches:** the same tree judged two different ways on two machines,
because one checkout wrote `\r\n` and the other wrote `\n`.

`^` and `$` bind to a line, and in a regular expression a line ends at a
**newline** — a carriage return sitting before it is not part of the ending. A
tree checked out with CRLF would leave `foo$` unmatched, and the miss goes the
silent way again: a `pattern` criterion turns red over an absence, an `absent`
one turns green without measuring. The answer would belong to **whoever checked
the tree out**, not to what is in it. A thermometer, not a gate.

So every file the engine reads for a pattern arrives with one line ending. The
normalisation sits in the **reader**, not in the matcher: a match position and
the string and comment spans measured over the same text (`strings: "exempt"`)
have to live in one byte space, and normalising only the matcher would pull the
two apart.

| The tree | `^func Value\(\) \{\}$` |
|---|---|
| ends its lines with `\n` | holds |
| ends its lines with `\r\n` | holds |
| does not carry that line | red, both ways |

**The limit:** a pattern looking for the line ending itself (`\r\n`) no longer
matches anything. Which ending a repository stores is git's question, not this
engine's.

<!-- x3-dist version=v0.87.0 capabilities=cc1dd6426e362f7b7e21d2e8a71f9f04fd6e63386565b7d322747bc4f8025941 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
