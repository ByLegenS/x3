# Where a run's wall clock goes

[The pages](INDEX.md) - [what x3 is](../README.md)

## Where a run's wall clock goes

A total tells you a run was slow. It does not tell you which part was slow, and
a gate that runs *no step at all* can still be slow — in which case the total is
measuring everything except the work.

```
x3 gate -region docs -phases
```

prints one line beside the summary, on stderr:

```
-- phases: startup 591ms, config 590ms, workers 0ms, cache-read 5ms, baseline 1ms,
   regions 7ms, build 0ms, sums 274ms, scopes 97ms, steps 56ms, cache-write 11ms,
   reach 0ms, rest 0ms (total 1.64s); 2 configuration read(s), 3 tree walk(s)
```

`startup` is measured from the moment the process reaches the engine, not from
the moment the command starts: the binary loading, an alias being resolved and
the flags being parsed all happen before any command can start a clock, and a
clock that starts inside the command reports nothing about them. `rest` is the
wall clock no phase claimed — a growing `rest` says the line itself has a hole
in it.

**Two of those numbers are not seconds, and that is the point.** How long a
phase takes depends on the machine, the disk and the size of the repository, so
it cannot be tied to a threshold. How many times the configuration was parsed
from scratch, and how many times the tree was walked, are properties of the
*mechanism*: a second full read inside one run means a path that does not
remember its answer, and that is the same defect on every machine.

### A trial the step does not run again

A step is one thing in the report, but its trials are remembered one by one.
When a step runs again because something it reads changed, every trial that
runs `x3` (and so reports what it read) is first looked up in the cache: if
nothing *that trial* read has changed, its exit code and output come back from
the record and the trial is not run. The line says so:

```text
== the two modules    (1573 ms)
  the a module judges the examples it holds exit=0 (want 0) - recalled
  the b module judges the examples it holds exit=0 (want 0)
```

That is the engine's own `trial memory control experiment`: one step, two
trials, each running `x3 case` on its own module. `b` changes, the step runs
again, and only `b` is measured; when `b` then breaks, `b` runs and the step is
red while `a` still answers from its record. Control: with recall switched off
the second arm is red (`a` never says `recalled`), and with recall that ignores
what a trial read the third arm is green when it must be red.

A trial that does not run `x3` (it reports nothing) is remembered alone only
when its step declares its own `touches` and the trial writes nothing into the
repository: its record is the step's derived scope, the declared `touches`, and
every repository file its arguments name. A step that inherits the gate's scope
keeps such a trial running every time: a tool can open files by default flag
values no argument names, and only the declaration says so. A file a non-x3
trial names in an argument is also part of the step's own record.

Most trials of a real gate are experiment arms on a fake tree; their inputs are
the tree (part of the cache salt) and the overlay files they name, not the
source file that changed. Measured on a pilot: 212 of 275 trials ran on a fake
tree, and a one-file change in the core re-ran all sixteen trials of one step
(77 s) while only two of them looked at the real tree.

What is never recalled:

- a trial whose command is not `x3` (`go run`, `go vet`): it leaves no
  observation, so the step's own scope stands for it and it runs whenever the
  step runs;
- every trial of a step where two trials name the same `{tmp}` path (one writes
  a report with `-out {tmp}/r.yaml`, another reads it with `-from x={tmp}/r.yaml`),
  or where the step's own `env` names `{tmp}`;
- a trial that failed for an environment reason, and every trial of a cold run
  (the gate re-measures red steps once after writing its own record).

A recalled trial's observation is added to the step's, so the step's record
and the undeclared-region check see exactly what they would have seen had the
trial run. `X3_CACHE_WHY=1` names the trial that ran again and the file that
moved it (`"<step> · <trial>" ran again: file x3/arch.yaml changed`).

### Trials that run side by side

A step's trials used to run one after another, so a step of fourteen
independent `go run ./cmd/<tool> -page <file>` trials took fourteen times one
trial. Now a trial that cannot read what another trial of its step leaves
behind runs on an idle gate worker when one is free (and a machine slot with
it, never waited for); otherwise it runs in order on the step's own worker.
The line says so:

```text
== trials that leave nothing for one another    (85 ms)
  one                                    exit=0 (want 0) - alongside
  two                                    exit=0 (want 0)
  three                                  exit=0 (want 0) - alongside
```

That is the engine's own `trials side by side control experiment`
(`internal/gate/testdata/side-by-side.yaml`). Controls: three trials sharing
their working directory `{tmp}` never say `alongside`, and with `-workers 1`
no trial does.

A path after `-o`/`-out`/`-output` counts as written, so a trial naming it waits
for the writer — **unless the command carries a bare `-check`**: then the path is
the file it compares against, not one it writes (`api-doc -check -out
report.json`; measured on a pilot, eight such trials ran one after another).
`-check name` (a value, as in `x3 syntax -check rule`) is not that flag. The
examples on `writes` and `together` in `internal/gate/trial.go` measure both.

A trial keeps its order (it waits for the trials before it, and the trials
after it wait for it) when:

- it names a `{tmp}` path another trial names, or the step's `env` names `{tmp}`;
- it plants the same tree as another trial (`tree: t:a` twice: the tree is
  removed and planted again in the same place);
- it writes a repository path with `-o`, `-out` or `-output` that another
  trial names;
- it runs on the real tree with `-w`, `-fix` or `-write`.

What it does not do: a trial that writes into the repository any other way
(a tool that rewrites a file it was only told to read) is not seen; give such
trials a shared `{tmp}` path or split them into their own step.

**A path a `steps` guard writes is held machine-wide while the guard runs.**
The gate cannot see inside `x3 guard`: a trial that runs it names no `-o`, yet
the guard's own steps build `go build -o out/x.exe` and read it back with
`go version -m`. Two such runs at once (two gate steps, or overlay trials of
one step side by side) read each other's half-written binary — measured on a
pilot: of thirteen concurrent runs of the clean guard, two said *"unrecognized
file format"*. Now every `-o`/`-out`/`-output` path of the guard's `steps` and
`after` that lies outside its `workspace` is locked (`~/.x3cache/claims`, one
lock per path, taken in order, dropped by the system when the process dies)
from the first step to the end of the teardown; a second run waits. Control:
the same four-overlay burst eight times over, 0 of 8 red. The examples on
`claims` (`internal/live/steps.go`) and `Outputs` (`internal/probe`) measure
which paths are locked; a workspace keeps relative paths private and locks none.

### Work a run remembers instead of repeating

Two memos are opened explicitly, by the command, and closed when it ends.

`source.Memoize` remembers the *listing* of a tree walk — never the contents,
because handing the same bytes to two rules would let one change what the other
reads. `config.Memoize` remembers a *configuration reading*: the root file, its
ancestors and every part it declares, merged, with the profiles expanded. The
key carries the overlay a run put on top, because the same file under a
different overlay is a different configuration.

Neither is opened by a command that *writes* what it then reads. A command that
edits a configuration and reads it back would be handed the remembered older
copy, and a stale configuration is a silent green.

Measured in the pilot on 2026-09-21, on a gate run that ran zero steps:

| | before | after |
|---|---:|---:|
| `x3 gate -region <one> -phases` wall | 16.5 s | 1.69 s |
| `-- phases` total | 15.31 s | 1.64 s |
| `reach` (verb coverage line) | 6.25 s | 0 ms |
| `regions` (region resolution) | 3.72 s | 7 ms |
| `x3 gate -regions` (prints a list, runs nothing) | 11.3 s | 1.21 s |

Nothing about the work changed; the same answers are computed once instead of
five times. The region resolution used to walk the tree and re-parse the
configuration once per question — `Known`, `Spread`, `Reach`, `Shadowed`, and
once more per step for the declaration check — while the entry gates parsed the
whole configuration twice before the command even began.

<!-- x3-dist version=v0.332.0 capabilities=acab9267b660ce2bb879a50621abb53e34cfe764125b0b99a100a70c0f62bdbb template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
