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

### One result file per command, the code identity in every slot

A command writes one file per project, named after the command and nothing else
(`gate.yaml`, `case.yaml`, `arch.yaml`, `arch-files.yaml`, `boxes.yaml`,
`graph.yaml`). The identity of the code that produced a result is part of each
**slot's** salt, not of the file name: a binary whose code for that command
differs misses only those slots and writes its own beside them in the same file;
neither binary's records are deleted, and an unasked slot ages out after `stale`
runs or falls off the slot cap, oldest first. Two binaries (the installed one and
a measurement build) can write one file in turn: the write is atomic (temporary
file and rename, under a lock) and merges the file as it is on disk.

It used to be `gate@c<code digest>.yaml`: one changed declaration in the
command's reach opened a new file and every record in it (a gate's steps, the
runner's thousands of examples) started cold. The user, 2026-11-15, looking at
four `gate@c...` and four `boxes@c...` files written in one half hour: *"Why do
the files repeat each other? That is not a cache."* The first write of the new
file takes the tree's witness from the newest old-named sibling, then every
`*@*.yaml` file of the directory is deleted (a one-time move; a path given with
`-cache` that itself carries `@` sweeps nothing).

**A rerun names the cause.** The file also records the code digest its slots
were written with (`code`). When a command's records miss because that digest
moved, the rerun line says so instead of "no record": `RERUN internal/cache: 44
have no record: c was written with other code (b6bd... -> b2ca...)`; when only
the settings moved it says `other settings (the code ... is the same)`; a file
written before the field existed says `other settings or code (the old record
names no code)`. `X3_CACHE_WHY=1` gives the same clause for an `arch` rule and a
gate step. Control experiment: two release-tagged builds of one tree on one
scratch cache (`x3 case -only ./internal/cache -cache <dir>/c`): the first run is
plain "no record" (cold), the second build names the other code, the second
build again recalls everything; the `Cache.Moved` examples in
internal/cache/cache.go cover the four answers.

Measured on 2026-11-15 in this repository with four builds of one tree and one
scratch cache directory (`x3 case -only ./internal/cache,./internal/release`,
132 examples; `x3 gate -only "syntax,architecture,comments"`, 4 steps):

| build | what changed | case | gate |
|---|---|---|---|
| a | warm, then unchanged | 19,8 s cold, then 0,45 s (2 hits) | 3,6 s, then 1,1 s (4 hits) |
| b | a gate body (`say`) | 0,59 s, 2 hits | 1,06 s, 4 hits (gate code moved) |
| d | a settings body the runner does not reach | 0,57 s, 2 hits | 1,08 s, 4 hits (gate code moved) |
| c | the runner's `Config.Check` | 18,8 s, 132 rerun (control) | - |
| a | again, after b, c, d wrote | 0,45 s, 2 hits | 1,06 s, 4 hits |

The directory held one `case.yaml` and one `gate.yaml` throughout. Control
experiments: the `Open`, `Pure`, `kin` and `sweep` examples in
internal/cache/cache.go and `recordCode` in cmd/x3 (with the pure salt turned
off, or the gate's own code put back into its salt, one example each goes red).

## Files the engine reads back

Some of what a gate reads is not source but state the project keeps beside it:
the frozen baselines, a findings baseline, the open-work list, the cache. They
are read through one reader — the same one that reads `x3.yaml` — and it **drops
a leading byte order mark**. Windows tools write one while Go's JSON decoder
calls it an invalid character, so having the settings file forgive it and a
baseline refuse it meant two files written by the same editor behaved
differently, and the error named a character nobody typed.

<!-- x3-dist version=v0.385.0 capabilities=abe9643e4d415960fb50ec226ac7475863c6f1fd74981d0af316ba18bd369d71 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
