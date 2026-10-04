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
`x3 scan`, `x3 lang`, `x3 secrets` and `x3 comments` read it per file; `x3 test`
and `x3 case` read it for something bigger than a file (a unit and a package).
`-cache <file>` points one run elsewhere and `-no-cache` measures everything
again.

An entry is used only when three things match: the **engine version**, a
**fingerprint of the whole configuration**, and the file's **content hash**.
Guessing which section affects which checker would be cheaper and would
eventually be wrong. *Whole* includes every part the root declares, each under
its own name: a fingerprint taken from the root alone would not move when a rule
in a part changed, and the next run would answer that changed rule out of a warm
cache — a stale answer, given as a fresh one.

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

### Each engine version keeps its own results

Result files carry the engine version in their name (`gate@<engine version>.yaml`), so two versions run one after the other on one project no longer empty each other's cache. The first run of a new version takes the tree's witness (which path, which digest) from the newest sibling file; readers of the tree (`snapshot`, `boxes suspect.gone`) fall back to that sibling until the version has written. The siblings are ordered by modification time first and only the newest readable one is decoded (pilot: twenty 40-74 MB siblings were all decoded and a new version's first read of the tree took 25 s; `x3 boxes` with `suspect.gone` went from 43 s to 20 s). Whole-file readers (`snapshot`, `boxes suspect.gone`) take each record's body as a parsed tree and do not write it back to bytes first. A sibling not written for 30 days is deleted. The worker-count record (`gate-tune.yaml`) stays unversioned: it is the machine's. Measured by the `Open` / `newest` / `sweep` examples in internal/cache/cache.go.

## Files the engine reads back

Some of what a gate reads is not source but state the project keeps beside it:
the frozen baselines, a findings baseline, the open-work list, the cache. They
are read through one reader — the same one that reads `x3.yaml` — and it **drops
a leading byte order mark**. Windows tools write one while Go's JSON decoder
calls it an invalid character, so having the settings file forgive it and a
baseline refuse it meant two files written by the same editor behaved
differently, and the error named a character nobody typed.

<!-- x3-dist version=v0.327.0 capabilities=670633e0f379004ab4fec67e1c4d9694ec074772a08ccbe03e0d5e7be318aa8f template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
