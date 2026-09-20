# What this project asks of the engine

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 config`

```
x3 config [-config <file>] [-out <file>] [-check]
```

**Catches:** a project whose own tooling nobody can see. A mature repository
declares a dozen capabilities across a dozen settings files, and the answer to
"what do we actually run here, and what does it cover?" lives in nobody's head.

`x3 config` writes that answer as settings, not prose: the engine's version, the
shortcuts `x3 do` offers with their flags, the capabilities this project declares
and the ones it leaves idle, and every gate step with its band, what it reads,
and whether it can be remembered between runs.

⛔ **Three lists, not two.** A command with no settings section of its own is
neither declared nor idle — `fmt` runs every turn and asks for nothing. Written
as "not used" the inventory reports a gap that is not there, and the reader goes
off to close it.

### A long line nobody types twice

```json
{ "alias": { "full": "gate", "quick": "gate -band fast", "suite": "test -fresh-db" } }
```

`x3 quick` runs `x3 gate -band fast`. The caller's own arguments are appended, so
`x3 quick -out report.yaml` works and adds to the alias rather than replacing it.

A command line that is not remembered is either typed wrong or not typed at all.
Measured in a production application on the day this was written: a gate run
without its database variable declared reported **18 red and a false 71 seconds**
— the line itself was right; what was missing was writing all of it every time.

⛔ **An alias cannot shadow a real command.** The table is consulted only for a
name the engine does not know, so `x3 gate` is always the engine's own gate even
if a project writes an alias by that name. Letting settings rename a tool's own
commands means the same command doing two different things in two repositories.

⛔ **It is binding, not decorative.** `-check` writes nothing and exits 1 when the
file on disk is not what this run would write. As a gate step it means a change
to the settings cannot land while the inventory still describes yesterday's
project. An inventory nobody verifies is worse than none: it is read with trust
and it is wrong.

## `x3 gate`

```
x3 gate [-config <file>] [-band fast|commit|full] [-only <names>] [-workers <n>] [-tune] [-out <file>]
        [-changed] [-red-exit <n>]
```

**Catches:** a gate that lives in a shell script — which every project then writes
again, and which runs its steps one after another because a script has no other
way to run them.

A gate is a list of things to run and the exit each one should give. That is a
**list**, not a program, and once it is written as a list two things follow: the
engine can run it, and it can run the steps **at once**. Most steps touch nothing
the others touch — each gets its own working directory — so the order a script
imposes is the script's limit, not the work's.

```json
"gate": {
  "binary": "_build/tool.exe",
  "build": "./cmd/tool",
  "stamp": "main.version",
  "steps": [
    { "name": "gofmt", "band": "fast",
      "trials": [ { "say": "every file formatted", "run": "gofmt -l .", "want": 0, "not": ".go" } ] },
    { "name": "language gate", "band": "fast",
      "trials": [
        { "say": "this repository", "run": "{bin} lang .", "want": 0 },
        { "say": "a planted sample", "run": "{bin} lang testdata/red", "want": 1 }
      ] }
  ]
}
```

### The steps run at once — and so does everything they call

**Measured in a production Go application on 32 logical cores:** the gate ran 32
steps at once, and each step's own tools spread across all 32 cores as well —
32 × 32 = 1 024 threads on 32 cores. The processor sat at **99.6% busy** and the
work still finished late: 44 compiler processes taking turns on 32 cores, each
one hauling its own data back into the cache on every turn. Two layers of
parallelism, neither aware of the other.

`gate.env` declares what every step's environment carries, and `{cores}` opens to
one step's share of the machine: processors ÷ workers, never below one.

```json
"gate": { "workers": 24, "env": { "GOMAXPROCS": "{cores}" } }
```

| The same tree, the same steps, nothing removed | Wall | Total work |
|---|---:|---:|
| 32 workers, no budget | 1:58 | 2 012 s |
| 24 workers, `{cores}` | **1:49** | **1 558 s** |

⛔ **The engine names no tool.** It hands out the share as `{cores}`; which
variable carries it is the project's word (`GOMAXPROCS`, `MAKEFLAGS`, …), and a
project that wants a flat number simply writes one. A step's own `env` and a
trial's `env` write over this one, in that order.

**Busy is not useful.** The measured difference between a share of 1 and a share
of 2 was noise (1:49 ↔ 1:47), so the division is enough; a separate ratio field
would only be one more number to get wrong.

### A comment is not an input to every step

A run that calls the engine reports what it read, and the cache binds the step to
those bytes. But most measurements never look at prose: a rule about imports, a
dictionary check, a containment rule — none of them change their answer when a
sentence above the code is rewritten. Binding them to the whole file makes every
comment edit re-run them.

So the observation now carries **how** a file was read, not only **that** it was
read. A command that asks for no comments (`x3 arch`, unless a rule declares
`comments: checked`) records a code reading, and the cache hashes that file with
its comment spans **removed**. ⛔ Removed, not blanked: blanking preserves line
length, so lengthening a sentence still moves the digest — measured, and the
first cut of this bought nothing because of it.

Measured on a real production Go application, one comment line edited in one core
file, nothing else:

| | bound to the file | bound to the code |
|---|---:|---:|
| steps that ran | 34 of 82 | **25 of 82** |
| serial work | 149 s | **45 s** |
| slowest step | a dictionary rule, 18.2 s | the comment-length gate, 8.5 s |

The slowest step in the second column is the one that **measures comments** — it
runs, and it should. Both directions were measured: editing only the comment
leaves those rules skipped; editing one line of code in the same file runs them
again, and the report names the file and says `code … changed`.

⛔ **Not knowing a language's comment syntax means the whole file is code.** The
cost of ignorance picks a direction: a step that sees too much runs for nothing,
a step that sees too little goes green without measuring.

### A body is not an input to every step either

A rule that asks *"what does this package import"*, *"is there a declaration of
this name"* or *"does this file declare anything that runs"* cannot change its
answer when the inside of a function is rewritten. Comment-free hashing does not
help it: renaming a local variable is code.

So there is a third reading. A file read as a **surface** is hashed by its
package clause, its import list (aliases and blank imports included), every
declaration's **signature**, every constant and variable's value, and **every
directive line anywhere in the file**. Function bodies do not enter it.

| Reading | The step is bound to | Freed from |
|---|---|---|
| `text` | every byte | nothing |
| `code` | the file without its prose | comment edits |
| `surface` | declarations, signatures, imports, directives | function bodies |

⛔ **A directive is not prose and is never dropped** — from either of the two
narrow readings. `//go:build` decides whether a file compiles at all, `//go:embed`
decides what ships inside the binary, and `//x3:allow` decides which finding is
forgiven. All three are written with a comment marker, and a digest that dropped
them would not notice an exemption being **added**: the gate would then repeat an
answer measured before the exemption silenced a finding. They are weighed wherever
they stand — including inside a body, which is where an exemption is often needed.

⛔ **A file the parser cannot read has no surface.** The digest comes back empty,
nothing matches it, and the step runs. Calling a broken file "unchanged on the
surface" would be a green measured on a tree that does not build.

**The reading is per file, not per run.** A rule set is narrow but the walk is
wide: `x3 arch` reads the whole tree to build the component map and the import
graph, even when the selected rule only looks at four dictionaries. Asking "what
is this *run's* reading" therefore applies the widest rule to every file it
touched. Each file is now asked separately — *which rules actually open you?* — and
files no rule opens are bound to their surface, because the only things the run
took from them are a component (a path), an import list and an exemption line.

This holds because every place that consumes a parsed file is behind that same
question. The three places that are not are exactly the three the surface carries:
the component map reads the path, the dependency graph reads the imports, the
exemption tally reads the directives.

Measured on a real production Go application, 82 gate steps, one core file
touched, nothing else — the same edit twice, once inside a body and once in a
signature:

| | one local variable renamed | one named result added |
|---|---:|---:|
| steps that ran | **45 of 82** (was 53) | **53 of 82** |
| serial work | **158 s** (was 287 s) | 350 s |
| wall clock | **19 s** (was 25 s) | 27 s |
| steps woken by a surface | **0** | **8**, each naming the file |

The eight steps that sleep through the body edit are the same eight that the
signature edit wakes. Both directions were measured on the same tree, and the run
carried the same 21 red steps, named identically, before and after.

⛔ **A run that does not report what it opened is bound to nothing it measures.**
This was found while measuring the above and is the more serious half of it:
`x3 syntax` read its files with the plain file reader instead of the engine's,
so its observation held 23 settings files and **not one of the 800 sources it
scanned**. A gate step built on it survived any source edit and repeated its last
answer without running. Every engine reader that opens a project file now reports
it — syntax checks, paired files, generated documents, document length, live
settings. The reading mode is a speed question; this one was a correctness
question, and it is why a narrow reading has to be declared by the rule rather
than guessed by the cache.

### The worker count is the machine's, not the project's

The peak sits in a different place on every machine: processors, disk, database
and the mix of steps all move it. A number found by hand once becomes a number
nobody dares touch, and the day the machine changes it is quietly wrong.

`-tune` runs the gate at four worker counts — the processor count and its three
quarters, half and quarter — and reports the peak:

```
x3 gate -tune
x3 gate: measuring this machine's worker count - 4 full run(s)
x3 gate: 24 worker(s) on 32 processor(s) - 32:118204ms 24:108094ms* 16:121530ms 8:186402ms
```

The result is written **beside the cache, not into the settings**: a worker count
is a property of the machine, and a machine's number pushed into a shared file is
wrong for everyone else who reads it. The record carries the processor count that
produced it, so a binary copied to another machine — or a virtual machine that
grew — measures again instead of trusting a number that no longer describes
anything.

**A gate with no declared `workers` tunes itself.** There is no command to
remember: an absent setting is what triggers the measurement, a stored one is
read, and `-tune` forces a fresh measurement over both.

⛔ **The measurement checks itself.** If one point leaves a different number of
steps red than the others, the points did not do the same work and their times
cannot be compared — a red step exits without finishing, which shortens the wall
clock and flatters the point that broke. Measured in a production application: the
same tree run without its database declared reported 18 red in 71 s, and with it
11 red in 111 s. A tuning run like that writes nothing and **names the steps that
moved** — a count alone sends the reader back to run the gate by hand twice to
find out which one is unsteady, and that name was already in the report. Two
points that leave the same number red but not the same steps are just as
incomparable, and are reported the same way. A tuning run like that says why — and then
**the gate runs anyway**, one worker per processor. A measurement is a
convenience, never a precondition: the opposite would leave a project whose gate
is red unable to run the gate that shows it. `-tune`, asked for on purpose, does
report the failure and exits 2.

### A step that cannot have changed is not run again

```
x3 gate [-cache <file>] [-no-cache]
```

Most of a mature gate measures **the gate itself**: steps that build a made-up
tree, run the engine against it and check what it says. Measured in a production
application: of 254 trials, **169 ran in a made-up tree** and cost 516 seconds of
every run — 55% of the total. Those trials cannot see the repository's own code,
so a commit to it cannot change their answer. They depend on the engine's version
and on the settings, and both are already in the cache's salt.

A step is remembered when the engine can name everything it reads: either **every**
trial carries a `tree` (nothing in the repository reaches it), or the step declares
its files in `touches` and their contents go into the key. One trial reading a
repository the step never declared puts the whole step outside the cache — a step
is one thing, and half of it remembered would be a gate reporting on work it did
not do.

⛔ **The key carries the binary the gate measures with.** A gate that builds its
own tool from the repository (`gate.build`) would otherwise remember a step across
an edit to that tool: the version stamp is a git tag and does not move when a
source file does.

⛔ **Red is stored too, and comes back red.** An earlier version kept only green,
reasoning that a stored red repeats a sentence this run never measured. The same
is true of green — so that rule was defending the *key*, not the colour. Once the
key carries the content of everything the step reads, both colours are equally
sound: the inputs did not change, so neither did the result. Measured in a
production application: two full runs back to back on an untouched tree ran the
same 57 steps twice (59 578 ms, then 57 200 ms, 9 red both times). Every one of
those seconds bought an answer that was already known.

⛔ **A remembered red is counted red.** The summary looks at the colour of every
step, not only of the ones that ran — otherwise a fault would go quiet on the
second run, which is the one failure this whole feature could cause.

⛔ **A step that declares no scope is named.** A step without `touches` inherits
the gate-wide `reads`, and that reads as *"my scope is wide"* to whoever wrote the
settings, when what it says is *"I wrote no scope"*. The run reports how many
steps did that and the report lists them by name — measured in a production
application: **65 of 82**. The report is where the list belongs; sixty-five names
on the terminal every run is a wall nobody reads. The step is not wrong to inherit,
and nothing is done to it: an observation replaces the inherited scope the moment
the step leaves one, so the ones that stay are exactly the ones running a command
the engine cannot see into.

⛔ **A record's key is generated, never typed.** A step's entry used to be filed
under its own display name — `go vet (integration) · apps/whatsapp`, a sentence
written for a human, with spaces, punctuation and whatever language the project
speaks. A free-form label doing an identifier's job fails the same way every time:
somebody improves the wording, and the match silently disappears. The key is now a
digest of the step's definition; the label lives inside the record
(`data.result.name`), so a person reading the file still sees which step it is while
nothing machine-facing depends on the words. The digest covers the **whole**
definition, name included: leaving the name out would let two steps land on one
identity, and a cache that hands back the wrong record says it measured something it
did not. Renaming a step costs one extra run; a collision costs a false green.

⛔ **A filter that selects nothing is a green that measured nothing.** `-only` looks
for text inside a step's name, so a renamed step or one mistyped letter in a shortcut
turns the whole run into `0 ran, 0 red, exit 0` — measured, and it is the loudest
possible argument against carrying a closed set as free text. Every pattern must now
hold at least one step; one that holds none stops the run and names itself, with exit
2, because this is not a finding but a failure to measure. The engine's other
commands already enforced this; it was missing where it cost the most.

⛔ **A run declares its reading, or it is bound to every byte.** Two more commands now
say what they read: an inline example is a **directive** and the code it measures is
compiled, so `x3 case` binds to the code and not to the prose around it; a criterion's
pattern defaults to reading code, so `x3 boxes` does the same unless a criterion asks
for `reading: text`. Measured in a production application: one sentence rewritten in a
core file ran those two steps for **34 s** and **50 s**.

⛔ **A record names paths the way the repository does.** Every path a step's memory
carries is written relative to the repository root. Absolute paths tie the record to
one machine and one directory: move the checkout, clone it elsewhere, or open a
second working copy of the same tree, and every one of those keys is dead — silently,
because a miss looks exactly like a change. Measured in a production application:
one gate cache held **5 309 absolute paths**; the file also halved, from 0.8 MB to
0.4 MB.

⛔ **The salt carries the gate's own section, not the whole configuration.** The
older fingerprint hashed every settings file into the salt, and a salt that does not
match discards the entire cache — so a comment added to any settings file made all
82 steps forget. That was right before observations existed: nobody could say which
section reached which measurement, and forgetting everything was the honest answer.
It is no longer a guess. A step that runs the engine reports the settings files it
**read** (23 of them in the pilot), so the configuration is already in the record,
per file and precise. Keeping it in the salt as well counted it twice — once exactly,
once wholesale. Measured after the change: editing `x3/arch.yaml` re-runs the one
step that reads it, names the file, and leaves the other 81 alone.

⛔ **An alias that carries a command's name never runs, and says so.** Resolution
only looks at the settings when a name is **unknown**, so an alias called `docs` can
never shadow `x3 docs` — that rule is deliberate, because a tool whose own names can
be overridden from settings does two different things in two repositories. What was
missing is the noise: the alias sat there looking like a shortcut and quietly ran
something else every time. `x3 config` now refuses the configuration and names it.

⛔ **Another command's cache is not this step's input either.** A run reads its own
cache and writes over it, so the gate leaves its own cache directory out of every
record. That exclusion used to follow the gate's cache wherever `-cache` put it —
which meant that moving the gate's cache made the *other* commands' caches, still
sitting where the settings declare, look like ordinary repository files. Both are
excluded now: the one in use and the one the settings name.

### One step written once, run per application

```json
{ "name": "vet · {each}", "each": { "dirs": "apps/*" },
  "touches": ["{each}/**", "core/**"],
  "trials": [{ "run": "go vet ./{each}/...", "want": 0 }] }
```

A repository with three applications should not carry three copies of the same
step, and one with a thousand should not carry a thousand. `each.dirs` matches
directories on disk and the step is produced once per match, with `{each}` opened
to that directory in the name, the commands, the environment and the scope.

**The list lives on disk, not in the settings.** A new application is measured the
day it is created; nobody has to remember to copy a step, and no copy goes stale
on its own.

⛔ **Each copy carries its own scope.** Written with `{each}` in `touches`, a copy
reads only its own tree — so a change to one application does not run the other's
steps. That is where the saving is; multiplying alone would only shorten the
settings file.

⛔ **A pattern that matches nothing is an error**, not a silent gap: a dead
pattern removes a step from the gate without anyone noticing, and a tree nobody
measures is exactly what a gate exists to prevent.

### A step whose files did not change

```json
{ "name": "messaging worker", "touches": ["apps/messaging/**", "core/**"], "trials": [...] }
```

`touches` is the step's cache key: the content of every file it names is hashed
into the key, and the step runs only when one of them changed since it last ran.
This is the same law as the step cache, applied to the steps that do look at the
repository: one reads a made-up tree and is remembered while the **settings** hold
still, the other reads the real one and is remembered while **its own files** do.

⛔ **The question is "since this step last ran", not "since the last commit".** An
earlier version asked git what had changed and skipped the steps that did not match
— which answers a different question. Switching branches, committing, or doing
nothing at all moves git's answer without moving a step's input. Worse, a skip like
that had to *assume* a colour, and it assumed green: a step that was red, whose
files were not in the last commit, came back green without being measured. Content
in the key removes the guess — the step returns the result it actually produced.

⛔ **A step that declares nothing runs every time.** The engine does not guess
what a step reads: a wrong guess shows something green that was never measured,
and it does so in silence. Writing `touches` is the project's decision, and it
documents the step's scope in the same line.

### One scope for the whole gate, and the steps that cannot have one

```json
{ "gate": { "reads": ["apps/**", "core/**", "settings.json"],
            "steps": [ { "name": "ledger ↔ production", "volatile": true, "trials": [...] } ] } }
```

`gate.reads` is the scope every step inherits when it declares no `touches` of its
own; a step's own declaration replaces it. Measured in a production application:
of 73 steps only **5** declared a scope, because declaring one meant repeating the
same twelve roots 68 times — so 68 steps re-ran on every gate, and two consecutive
runs on an untouched tree measured 57 steps twice. The cost of repeating yourself
was being paid in seconds.

⛔ **A scope has to be honest, not narrow.** A list naming every measured root of
the repository is already enough to answer "nothing changed at all", which is the
case that costs the most. Narrowing it — this step reads only this application —
is a second, separate gain, made one step at a time and measured.

### A step whose input is a database

```json
{ "name": "ledger reconciliation",
  "state": "x3 testdb state -env APP_DSN",
  "trials": [...] }
```

`state` names a command that **prints** the step's non-file input; its output goes
into the key. The step is remembered while that line holds still and runs again
the moment it moves. The engine does not know what is being asked — the project
writes the command, so a project on another kind of store writes another command.
`x3 testdb state` is the one the engine ships: one line summarising the database,
measured at **1.4 s** against a remote server, in place of steps that cost 76 and
78 seconds every run.

⛔ **A state command that fails leaves the step unremembered.** A gate that cannot
read the state must not assume the state did not move.

⛔ **`volatile: true` says the step's input is not a file**: database rows, the
network, the clock. Such a step inherits no scope, is never remembered, and runs
every time. It is a separate word on purpose. A reconciliation step in a production
application asked twenty-four queries of a live schema while declaring a file scope
in the settings; the database can move without a single file moving, so the gate
skipped it as *"nothing it reads has changed"* and **counted it green while it was
red**. Declaring both a file scope and `volatile` is refused rather than resolved:
a reader could not tell which one the engine believed. So is declaring both
`state` and `volatile`: one says the input can be read, the other says it cannot.
Prefer `state` wherever the input can be printed — `volatile` is for what cannot,
such as a step that builds the database it measures.

⛔ **The cache says why it missed.** `X3_CACHE_WHY=1` makes a step that ran
again name the input that moved: `"architecture" ran again: dir ops/tmp/log
changed`. Without it, finding out why a step re-ran on a tree nobody touched
means guessing — and the answer is often that the run itself wrote into the tree
it measures. Measured in a production application: a step kept re-running
because the operator was redirecting the gate's own log into a directory that
step scans. That is not a cache fault, and no amount of reasoning about the
cache would have found it.

⛔ **A tree the walk does not enter can still be a dependency.** Fixture
directories (`testdata`) are skipped by every scan, but a test runs on top of
them: change a fixture and the answer changes. A step's memory therefore carries
the **content digest of every fixture tree** the run walked past, not just the
files it opened. Not entering and not depending on are different things, and
the first silently implied the second.

⛔ **An answer that came from a command, not a file, is an input too.** A step
that asks the version control system *"what changed?"* gets a list back, and when
that list comes back empty the step opens **no file at all** — its whole
observation is the settings it loaded. Nothing in the tree is then in the key, so
the step is remembered until a settings file moves, and in between a real change
goes unmeasured. Measured in a production application: a *documentation ↔ code*
step answered **green from the cache and red without it, on the same tree at the
same moment** — a finding the gate had lost. So the engine records the question
as well: every command it runs against the repository's history is stored as
*where it was asked, what was asked, and the digest of the answer*, and
validating a memory **asks it again**. The question is checked last, because a
file digest reads the disk while a question starts a process.

⛔ **A recorded question can only be a bare program name.** The memory is a text
file and the check runs what it finds in it; a name carrying a path separator is
neither recorded nor run.

⛔ **The file set is hashed once**, not per step, and only the files some step
declares. Measured in a production repository: 1 142 source files, 19.9 MB, hashed
in full in **206 ms** — which is also the whole of what a timestamp check could
save. A timestamp lies in both directions: a checkout refreshes it with the content
unchanged, and a copy carries an old one onto new content. The second is not a
slow gate, it is a green one that measured nothing.

### A hook can run the gate at the end of every turn

`x3 gate -band fast -changed -red-exit 2` — two flags for callers that are not a
person reading the output. `-changed` measures nothing when the working tree is
clean, so a hook that fires every turn costs nothing on the turns that wrote no
files; when git cannot be read the gate **runs anyway**, because a gate that
cannot read its own input must not go quiet. `-red-exit <n>` picks the code a red
step returns (1 by default): hook protocols differ, and without the flag a
project had to wrap the gate in a script that only translated one number into
another.

### Overlays can live in one ledger

`-with <file>` takes an overlay. `-with <file>:<name>` takes one **out of** a
ledger — a file whose top-level keys are overlays. One file per overlay measured
32 files in a production repository, averaging fifteen lines each; thirty-two
files in a directory are not a list, they are a pile, and comparing two of them
meant opening two files. A name the ledger does not carry stops the run.

### A trial can run on a planted tree

A control experiment needs a small wrong tree to measure against. Keeping those
trees on disk looks harmless and is not: measured in a production repository,
nineteen experiments held **420 files of which only 140 were unique** — 67% was
one limb copying another limb's `go.mod`. A copy goes stale the day the base
changes, and four hundred files are not read, they are scanned.

`gate.trees` declares them instead. A tree has a **base** and **limbs**; a limb
writes only what differs, and `null` removes a file the base laid. The trial
names `"<tree>:<limb>"`, the engine plants it before the call and removes it
after, and `{tree}` is where it stands.

```json
"trees": {
  "sample": {
    "base":  { "x3.yaml": "{ … }", "data.json": "{ \"ok\": true }" },
    "limbs": { "green": {}, "broken": { "data.json": "{ \"ok\": tru" },
               "pruned": { "data.json": null } } } },
"steps": [
  { "name": "planted tree", "band": "fast", "trials": [
    { "say": "the base alone parses", "tree": "sample:green",
      "run": "{bin} syntax -config {tree}/x3.yaml {tree}", "want": 0 },
    { "say": "a limb that breaks a file", "tree": "sample:broken",
      "run": "{bin} syntax -config {tree}/x3.yaml {tree}", "want": 1 } ] } ]
```

A limb nobody declared, and a `null` over a file the base never laid, are both
settings errors — a limb quietly running on an empty directory measures nothing.

**`run` takes a line or a list.** A line is split the way a shell splits one:
spaces separate, and a quoted span (`"` or `'`) stays one argument. A list is
taken as written. Measured in a production repository before this was allowed:
of the gate's 3 264 lines, **1 929 were the `run` array alone** — 210 calls, each
spread over nine lines. A gate list is read; a call spread over nine lines is
scanned, not read. The list form stays, because it is how an argument carrying a
space is written without quoting, and how generated settings write it.

`{bin}` is the binary the gate builds before any step runs — built first on
purpose, because steps run in parallel and one of them producing the file the
others call would be a race. `{tmp}` is the step's **own** working directory, so
two steps writing "the report" never write the same path.

### Every trial says what it expects

`want` is not optional. *"It ran and did not explode"* is not a measurement, and a
green step means something only when the red beside it is really red — which is
why a step carries several trials: the clean tree exits `0`, the planted one exits
`1`, and both are printed. `says` and `not` measure the output the same way for a
command whose exit code alone cannot tell the two apart.

### The ladder, and what is skipped

`band` puts a step on the ladder `fast < commit < full`; asking for `commit` runs
`fast` too, because a commit run that skipped the quick steps would measure not
less but **wrong**. `needs` names an environment variable a step cannot run
without. A skipped step is always **printed with its reason**: a step that goes
quiet is a hole in a green run that nobody can see.

⛔ **An undeclared `needs` is worse than a missing one, and it was measured.** In
the pilot, 14 of 82 steps read a database and none of them said so. Run without
the connection string they did not skip — they **ran and went red**, and the
summary could not tell a missing environment from a finding: **21 red, of which
only 13 were real**. Worse, a step that goes red this way is remembered like any
other: five of those reds survived into a later run that *did* have the database,
which then reported a fault measured on a broken machine. With `needs` declared,
the same tree reports **14 skipped, 8 red** — and every one of the 8 is in the
13 the database run finds. A skipped step is never remembered, so nothing leaks
into the run that has the environment: giving the variable back brought the same
13 reds, named identically.


### A red the step can live with

Some reds are not the tree's fault: the tool is not installed on this machine,
the vulnerability database could not be reached. A gate written as a script
handles those with a branch that reads the output and downgrades the colour —
and every project writes that branch again, differently.

```json
{ "run": ["govulncheck", "./..."], "want": 0,
  "tolerate": ["loading vulnerability database", "executable file not found"],
  "because": "the network's failure is not this repository's red" }
```

`because` is required. A tolerated red nobody explained is never questioned
again, and the difference between "the database was unreachable" and "the
check has been off for a month" is exactly that sentence.

### The variables a step declares

`needs` says a step cannot run **without** a variable. `env` says what the step's
own environment **is**:

```json
{ "name": "schema drift", "band": "full",
  "env": { "APP_DEV_DSN": "${APP_DSN}" },
  "trials": [ { "run": ["{bin}", "guard", "-only", "schemadrift"], "want": 0 } ] }
```

A trial can carry `env` of its own, and it wins over the step's: one command,
two arms, two environments. `${NAME}` also opens **inside** a value
(`"sandbox;${PATH}"`), and `${NAME:-fallback}` gives an unwritten variable a
default that lives in the settings rather than in a script. The same opening
happens in a command's **arguments**: a step whose target changes from machine
to machine carries its default where the step is read.

One address, two names, and both of them pointing at the same place in the same
run — otherwise the same tree gives two different numbers in two gates. A shell
script does this with a wrapper function around every step that shares a database;
a list does it in the step itself, where it is read next to the step it belongs to.

A declaration whose source is **empty** is red and the command does not run at
all: a variable that quietly arrives blank is a step measuring something other
than what it says. A value that is not `${...}` is used as itself.

### What parallel buys, measured

The report prints the wall clock next to the work done — `1.5x faster than one
after another (20476 ms of work)` — and the five slowest steps with their share.
Both numbers are there for the same reason: a gate nobody can see inside of is a
gate nobody makes faster, and a single total hides the one step eating the run.

<!-- x3-dist version=v0.250.0 capabilities=96ca39ab4786b45b9750319ad62467e417e5c13c6ed211793e24b02e5240b9d4 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
