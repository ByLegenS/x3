# Traffic, written down

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 record`

**What it catches:** nothing on its own — it produces the source a later run is
compared against. Every other checker here reads the project **at rest**; this
one reads it **in motion**, standing in front of the running application as a
reverse proxy, passing traffic through untouched and writing down what went by.

```
x3 record [-config <file>] -listen <addr> -target <url> -ledger <file>
```

The application is not modified, not rebuilt and not linked against x3 — no
middleware, no import, no build tag — which makes the capability
language-independent from the first line. Exit codes: `0` on a clean shutdown,
`2` for usage, configuration or I/O errors. **There is no `1`**: recording is not
a gate.

### The ledger

One file per suite, JSON Lines, one interaction per line:

```json
{"n":1,"req":{"method":"POST","path":"/orders","headers":{"Content-Type":["application/json"]},"body":{"name":"a cup"}},"res":{"status":201,"body":{"id":"17","state":"created"}},"ms":34}
```

One line per interaction is deliberate: a behavior change then shows up as a
**diff a human can read** in review. `n` is the recorded order, and replay
follows it. Header and query values are kept as lists, because a header folded
into one string comes back different when it is sent again. A JSON body is
stored parsed, so a change inside it reads as one changed field; anything else is
text. `ms` is written for the reader — nothing compares it.

The ledger is a source file: committed, reviewed, and the thing that shrinks a
pile of hand-written behavior tests. Which is exactly why nothing secret may
reach it.

### Redaction happens before the disk

A secret that was never written cannot leak later, so redaction sits between
reading the response and writing the line. Three layers, the first two needing no
configuration:

1. **Credential headers**, always: `Authorization`, `Cookie`, `Set-Cookie`,
   `Proxy-Authorization`. What they carry is identity, not behavior.
2. **The `secrets` pattern set**, applied to every recorded value.
3. **The project's own field paths**, under `record.redact`.

```json
{ "record": { "redact": [
    { "path": "res.body.token", "reason": "session token" },
    { "path": "res.body.items.*.email", "reason": "personal data" } ] } }
```

A rule without a reason is refused. A path starts with `req` or `res`, then
`headers`, `query` or `body`; `*` means every element of an array or field of an
object. A path that reaches nothing is not an error — that field did not appear
in this run — but a path that **cannot mean anything** (`res.query.page`) is a
configuration error, because a misspelled rule would otherwise look exactly like
a rule with nothing to hide.

Hidden values are written as `"<redacted:reason>"`, so a reader can tell a masked
field from an absent one. **The proxy stays transparent**: the client receives
the application's answer exactly as sent. Redaction applies to what is written
down, never to what is served.

### What is recorded, and what is not

**Inbound HTTP**, by decision — in-process middleware would require the project
to import x3 and tie the capability to one language. Connection headers are
neither forwarded nor recorded, so replaying cannot send a proxy's own settings
on to the application. When the target does not answer the client is told so with
`502` and **no line is written**: that answer came from the proxy, not from the
application.

<!-- x3-dist version=v0.70.0 capabilities=d6bca49b1ee20fb16cf56855193fb72748bc6213792c4f4e81682cf9ef31d4b0 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
