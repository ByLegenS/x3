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
and not the next. **A selector is weighed inside the package it was given**, and
a pattern on its own knows nothing about place: `runs ./service/ -run Manifest`
does not hold a `TestManifestIsWritten` declared under `src/`. Read across the
whole tree instead, the error runs both ways — a namesake exam in an unrelated
package reads `held` and stays undeletable, while the reader who trusts that
`held` stops the round in the wrong place and sees no symptom. Where a criterion
names **several** places, any one of them covering the file is enough.

Names are matched **as words**: a criterion saying `jeton` does not hold a
declaration called `ton`, and one saying `Readiness` does not hold `Read`.

### The exam a criterion names, and has not been written yet

An **open** box may name a file together with the exam that will one day prove
it — the criterion is written first, the exam follows. Until then that criterion
matches nothing in that file, and a criterion that measures nothing cannot
*stop* measuring: the silent green this mode exists to prevent cannot be born
there. So a `pattern` criterion of an open box that names a file it does not
match today holds nothing, and the file is free to go.

Three things narrow the rule, each for its own reason:

- **only an open box.** A closed one claims it measures the file today; if that
  claim is false it is said elsewhere (`box_unproven`), and holding a file one
  round too long is cheaper than deleting the subject of a claim.
- **only `pattern`.** An `absent` criterion asks for something *not* to be
  there, and removing the file would turn it into an empty green for good.
- **the file's own text is read**, not the tree: the same exam declared in some
  other package says nothing about this one.

Measured in a pilot: two exam files were held by exactly this writing — an open
box naming an exam that exists nowhere in the tree — and neither could be
removed while that was read as a bond.

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

<!-- x3-dist version=v0.160.0 capabilities=e88f95261480eb59757c7a6380cf3f12994a6220aec98dc46bd65ac5e6967e82 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
