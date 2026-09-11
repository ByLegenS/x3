# Speed, the cache, and what a run leaves behind

[The pages](INDEX.md) - [what x3 is](../README.md)

## Speed

Files are independent, so reading and parsing them is done on every core and the
results are put back in file order — the report is byte-identical whatever the
core count. Measured on a real Go application of **1174 Go files** (2040 in
total), sixteen cores:

| Command | Before | After |
|---|---|---|
| `x3 lang` | 5.09 s | **0.24 s** |
| `x3 scan` | — | **0.12 s** |
| `x3 secrets` (all 2040 files) | — | **0.54 s** |

The same run produced the same bytes before and after, which is the part worth
checking: a parallel walk that reordered its findings would turn every later
comparison into noise.

### The incremental cache

A run can remember what it measured, keyed on the **content** of each file.

```json
{ "cache": { "dir": ".x3cache" } }
```

A directory, not a file: the file name comes from the command, because two
commands sharing one file would each delete the other's entries. Nothing is
written unless the section is there, and the directory belongs in `.gitignore`.
`x3 scan`, `x3 lang` and `x3 secrets` read it; `-cache <file>` points one run
elsewhere and `-no-cache` measures everything again.

An entry is used only when three things match: the **engine version**, a
**fingerprint of the whole configuration**, and the file's **content hash**.
Guessing which section affects which checker would be cheaper and would
eventually be wrong.

| Command | Full scan | Cached | Cache size |
|---|---|---|---|
| `x3 secrets` (2040 files) | 0.34 s | **0.12 s** | 0.5 MB |
| `x3 scan` (1174 Go files) | 0.13 s | **0.09 s** | 145 KB |
| `x3 lang` (1174 Go files) | 0.17 s | 0.18 s | 5.8 MB |

`lang` is the honest row: on that application the gate is red on thousands of
lines, and reading 5.8 MB of stored findings costs as much as parsing the files
again. **The cache pays off when a checker's output is much smaller than its
input** — which is why it is declared per project rather than switched on for
everybody. Every one of those runs produced a report byte-identical to the
uncached one.

## Files the engine reads back

Some of what a gate reads is not source but state the project keeps beside it:
the frozen baselines, a findings baseline, the open-work list, the cache. They
are read through one reader — the same one that reads `x3.json` — and it **drops
a leading byte order mark**. Windows tools write one while Go's JSON decoder
calls it an invalid character, so having the settings file forgive it and a
baseline refuse it meant two files written by the same editor behaved
differently, and the error named a character nobody typed.

<!-- x3-dist version=v0.103.0 capabilities=c085ea2f8775173860b02846284ab6863013fe369140640561fafd940510696a template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
