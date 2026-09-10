# The recording, sent again

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 replay`

**What it catches:** a behavior change. `record` writes a run down; this sends it
again and compares. The behavior test is not a file somebody wrote — it is a
recording the machine took.

```
x3 replay [-config <file>] -target <url> -ledger <file> [-out <file>]
```

Three things are compared: the **status code**, the **headers named in
configuration**, and the **body, field by field**. Every field not named in a
rule is compared exactly — fail-closed, the direction every other gate points.

```
DIFF 2 res.body.state: value_differs
        recorded: created
        received: queued
```

### What is allowed to differ

A recording that compares timestamps fails on the second run.

```json
{ "replay": { "headers": ["Content-Type"], "normalize": [
    { "path": "res.body.created_at", "as": "time" },
    { "path": "res.body.id",         "as": "uuid" },
    { "path": "res.body.items.*.n",  "as": "number" },
    { "path": "res.headers.Date",    "as": "any" } ] } }
```

`time`, `uuid`, `number` and `any` are the four kinds. A normalized field is not
compared by value — but its **presence and kind still are**, so dropping it or
returning a string where a time was recorded is `kind_differs`. `headers`
defaults to `Content-Type`, because the shape of a body is behavior while `Date`
and `Content-Length` are not. This is the one place the comparison is opt-in
rather than fail-closed: a full header comparison is red on every run, and a gate
that is always red is a gate somebody switches off.

### Exemptions, and the dead ones

```json
{ "replay": { "ignore": [ { "path": "res.headers.X-Request-Id",
                            "reason": "per-request id, not behavior" } ] } }
```

Declared in `x3.json`, never inside the ledger, always with a reason — and an
exemption that silenced nothing is `dead_exemption` and **red**. `normalize`
rules are not held to this: a rule for a field that did not appear says nothing
about whether it is still needed.

### Carrying a session

Recording masks credentials, so a suite behind a login cannot simply be sent
again — the `Authorization` header on file says `<redacted:credential-header>`.
The way out is not to unmask the recording but to take a **fresh** value from the
run itself:

```json
{ "replay": { "carry": [ { "from": "res.body.token",
                           "into": "req.headers.Authorization",
                           "as": "Bearer {value}" } ] } }
```

Every answer is read for `from`, and whatever it yields is poured into `into` on
the requests that follow. A value may also come from the environment
(`"from": "env:X3_TOKEN"`). A value is carried **into a request only** — writing
into a response would mean editing the thing being compared. The template must
contain `{value}`. If the field is not in the recorded request at all it is
added, which is the common case: the header was masked away, and what replaces it
is a live value.

### Replaying in parallel

```json
{ "replay": { "workers": 8 } }
```

Against a target that waits 20 ms per call, 60 interactions: sequential 1.28 s,
8 workers **0.21 s**. Against a local application answering instantly the two are
the same, because what parallelism buys is the waiting, not the work. The report
is identical either way — findings are collected in interaction order.

Parallelism is declared, never assumed: only the project knows whether its
interactions are independent. **`workers` and `carry` together are a
configuration error**, refused before the run — a carried session needs the
recorded order, and going faster while getting a different answer is not going
faster.

### A database of its own

`testdb` and `replay` need no new feature to pair:

```
x3 testdb run -- ./start-app-and-replay.sh
```

`testdb run` clones a template database, exports its DSN, runs the command and
drops the database afterwards. What the engine deliberately does not do is start
the application itself — it does not know how, and a wrong guess would be worse
than the two lines of script.

### What a recording cannot send back

A masked value is not sent to the application: `<redacted:credential-header>` as
an `Authorization` header would come back `401`, and a reader would file an
identity error as a behavior change. Those values are counted instead:

```
x3 replay: 2 interaction(s) - 0 difference(s) - 0 exempted - 1 value(s) could not be sent back
```

The report carries no timestamp, and recorded and received values are truncated
and run through the `secrets` pattern set before they are printed — the report
that finds a leak must not become one.

### Calls the application makes

`x3 record` sees what the world asks of the application. This sees what the
application asks of the world — the rate service, the mail gateway, the payment
provider — and later answers those calls itself, so a replay does not reach
anybody outside.

```
x3 outbound record -listen :9101 -ledger out.jsonl
x3 outbound serve  -listen :9101 -ledger out.jsonl
```

This one is a **forward** proxy: the application is told about it the way every
HTTP client already understands, with `HTTP_PROXY`. In `record` mode the call
goes out and is written down with the same redaction. In `serve` mode nothing
goes out at all — the answer comes from the ledger, matched on method plus
scheme, host and path, in recorded order, so an application calling the same
endpoint twice gets the first answer first.

A call the ledger never saw is refused with `502` and counted, and the command
exits `1` when the count is above zero: during a replay a *new* outbound call is
new behavior, and a proxy that quietly let it through would hide exactly what the
replay is for. **Encrypted calls are refused, not tunnelled** — a `CONNECT` gets
`501`, because recording HTTPS would mean terminating TLS with a certificate of
x3's own, and believing you recorded a call you did not is worse than knowing you
did not.

<!-- x3-dist version=v0.71.0 capabilities=f3be4db814129e87baddf35ca71e7ba8ddc4be4aeb4640bfe38ac1ee57ad51e3 template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
