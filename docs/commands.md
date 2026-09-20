# The program a command name means

[The pages](INDEX.md) - [what x3 is](../README.md)

## The program a command name means

Several settings hand the engine a command to run: a setup step, the command
`testdb run` and `guard` wrap, the `command` criterion behind a box or a guard,
and the external parser `syntax` calls. All of them take the command as a
**name**, and a bare name is resolved against `PATH`.

⛔ **A bare name is not one program.** It is whatever the `PATH` of the shell
that started the gate resolves it to, and that answer changes with the shell.
Measured on Windows: `bash` is the MSYS2 shell when the gate is started from a
Git shell, and the WSL launcher Windows ships in its own system directory when
the gate is started from PowerShell. Those are two operating systems with two
different `PATH`s, so a hook that runs `go` is green by hand and exits `127` in
the gate, from the same settings file and on the same machine.

The engine cannot know which of the two a project meant. What it can do is stop
hiding the difference, so it does two things before it runs anything:

| | |
|---|---|
| the name resolves to nothing | the run says so in those words — `"bash" not found on PATH` — instead of letting the failure arrive as an exit code that also means a hundred other things |
| the name resolves | the program is launched **by its resolved path**, and every message about it names that path: `migration hook "bash" (/usr/bin/bash) failed` |

A message that repeats the name as written sends its reader to the wrong
diagnosis. This one was measured too: the failure above was read for a while as
*"the hook does not inherit `PATH`"*, and the environment was never the problem —
the environment is passed through untouched, and both shells hand the child the
same variables. What differed was which program the name meant.

A path written out in full resolves to itself and is printed once, not twice: a
project that already spells the interpreter it wants sees no extra noise. That
is also the fix for the ambiguity — a name the engine has to guess at is a name
the project can write out.

```json
{ "testdb": { "migrate": {
  "command": "/usr/bin/bash",
  "args": ["-c", "exec go run ./cmd/app -migrate"] } } }
```

### Where a flag may stand on x3's own command line

Every command that takes a directory takes it **last**, and a flag written after
it is **refused** (exit `2`), naming what it dropped:

```
x3 case ./core -out report.json
x3 case: 2 argument(s) written after the directory are read by nothing: -out report.json
        flags are read only BEFORE the directory: x3 case -out report.json ./core
```

⛔ **Why this is refused rather than tolerated.** Flag parsing stops at the
first argument that is not a flag, so everything after the directory arrives as
plain words. Until this refusal those words were simply dropped: the run
measured the right tree, printed its findings to stdout, exited `1` — and the
file the caller asked for was never created. A wrapper that reads the red out of
a report file is then blind **exactly when there is something to read**, which
is the one moment it exists for. The same trap was met once inside `outbound`,
where a mode word sits before the flags; this is that answer applied everywhere.

<!-- x3-dist version=v0.252.0 capabilities=3ed7af678967f94585d43ee9355f1770a870cfbd29028c170e635da8a0eccbb9 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
