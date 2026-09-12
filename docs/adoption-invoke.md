# How the engine is called

[The pages](INDEX.md) - [what x3 is](../README.md)

## How the engine is called

The engine's name cannot be assumed. x3's own gate compiles a binary to a
run-scoped path and calls it through a variable — `& $bin scan .` — and a
measure that only knew `x3 scan` would report *"this project runs nothing"* on
the very repository that runs everything. So the spelling is declared, and the
default (`x3 <command>`) is only a default. A captured word is kept **only if
the engine has a command by that name**, and comments are blanked before the
pattern is applied, because a step that was deleted must not stay alive in the
prose that described it.

**A gate script also prints.** The line a gate writes to the console — *"the
upgrade is the engine's own work — x3 update"* — is not a comment. It is a
string, it survives the comment pass, and a measure that reads it as a run
reports a capability nobody uses as used. That is the worst mistake this gate
can make: it exists to show what is **not** running, and a measure that goes
blind exactly where it is needed is a fault, not a boundary.

**The rule is not "drop strings."** Real calls carry the command name in quotes
often enough — `Start-Process -ArgumentList 'record', '-listen', …` — and a
blind rule would drop those as well, so the measure would lie in the other
direction instead. What separates the two is **where the match begins**: a match
that runs from its first character to its last inside one string is text; a
match that begins in code is a call.

That leaves the decision with whoever writes the pattern, which is where it
belongs. `-ArgumentList '([a-z]+)'` begins at a flag, in code, so its quoted
capture counts. A project that genuinely builds its command line as one string
writes a pattern that begins before the quote — `[$]cmd\s*=\s*["']x3\s+([a-z]+)` —
and that counts too.

| Shape in the gate script | Counted as a run |
|---|---|
| `x3 scan .` | yes |
| `& $bin scan .` | yes, when `invoke` declares `[$]bin` |
| `-ArgumentList 'record', …` | yes, when `invoke` declares the flag |
| `# the x3 scan step measures …` | no — a comment |
| `Write-Host "  next: x3 update"` | no — a string, start to finish |

<!-- x3-dist version=v0.156.0 capabilities=c8bb02269798cd209388b465a9941635adeb7d3cc673c6643bd116b9a446af22 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
