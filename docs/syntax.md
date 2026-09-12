# Files no compiler reads

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 syntax`

**What it catches:** a file no compiler reads going out broken — a template, a
settings file, a script the browser loads at run time. The server still answers
200 and the screen is simply blank.

```
x3 syntax [-config <file>] [-out <file>] [-check <names>] [-baseline <file>] [-update-baseline] [dir]
```

The gate does not guess which parser a file wants; a project declares it, and
each check is exactly one of four kinds:

```json
{ "syntax": { "checks": [
    { "name": "every-settings-file-parses", "sources": ["**/*.json"], "as": "json" },
    { "name": "browser-scripts-parse", "sources": ["ui/**/*.js"], "run": ["node", "--check"] },
    { "name": "no-escaped-quote-in-an-attribute", "sources": ["ui/**/*.html"],
      "deny": "=\"[^\"]*\\\\'",
      "reason": "a backslash escape inside an attribute is not valid here" } ] } }
```

`as` names a parser the engine carries — `json` is the only one, because a
format half-understood is worse than one not understood at all. `run` names an
external parser: the path is appended, and a non-zero exit is a finding carrying
the parser's own first line. `deny` is the other half of the same problem — text
that parses but means nothing in this format — and it needs a `reason`. The
fourth, `encoding`, asks a question no parser answers: [The file the shell has
to decode first](syntax-encoding.md#the-file-the-shell-has-to-decode-first).

Where the parser is a one-line expression rather than a program that takes a
path, the arguments may name the file themselves with `{file}`; the path is then
substituted rather than appended, because a second path at the end would become
an argument of that expression. This is what lets a script parser be declared
without a project writing a wrapper script of its own.

An outside parser handed a path may decide what to read from the name rather
than the bytes, and pass a file it never checked: [The parser that reads the
name](syntax-content.md#the-parser-that-reads-the-name).

**A parser that is not installed is red.** A gate that quietly skips its check
on a machine without the tool reports green having verified nothing. A project
that genuinely wants it optional writes `"missing": "warn"`. A check whose
sources match nothing is `empty_scope`.

`-check` runs only the checks named, comma separated. It exists for the
experiment that proves one check reads what its author thinks: such a test
builds a tiny tree where every *other* check matches nothing, and the run would
be `empty_scope` red for reasons the experiment is not about. Narrowing does not
blind - the named check still measures, the report carries `selected`, and a
name no check carries is a configuration error rather than a silent empty run.

A `deny` check may also carry `directives: "skip"`, which keeps the engine's own
`//x3:` lines out of what the pattern reads — [The engine's own
lines](patterns.md#the-engines-own-lines). On `as` or `run` it is a configuration error:
those parsers read the file from disk and nothing would measure it.

A denied word is often legitimate somewhere else in the same tree — another
language's own keyword or type. Those lines are excluded rather than denied:
[The word another language owns](syntax-ignore.md#the-word-another-language-owns).

<!-- x3-dist version=v0.159.0 capabilities=7f149416d4d1ff326e5dfdfc03d02ef69f7f13faf78170206823bb1ba5536ceb template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
