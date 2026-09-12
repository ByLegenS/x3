# Credentials in the source

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 secrets`

**What it catches:** a credential in the source — the one mistake that cannot be
taken back, because it is in the history and the history is shared.

```
x3 secrets [-config <file>] [-out <file>] [-baseline <file>] [-update-baseline] [dir]
```

**No `secrets` section is not an error** — the builtin patterns apply. A leak
scan is not a check somebody skips by not configuring it.

```json
{ "secrets": { "sources": ["**"], "exclude": ["internal/secrets/testdata/**"],
    "patterns": [ { "name": "internal-service-token", "match": "svc_[0-9a-f]{32}" } ] } }
```

`builtin: false` turns the shipped patterns off, and then the project must write
its own — a scan with no patterns is refused rather than passed. `directives:
"skip"` keeps the engine's own `//x3:` lines out of the scan, for a tree whose
examples live inside production files and carry key-shaped values of their own —
[The engine's own lines](patterns.md#the-engines-own-lines).

### The report carries no secret

**The value found is masked**: the first four characters, then its length.

```
BLOCK config.go:7: secret_found
	a value matching "aws-access-key" is in the source
	value: AKIA... (20 characters)
```

A report is written to a log, pasted into a ticket, shared on a screen. A tool
that found a leak and then printed it would be a second leak.

### The shipped patterns

`private-key-block` (a PEM header of any kind), `aws-access-key`
(`AKIA`/`ASIA`), `google-api-key` (`AIza`), `slack-token` (`xox[baprs]-`),
`github-token` (`gh[pousr]_`), `json-web-token` (a three-part JWT), and
`url-with-password` (a password inside a connection string).

**Every pattern is a recognisable format, not an entropy score.** A value that
looks random is not thereby a secret, and a gate that reds every random-looking
string is switched off within a week. Binary files are skipped for the same
reason.

### Excluding what the pattern also catches

**What it catches:** the noise that gets a real gate switched off. A pattern for
an IPv4 address also matches a private range, an RFC 5737 documentation address,
a browser version like `126.0.0.0`, and a date written with dots. Measured on a
real production Go application, the bare patterns for its own credential shapes
returned **530 findings where 63 were real**.

```json
{ "name": "ipv4-address",
  "match": "(?:^|[^0-9.])((?:[0-9]{1,3}[.]){3}[0-9]{1,3})(?:[^0-9.]|$)",
  "ignore": [
    { "match": "^(?:0|10|127)[.]", "reason": "this host and the private range" },
    { "value": "255.255.255.255", "reason": "the broadcast address" } ] }
```

An entry writes **`match`** or **`value`**, never both, and always a **`reason`**.
It reads the **value that was found**, not the line — excluding by line would
hide every other value sharing it; `"on": "match"` hands it the whole match
instead, for when the surroundings decide rather than the value. An exclusion
that excluded nothing is `dead_ignore` and **red**.

**A capture group is the value.** A pattern usually matches the characters
around a value — a separator, a boundary — and those are not part of the secret,
so the mask covers the group and the exclusions read the group. Every match on a
line is examined, not only the first.

**Lookaround does not exist here.** Go's engine is RE2, so `(?<!...)` comes back
as `invalid named capture`, which sends the reader hunting for a group nobody
wrote. The engine names the real gap and points at what replaces it:

```
patterns[0]: match: error parsing regexp: invalid named capture: `(?<![0-9.])[0-9]{15,17}`;
"(?<!" is a lookaround and RE2 has none - write the exclusion as an ignore entry instead
```

Exclusions belong to the pattern that carries them, so the shipped patterns
cannot take one.

### Exemption, with a reason

```go
//x3:allow:secret: a documented example key, not a live credential
const example = "AKIAJ4EXAMPLEKEY9ABC"
```

The directive covers **its own line and the one below it**, and must be the
first thing on its line after the comment opener — `#`, `--`, `/*`, `*`,
`<!--`, `;` — so a sentence that merely *mentions* it is not one. That is not
theoretical: this package's own comments describe the directive, and the first
scanner read them as exemptions. An exemption nobody needed is `dead_exemption`
and red; the example above carries a real-shaped key on purpose, because written
with an ellipsis the exemption over it would cover nothing and this document
would fail the scan it describes.

<!-- x3-dist version=v0.127.0 capabilities=2801084864972251f16605de3f015cce95cb887510f91b29286bf29571907c96 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
