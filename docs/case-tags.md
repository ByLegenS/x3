# Examples behind a build tag

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, examples behind a build tag

Some of what an example needs is not hidden behind an import but behind a
**build constraint**. A package's database helpers commonly sit in a file that
opens with

```go
//go:build integration
```

and that file is simply **not part of the package** unless the build carries
the tag. An example whose setup calls such a helper does not fail its
assertion; it never compiles. The run says `does_not_build — undefined:
fixtureRate`, and no amount of declaring imports helps, because the name is not
in another package — it is in *this* package, in a file the compiler was not
given.

### The declaration, and the run that carries it

A file says which tags its examples need:

```go
//x3:tags: integration

package usage
```

and a run says which tags it carries:

```
x3 case -tags integration ./...
```

or, for a project whose default run always carries them:

```json
{ "case": { "tags": ["integration"] } }
```

The flag **replaces** the setting rather than adding to it: someone narrowing a
run by hand would otherwise measure a tree the setting silently widened. A
declaration names tags and only tags — `!live`, `a && b` and the rest of the
constraint algebra stay in Go, where they already work; the only question asked
here is whether the run carries this name.

The directive binds the **file**, exactly as `//x3:import:` does, and is written
above the `package` clause for the same reason: a declaration sitting over one
declaration reads as if it bound only that one, and anywhere else it is
`malformed`. What the tag reaches, though, is wider than the file — in Go a
build tag decides which files the whole **package** is assembled from. The file
declares because the file is what needs it; the report says which tags the run
carried, so the wider reach is never a guess.

### An example a run does not run, said out loud

When a run does not carry a tag a file declares, that file's examples are
**deferred**: they do not run, and the run does not pretend they did.

```
DEFERRED rate.go:14 (Scale): the run does not carry build tag(s) integration
x3 case: 2 example(s) in 1 package(s) - 1 passed, 1 deferred, 0 finding(s)
```

Every deferral is printed with its file, its line and the declaration it calls,
and the report carries the same list under `deferred[]` with a `deferred` count
in the summary. A gate that wants to refuse any unrun example can read one
number and say so.

This is not a skip, and the difference is the whole point. **A skipped example
is green without having run**, which is the one thing an example must never be.
A deferral is decided **before anything is compiled**, from a declaration
written in the source, by a switch the run itself sets — so the run can name
every example it is not running, ahead of running the rest. An example that
starts and then bails out remains what it always was: `never_ran`, and red.

Deferral also hides nothing. The same example, once the tag is carried, runs and
is judged: a wrong expectation is `example_failed` exactly as it would be
anywhere else. And an example that declares nothing is never touched by any of
this — it runs in a tagged run and in an untagged one alike.

### Tags that do nothing are red

A tag on a command line is a claim that it does something. Two reds keep the
claim honest, and they are the same law read from either end:

- the run carries a tag **nothing in the tree uses** — `dead_tag`, named on the
  configuration file or on `-tags`, whichever declared it;
- a file declares tags and **has no examples** — `dead_tag` on that line: the
  declaration binds that file's examples, and there are none to bind.

*Uses* has two doors, and the second opened this page: an example **asks** for a
tag with `//x3:tags:`, and a `//go:build` line **binds** the build with it,
pulling in a file no example declares. Asking only the first reddened a correct
run — `x3 case -tags integration` over a directory whose helpers sit behind that
tag. A misspelled tag is still caught, because neither door names it.

Without the first red, a gate script could keep a tag long after the last file
that needed it was renamed, reporting green over a narrower tree than anyone
believes it measures. The second catches it one file earlier.

### An example that never ran can be refused

A deferral is loud and not red, which is right for a run narrowed on purpose
and wrong for a gate: drop one entry from `case.tags` and every example behind
that tag defers, the run says so, and the exit code is still `0` — the hole a
skipped test leaves, one level up.

```json
{ "case": { "deferred": { "policy": "block", "max": 0 } } }
```

| Field | What it is |
|---|---|
| `policy` | `warn` (default) or `block` |
| `max` | in `block`, how many deferrals are allowed; 0 when not written |
| `listed` | how many examples the finding names; 5 when not written |

The fields and their defaults are the ones `test.skips` carries, because two
places asking one question should take one answer under one name. The default
stays a warning — **a declaration cannot redden an existing tree by itself** —
and the finding is charged on the configuration, naming the examples. It is
asked on every run, narrowed or not: it measures this run rather than judging
a declaration, so the scope rule above does not apply to it.

The run-side question is asked of a tag from the configuration **only when the
configuration file lies inside the tree being run** — the same rule that governs
`dead_import`, for the same reason: a repository-wide setting judged from a
single directory would call a correct declaration dead. A tag typed on the
command line is not held back that way; both doors are asked of it here.

### An example only a person asks for

Some measurements must not run on every gate: they call a paid service, a real
provider, or something that takes minutes. Moving them out of the package costs
what an example is for — they need the package's own unexported setup. They stay
where they are, behind a tag the configuration names **opt-in**:

```yaml
case:
  optin: [live]
```

```go
//x3:tags: live
//x3:env: PROVIDER_ENDPOINT

package quote
```

A default run does not run them, and does not call them deferred either. A
deferral is a hole in the tree — a tag dropped from the settings — and
`deferred.policy: block` turns it red; an opt-in example not running is the
run's own decision, so it is counted apart and the gate stays green:

```
OPT-IN probe.go:13 (Quote): runs only when asked for, with -tags live
x3 case: 2 example(s) in 1 package(s) - 1 passed, 1 opt-in not asked for, 0 finding(s)
```

A person asks for them by name, narrowed if they like:

```
x3 case -tags live -only internal/quote
```

| What happens | What the run says | Exit |
|---|---|---|
| asked for, its environment set, the answer right | `OBSERVED probe.go:13 (Quote): reached api.example` | 0 |
| asked for, a variable `//x3:env:` names is empty | `UNMEASURED probe.go:13 (Quote): asked for, but the environment does not carry PROVIDER_ENDPOINT` | 2 |
| asked for, the answer wrong | `example_failed` with the comparator's own sentence: `out[0] = "quoted by api.broken", want "quoted by api.example"` | 1 |
| the configuration's `case.tags` carries an opt-in tag | refused at load: every gate would run what only a person may ask for | 2 |

`//x3:env:` names the variables the file's examples need before they start —
names, never values, because a value written in the tree is a value in the
repository. An empty variable counts as unset. An example that cannot be
measured is **neither green nor red**: exit 2 is the same contract as a run that
measured nothing. What an asked-for example logs with `t.Log` is printed as
`OBSERVED` and carried in the report under `observed[]`, apart from the verdict;
when such an example fails, the finding is still the comparator's sentence, not
the line a helper logged before it. A package whose opt-in examples run is never read
from or written to the cache: their answer comes from outside the tree, and the
tree's fingerprint cannot vouch for it.

A shared helper the opt-in file needs sits in a fixture whose constraint names
both, `//go:build x3fixture || live`; under `-tags live` it is compiled once,
not twice. The opt-in file may name that helper in its **code** as well as in
its examples, and either keeps the declaration alive: a file that carries
`//x3:tags:` builds under the same tag the fixture does, so its code counts
toward `dead_fixture` the way an example or a test file does. Untagged code does
not count — it cannot see a fixture, and a name it shares would only be its own.
Until 2026-10-03 only examples, fixture lines and test files counted, so a helper
called from the opt-in file's code was reported dead. A file hidden behind `//go:build live` **alone**, with no
`//x3:tags:`, never reaches the compiler in a default run, and the run says so
with the remedy instead of a bare `undefined:`:

```
quote.go:14 (Quote): does_not_build
	undefined: Quote; the file builds only under //go:build live, which this run does not carry - write //x3:tags: live above the package clause, and the run defers its examples or runs them with -tags live
```

<!-- x3-dist version=v0.331.0 capabilities=4dd95ef758f7d81516514d63de82e8b4bd7d4d60b318b986c8348e71d92492c2 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
