# Which hold is really a hold

[The pages](INDEX.md) - [what x3 is](../README.md)

## Which hold is really a hold

A file declares names, and a name is a short word. Ask about a file and the
engine asks about everything in it — which is the point, and also the danger: a
declaration called `Do`, `now` or `run` appears in the words of criteria that
never look at that file. Counted as bonds, those records bury the real ones, and
a run that cannot trust the answer filters it by hand — where the real bond gets
filtered away together with the noise.

So the two questions are kept apart.

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
declared under `src/`, because the criterion never looked there. When a criterion
names **several** places, any one of them covering the file is enough.

Names are matched **as words**: a criterion saying `jeton` does not hold a
declaration called `ton`, and one saying `Readiness` does not hold `Read`.

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

<!-- x3-dist version=v0.79.0 capabilities=3c45ec9abee79b86bf9ba9bca65f118d2c089dfaf5d4502d319cba631f64e38a template=4c123e84344b7bfc12ab4a657b26ee954cda22cc067d7dff91cf88d206276e26 -->
