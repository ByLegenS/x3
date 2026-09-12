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

<!-- x3-dist version=v0.135.0 capabilities=20b1c981592aadc535186606e8f8a71ef40bca930051d3944300ed66b331d3e9 template=36de115a7d2b7ce379f073b81526b976f20d62ea52cb57c9054b36ca5cdb0a46 -->
