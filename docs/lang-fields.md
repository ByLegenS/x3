# The field name written inside a string

[The pages](INDEX.md) - [what x3 is](../README.md)

## The field name written inside a string

**What it catches:** a second language in the keys of a structured log line —
written inside string literals, and therefore invisible to every project that
has declared its strings free.

`strings: "any"` is the setting that keeps a language gate switched on at all:
the prose a user reads is not the gate's business. A **field name** is not
prose. It is what a query filters on, what a dashboard groups by, what an alert
matches — an identifier that happens to be spelled inside quotes. Reading every
string is unusable; reading none leaves the log line unwatched, and the log
line is where a second language leaks in most easily.

### Declaring where a field name sits

```json
{ "language": { "strings": "any", "fields": [
    { "call": "slog.Info", "at": 2, "every": 2 },
    { "call": "slog.String", "at": 1 } ] } }
```

| Field | Meaning |
|---|---|
| `call` | the call as it is written — `pkg.Func`, or a bare `logf`. A deeper selector (`a.b.C`) is refused rather than silently never matching |
| `at` | the position of the first argument that is a field name, counted from 1 |
| `every` | the stride onwards from `at`; left out, only `at` is read |

Both shapes in the wild are covered by the same two numbers.
`slog.Info(msg, k, v, k, v)` puts its keys at every other argument from the
second — `at: 2, every: 2` — and `slog.String(k, v)` names one, `at: 1`. One
entry per call: four of them cover `Info`, `Warn`, `Error` and `Debug`.

Only a **string literal** in those positions is read. A variable or an
expression is not the writer's spelling of a word, and the argument next to a
key is the value, which is data.

### Not a corner of `strings`

The reading does not ask what `strings` says. Declared field names are read
when strings are free and when they are held — and when they are held, the
literal is read **once**: it is reported as `field`, never a second time as
`string`. Two readings would put two records in a baseline, and fixing one word
would drop two lines in a single run.

### A call nobody makes

An entry is a claim about the tree — *this project logs like this* — and a
claim that does not hold is `empty_scope` and red, the same as a `sources`
pattern that matches nothing; a misspelled call name would otherwise be the
quietest way to switch the reading off. The claim is answered even on a run
that reads every file from the cache. A list written as `[]` is refused with
exit `2`: no declaration is not an empty declaration.

### What is not read

This reading is the Go parser's — a call has to be parsed before anyone can say
which argument is which. Files read as text, past the Go tree, have their
strings read whole or not at all.

<!-- x3-dist version=v0.141.0 capabilities=766f6a0c8fc2af6b7d6fc9bc993bd7568b52aff3738b034a1bdf30eb171edf54 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
