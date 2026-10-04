# Changes that must not travel alone

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 docs`

**What it catches:** a change that travelled alone — code without its
documentation, a migration without its release note.

```
x3 docs [-config <file>] [-out <file>] [-scope auto|working|head] [-changed <files>] [-reason <text>] [dir]
```

```json
{ "docs": { "rules": [
    { "name": "code-changes-carry-documentation",
      "when": ["internal/**", "cmd/**"], "then": ["docs/**"] } ] } }
```

That is this repository's own section, and its own gate runs this command
against itself on every `check.ps1`.

### What counts as changed

| `-scope` | Reads |
|---|---|
| `auto` (default) | the working tree when it is dirty, the last commit when it is clean |
| `working` | `git status`, untracked files included; a rename counts as its new name |
| `head` | the files in `HEAD`, with the commit body as the place a reason may live |

The default is not a convenience: checking the last commit while the tree is
dirty would count documentation that has not been written yet. **A directory
that is not a repository is red**, not green.

`-changed a.go,b.sql` says the list outright and **git is not asked at all**;
the report's scope reads `given`, so nobody mistakes it for a diff. What this
check measures is what a change must bring with it — where the list of changed
files came from is the caller's business, not the rule's.

It exists because the rule could not otherwise be experimented on. Its control
experiment has to produce a change, and producing one meant `git init` plus a
commit inside a fixture tree — a repository inside a repository. Measured in the
pilot project: the experiment stayed in a shell script for months for exactly
this reason, and when that project's settings moved to YAML the script stopped
parsing them, so the rule ran with no experiment behind it at all. With
`-changed`, the experiment is five lines of gate settings and needs no tree.

### Exemption, with a reason

```
docs: none (code-changes-carry-documentation) - wording of one stderr line; the capabilities document does not quote it
```

The marker **names the rule it excuses**: `docs: none (<rule>)` unless the rule
says otherwise (`exempt`). In `head` scope it lives in the commit body, in
`working` scope it is passed with `-reason`. **The marker alone is red** —
`exemption_without_reason` is a separate code from the missing change, because an
exemption nobody had to justify becomes the only path within a month. The reason
is read to the end of **that same line** and no further, so both stderr lines end
in `followed by a reason ON THE SAME LINE`: the wording that stopped at "followed
by a reason" cost two turns in a row, each to a reader who wrote it underneath.

**In `snapshot` scope** (the default, and what a gate step runs) **the branch is
judged whole**: the unit is what gets merged, not each commit. Every file the
commits of `merge-base(main, HEAD)..HEAD` touched — or `HEAD^1..HEAD` when HEAD
sits on `main` (a `--no-ff` merge brings its branch with it) — joins the
uncommitted files and the snapshot in **one** change. Code in it is answered by
a `then` change anywhere in it, or by the marker with a reason in the body of
**any** commit of the range (or in `-reason`). A marker with no reason is red
only when no other body carries a reason. Without git or a range the snapshot
alone is judged, as before. The report's `commits` field and a stderr line name
the range read, and each finding names it:

```
BLOCK doc-follows-code: missing_counterpart_change
	branch 5d0e2b1..HEAD: 1 file(s) matched "inside/**" and nothing matched "papers/**"; write the counterpart change, or ...
	changed: inside/a.txt
x3 docs: branch 5d0e2b1..HEAD judged whole - its commits, uncommitted files and snapshot are one change; any page or any commit's exemption marker in it answers
```

Why the range and not the snapshot (W429, measured in the pilot): the snapshot
says *since the last run*. Code and its API page sat in one commit; the page had
been written before an intermediate gate run and the code fixed after it, so the
snapshot held the code alone and the step stayed red (from the cache, too) while
`-scope head` found nothing. The range is read from git, so the page is in it.

Why the branch and not each commit (W434, measured in this repository): judging
each commit on its own left a branch red for good. A worker whose budget runs
out commits its tree as a WIP commit and the page follows in the next commit;
the exemption lives in a commit body, and a written commit cannot be answered
without rewriting history. Two such commits kept the release branch of this
engine red while their pages sat a few commits later in the same range — and a
`--no-ff` merge would have carried the same red onto `main`. Every git answer
the judgment reads is recorded as the step's input, so a new commit runs the
step again and an intermediate run does not.

The gate plants these cases as real repositories: a tree may carry
`history: {<limb>: [{message, files}]}`, the base is committed to `main`, each
seed becomes one commit on `work`, and the limb's own files are written last,
uncommitted. A seed may name its `branch:` (for example `work/pages-x`); the
commit lands there, and a rule that reads the branch NAME can be planted on two
names with the same content. A planted tree lives until its step ends, so two
working copies of one step stand side by side, as real ones do. A gate step cannot call git (`forbid`) and a fixture tree cannot
hold a `.git` — the engine plants the history itself, through its one git door.

In the same scope the files the rules' `when` and `then` patterns reach are
recorded as the step's **inputs**. Measured in the pilot: the snapshot weighs
the tree through the cache's own reading, which observation never sees, so the
step remembered only its settings — a red taken while the code had changed and
the page had not came back from the cache after the page was written (cached
exit 1, `-no-cache` exit 0). Now a change under either side runs the step again.

The rule's name is in the marker because a commit body answers **one** gate: the
writer saw one rule turn red and answered that rule, while a shared marker takes
that sentence and silences every other rule too, including ones nobody saw.
Measured here — with a shared marker the `docs gate` step's own control
experiment, a rule that must exit `1`, exited `0`: the exemption had excused the
experiment.

<!-- x3-dist version=v0.311.0 capabilities=f78154bbee065e6965565472d835bb54b1e426dfbddbaf80774fab6d9cd67105 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
