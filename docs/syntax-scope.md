# What a check does not read

[The pages](INDEX.md) - [what x3 is](../README.md)

## A syntax check remembers each file

`x3 syntax` stores each check's verdict on each file under the file's content
digest, in the declared cache directory (`syntax.yaml`), salted by the `syntax`
section alone. One page changing re-measures that page only; editing a check
re-measures every file. What a check reads that the section does not spell is in
the key too: the values a `denyFrom` query returned this run, and for a parser
fed on stdin the parser binary itself (path, size, time).

```yaml
cache:
  dir: ~/.x3cache/myproject
```

```text
x3 syntax: cache 1 hit(s), 1 miss(es)
```

What it does not do: a parser that opens the file **by path** is never
remembered (it may read other files, so its verdict is not a function of these
bytes), and a run the parser did not answer (unreadable file, time ceiling) is
not stored. A remembered file still passes through the shared reader, so the
gate observes the same reads. `-no-cache` measures everything again.

`x3 arch` remembers the same way, one file at a time: a consistency rule's side
(`left`, `right`, `when.set`) reads each of its files into a set of values, and
that contribution is keyed by the side's whole spelling, the file's path and its
bytes. The comparison itself (difference, dead filters, on-disk lookups) is made
again on every run, so knowledge across files never comes from an old tree: when
only the right-hand file changes, the left-hand files answer from the record and
the missing value is still caught. The salt is the engine version alone, so an
overlay (`-with`) or another rule's edit does not drop the record. Measured on
the pilot: `x3 arch .` 5.9 s cold, 2.2 s with nothing changed. The engine's own
`arch per-rule cache control experiment` proves it: a changed key file is read
again and caught; a changed label file alone is read again and the comparison
is red; a changed pattern reads that side's files again while the other side
is recalled. `-cache <file>` and `-no-cache` work as in `syntax`.

The parse of each Go file (imports, exemption lines, declared names, examples,
literals, terms, comments, body digests) is remembered too, in a record of its
own beside the rule record (`arch-files@<version>.yaml`): keyed by the file's
path, its bytes and the run's reading request, salted by the command's code
identity only. So the overlay run (`-with`) parses nothing the plain run
already parsed, and a one-file edit re-parses one file. The record is the
command's **side record**: a release whose `arch` result code is unchanged
inherits it like the rule record (before, every release re-parsed the whole
tree, because the file name `arch-files` found no row in the result table),
and a file keeps only its current and previous bytes (two slots, not the ten
of the rule record; on a pilot the record had reached 81 MB). A Go file whose
**size and modification time** are those of the last run is not opened at all:
the record maps them to the digest of its bytes, and the parse comes from the
digest. Measured on a pilot: a run that recalled every rule still opened and
read all 4 741 Go files just to digest them, a third of its CPU time in file
system calls. The map is written only for a file modified more than two seconds
earlier (a file rewritten within the same clock tick can keep its time — git's
"racy" file), and size and time are taken *before* the read, so a file that
changes while it is read carries a new time next run. The same digests answer
the recall check of every rule (`filesSum`), so a recalled rule does not reopen
the files it read either. Control (`recalledByStat`): a file whose size or time
changed is read again; with the lock removed the old imports come back and two
examples are red. The text pass (literal `deps` and `vocabulary` rules over
non-Go files) keeps the same kind of record per text file: its verdict — each
rule's counters and the file's findings in the order they were written — keyed
by the file's size, time and owner and by the pass's rules (their full
spelling, term sets and owner sets, comment syntax). Before, every run reopened
every text file the rules read and ran every pattern again. Control
(`recalledText`): a changed rule set or a rewritten file is judged again.
Duplicate bodies are
compared by their SHA-256 digest and line count, not their text, so the record
stays small. `X3_CACHE_WHY=1` names every rule judged again and why (it reads
the settings, it reads a tree or a command, or no record holds for its
sources); the summary line says `N rule(s) recalled, M judged`.

A side that enters skipped directories (`hidden: true`) walks only as deep as
its `select` pattern can reach: `dirs:*` lists the root's entries and stands
each directory by its first file, instead of walking every file under it. A
pattern with `**`, a non-tree side or a rule with `exclude` still walks the
whole tree.

```text
$ X3_CACHE_WHY=1 x3 arch -config x3.yaml .
x3 arch: why every-x3-capability-runs-in-a-step: judged again - it reads the settings themselves (from: x3)
x3 arch: 154 rule(s) recalled, 3 judged
```

Measured on the pilot (2026-11-15, same machine, other gates running): a warm
`x3 arch .` 3.8-4.3 s → 2.9 s; one comment appended to a core file 5.3 s →
4.2 s per run (74 of 157 rules read that file's bytes and are judged again).
With the reading-bound record (v0.377.0) the same comment re-judges 6 of 157
rules (3 read the settings themselves, 3 read comments), 3.8 s per run against
3.2 s warm. Control experiments in `internal/arch/recall.go`: `codeSum` (a
trailing or end-of-line comment keeps the digest, changed code, a shifted line,
a new directive or a changed raw string move it), `filesSum` (a comment-only
change keeps the code-read sum and moves the byte-read sum), `readsText` (one
side that reads comments binds the whole rule to bytes) and `ruleInput` (a
comment-only change recalls a code rule and re-judges a `comments: checked`
rule; a consistency rule that names its file is not bound to Go files - with
`opens` replaced by the rule's own sources that example is red).
The walk for the root-directory rule had covered 197 156 files, 174 146 of them
inside worker checkouts. Control experiments: `parsed` (a changed import on the
same path is parsed again; with the bytes taken out of the record input the
old import comes back and the example is red) and `PathsEverywhereWithin`
(fixture `internal/source/testdata/everywhere`).
The measuring commands (`syntax`, `arch`, `lang`, `comments`, `secrets`, `scan`,
`case`) also keep one configuration reading for the whole command; on the pilot
re-reading it per question was 4.2 s of a 5.4 s `x3 syntax`.

## What a `syntax` check does not read

A source glob describes a **tree**, and a check's real subject is often a tree
*minus* a handful of paths: the four screens behind an administrator's door,
the ledger's own type, the file written to disk and never sent anywhere. Stars
cannot write a subtraction, so the scope had to be widened until it was wrong
and the gate turned noisy — and a noisy gate is a closed gate.

```json
{ "syntax": { "checks": [
    { "name": "no-margin-word-reaches-the-browser",
      "sources": ["app/**/*.go"],
      "exclude": ["**/*_test.go", "app/ledger/ledger.go"],
      "deny": "cost_micros", "reason": "the margin is ours, not the tenant's",
      "comments": "exempt" } ] } }
```

`exclude` removes paths the `sources` glob took, and its name and reading are
the ones an `arch` rule already carries. It is not the third escape hatch of
this check but the first of three, each on its own axis: **`exclude` drops a
file**, `ignore` drops a **line** the pattern caught, and `holds`/`lacks` drop
a file by what it **contains**.

### Prose is not a violation

`comments: "exempt"` stops the denied pattern from reading comment text. The
sentence that explains *why* a word is forbidden has to write that word:

```go
// cost_micros is ours; a tenant-facing endpoint never returns it.
```

Counting that as a violation rewards deleting a correct explanation, and it
does worse than that — it produces confident wrong diagnoses. Measured on a
real production Go application: of the findings a first port of one gate
raised, the ones left after the paths were excluded were **entirely** comments
explaining the rule, and reading them as code had already turned into *"we
found six violations"* when there were none.

Comments are **blanked, not deleted**, so line numbers stay where they are and
the finding names the real line. The default is `checked` — the reading every
configuration so far was measured under, and a default that changes quietly is
a rule nobody declared.

### The laws

- Both belong to the check they are written on, not to the section.
- `comments` needs `deny`. A parser reads the file **from disk**; a blanked
  copy never reaches it, so the declaration would measure nothing. The same is
  already true of `ignore` and of `directives`.
- `syntax` replaces the comment markers for an extension, and it needs
  `comments: "exempt"` beside it, or it changes nothing.
- An **unknown extension is read as all code**. Not knowing costs in one
  direction or the other, and this one is chosen: a check that reads too much
  shouts, a check that reads too little agrees with everything.
- An `exclude` that empties the whole set is `empty_scope` and red, the same
  as a `sources` glob that matches nothing.

### The one line a reason reaches

A denied pattern with one legitimate use is excused **on the line itself**: a `//x3:allow:syntax[:<check>]: <reason>` comment closes itself and the line under it — no reason and it doesn't parse (stays red); a `dead_exemption` (exit `2`) can't be frozen into a baseline.

Green: `//x3:allow:syntax: schema is gone` above `panic("missing")`. Red: the same `panic(...)` two lines later with no directive above it.

<!-- x3-dist version=v0.388.0 capabilities=0e20b5613ce4c8c3986b7359f0f251003201957dffdee335ca335b095d8538e0 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
