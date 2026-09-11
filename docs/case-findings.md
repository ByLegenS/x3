# What a run says

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 case`, what a run says

### Findings

See **case finding codes** in [REFERENCE.md](../REFERENCE.md#case-finding-codes).

`does_not_parse` is the one finding that is not about an example. A file the
parser cannot read carries no examples the gate can see, and a package whose
files were never read cannot be called green: the run would print *"0 example(s)
in 0 package(s) - 0 passed, 0 finding(s)"* and exit `0` over a package that does
not compile. **A gate does not lean on somebody else's red.** The compiler will
say it too, and saying it twice is cheaper than a sentence that is not true.

### A run that dies is not a run that skipped

A panic is caught and turned into that example's red — the generated test
recovers, and the neighbours keep running. **A fatal runtime error is not a
panic**: `fatal error: concurrent map writes`, a deadlock, a run out of time.
Nothing recovers from those; the process dies where it stands, its own verdict is
never written, and no example after it ever starts.

Read by verdict alone that looks like a skip, and the whole package reads as
*"never ran"* — the exit code is right and the diagnosis sends the reader
somewhere else. So the engine reads the **shape**, not the words: an example the
stream started and never finished is `crashed`, and the reason is whatever the
run itself said. No runner's sentence is written into the engine, because a
fatal is spelled one way here and another way in the next language.

The neighbours are then **measured, not guessed**: the dead example is charged
and dropped, and the package runs again without it, inside the same round budget
that compile errors use. Measured on one tree of four examples — one killing the
process, three sound: before, four `never_ran` findings and nothing proven;
after, one `crashed` naming the culprit and **three passed**. If the budget runs
out first, the examples that never started say so, with the name of what killed
the run in their finding.

### Propositions stop at the first one that fails

Propositions in one `then=(...)` are evaluated in order, and the first one that
does not hold ends the example. Written in sequence they are each other's
**precondition**:

```
//x3:case: in=("nope") then=(out0 != nil, out0.Error() == "not found")
```

The second is meaningful only while the first holds; without it, `out0.Error()`
on a nil error is not a red, it is a **panic**. A panic ends the process that is
running the package, and the results of every other example in that package are
then never reported — the gate would be right that something is wrong and wrong
about what. Short-circuiting is the same thing Go's own `&&` does.

A panic that survives anyway — a single proposition that dereferences nothing —
is caught and charged to **that** example as `example_failed`, saying it
panicked. Its neighbours keep running.

The first three are answers the toolchain gave; the last four are refusals made
**before** anything runs.

### Settings

Optional — an example lives in the source, not the configuration:

```json
{ "case": { "exclude": ["internal/legacy/**"],
            "imports": ["net/http/httptest", "typeset \"example.com/lib/text/core\""],
            "timeout": "2m" } }
```

`imports` is the pool above, each entry `[<name> ]<path>` — the same grammar the
`//x3:import:` directive uses, because one declaration read two ways would be
valid in one place and malformed in the other. `timeout` (default `1m`) is applied to the test binary **and** to the toolchain
call around it; only the first would leave a run that hangs downloading a
dependency waiting forever.

### The report

```json
{ "version": 1, "root": ".", "config": "x3.json",
  "findings": [ { "file": "wallet.go", "line": 8, "target": "Add",
                  "code": "example_failed", "message": "out[0] = 5, want 6" } ],
  "summary": { "files": 1, "packages": 1, "cases": 1, "passed": 0, "findings": 1 } }
```

`passed` is counted separately from `findings` on purpose: "no findings" and "no
examples" are not the same sentence.

<!-- x3-dist version=v0.107.0 capabilities=e5b56b1fd4cf5a1646ddb67bc4b87955524c10c3013d934948604da0d7630322 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
