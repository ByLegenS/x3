# The parser that reads the name

[The pages](INDEX.md) - [what x3 is](../README.md)

## The parser that reads the name

**What it catches:** the check that runs, exits zero, and parsed nothing.

`run` hands the parser a **path**. Most parsers then decide what grammar to use
from the *name* — the extension, and the project files sitting next to it. When
they guess wrong they do not say so; some simply stop and exit zero. The gate
sees a parser that ran and a zero, which is exactly what a healthy check looks
like.

Measured, Node 24.15, the same bytes each time — an ES module carrying a real
syntax error:

| how it was asked | valid module | broken module |
|---|---|---|
| `node --check <path>` | **0** | **0** — nothing was checked |
| `node --check` fed the content | 1 — read as a script | 1 |
| `node --check --input-type=module <path>` | **1** — a false red | 1 |
| `node --check --input-type=module` fed the content | **0** | **1** |

Only the last row is right in both directions, and it is the only row that needs
the content rather than the path. So a check can hand the parser the file's bytes
instead:

```json
{ "name": "browser-modules-parse", "sources": ["ui/**/*.mjs"],
  "run": ["node", "--check", "--input-type=module"], "stdin": true }
```

With `stdin`, the file's bytes go to the parser's standard input and **the path
is not appended** — a parser that could still see the path would read it and
answer about the file on disk, and the bytes handed to it would go unmeasured.
The finding names the file either way: the engine knows which file it fed.

The default is unchanged. Without `stdin` the parser is handed the path exactly
as before and its standard input ends at once, because feeding every parser
blindly would break every parser that expects a path. And `stdin` on a check that
starts no process — `as` or `deny` — is a configuration error: there is nothing
to feed.

<!-- x3-dist version=v0.152.0 capabilities=99e1a9e5ef7349cef2de389de0c82b8654db18f948462add95ca7aead639768c template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
