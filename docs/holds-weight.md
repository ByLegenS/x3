# Which hold is really a hold

[The pages](INDEX.md) - [what x3 is](../README.md)

## Which hold is really a hold

A file declares names, and a name is a short word. Ask about a file and the
engine asks about everything in it — the point, and the danger: a declaration
called `Do`, `now` or `run` appears in criteria that never look at that file, and
counted as bonds those records bury the real ones. So the two questions are kept
apart.

**The file itself** (`via: file`) is always a bond. Somebody wrote that file into
a criterion. The mention is read as a **path token**, not as a word: the match is
widened to the whole path it sits in, and that path has to end the file being
asked. `internal/parse/line_test.go` written in a criterion does not hold
`web/line_test.go`; a bare `line_test.go` holds both, because it names no
directory and every tree carries two files with the same base name.

**A name the file declares** (`via: symbol`) is weighed against the place the
criterion looks at:

| The criterion | Where it looks | A declared name is |
|---|---|---|
| `pattern`, `absent` | the files its `sources` match | a bond only there |
| `file` | the path it names | a bond only there |
| `command` | the place read out of its arguments | a bond there, and everywhere when it names no place |
| `sql`, `manual` | nowhere the engine can read | **`unsure`** — kept, never dropped |

The place is read out of the **shape of the arguments** — one that carries a
slash and nothing but path characters is a place, `./...` being the whole tree —
never out of a runner's flag names, which would make the engine know one runner
and not the next. A selector is a pattern and a pattern knows nothing about
place: `runs ./service/ -run Manifest` does not hold a `TestManifestIsWritten`
declared under `src/`. Where a criterion names **several** places, any one of
them covering the file is enough.

Names are matched **as words**: a criterion saying `jeton` does not hold a
declaration called `ton`, and one saying `Readiness` does not hold `Read`.

### A name whose file is gone

A name already deleted declares nothing anywhere, so there is no file to weigh a
criterion against and every criterion that mentions it — or whose selector
matches it — reads as a bond. Measured over 172 names removed from one
repository's history: **5 held, every one of them from a command running in a
different part of the tree.** So an ask may carry the place the name came from:
`service/window:TestWindowReadsBothEnds`. It is weighed against **`command`
criteria only**, by directory — one whose place covers it, or lies inside it,
still holds the name; one that runs elsewhere does not. Pattern and file criteria
are left alone on purpose: their places are file globs, and a directory measured
against a file glob misses, turning a real bond into a silent "free". A left half
that is neither a path nor a directory in the tree is no place, and the ask stays
one name. The same 172 asked with their places: **0 held, 172 free** — while a
name still living inside the place its criterion runs in is held either way.

### The record the engine refuses to decide

A `sql` or `manual` criterion carries no place, so relevance cannot be measured.
A declared place naming something the file declares cannot be weighed either —
a runner **is** a place, so there is nothing left to weigh it against — and is
printed as `SUSPECT`. Both count as `unsure` and the run stays red. Dropping it would be a
guess in the direction that kills measurements; calling it a bond would hide
which records were actually weighed. The two mistakes are not equal: a wrong
"held" leaves a file in the tree that could have gone, a wrong "free" is the
silent green this whole mode exists to prevent — so every class the engine
cannot decide is counted on the bond side, and said out loud.

<!-- x3-dist version=v0.158.0 capabilities=e84674ef730d4c01bd28143e856bf5a927aed61a6b4eead5472b98cb6d4baaa2 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
