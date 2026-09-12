# Files no compiler reads

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 syntax`

**What it catches:** a file no compiler reads going out broken — a template, a
settings file, a script the browser loads at run time. The server still answers
200 and the screen is simply blank.

```
x3 syntax [-config <file>] [-out <file>] [-baseline <file>] [-update-baseline] [dir]
```

The gate does not guess which parser a file wants; a project declares it, and
each check is exactly one of three kinds:

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
that parses but means nothing in this format — and it needs a `reason`.

**A parser that is not installed is red.** A gate that quietly skips its check
on a machine without the tool reports green having verified nothing. A project
that genuinely wants it optional writes `"missing": "warn"`. A check whose
sources match nothing is `empty_scope`.

A `deny` check may also carry `directives: "skip"`, which keeps the engine's own
`//x3:` lines out of what the pattern reads — [The engine's own
lines](patterns.md#the-engines-own-lines). On `as` or `run` it is a configuration error:
those parsers read the file from disk and nothing would measure it.

<!-- x3-dist version=v0.127.0 capabilities=2801084864972251f16605de3f015cce95cb887510f91b29286bf29571907c96 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
