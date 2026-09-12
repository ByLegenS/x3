# Releases, and calling the engine from another project

[The pages](INDEX.md) - [what x3 is](../README.md)

## Releases and reproducible builds

The engine is published as binaries into a public repository carrying nothing
else: two binaries, `SHA256SUMS.txt`, and a generated set of documents — this
guide's own pages. The source repository is private, so the binary and its
documentation are the whole public surface.

One command produces a release, and if any step fails nothing is published:

1. it builds `windows/amd64` and `linux/amd64` with
   `-trimpath -buildvcs=false -ldflags "-s -w -buildid= -X main.version=<tag>"`
   and `CGO_ENABLED=0`, so the binary carries no build path, no build id and no
   VCS stamp — only the tag;
2. it builds **each target a second time** and compares the two SHA256s. A build
   that does not reproduce is not published, and that comparison happens on every
   release rather than in a one-off experiment;
3. it writes `SHA256SUMS.txt` and generates every public document from this
   document plus a template — what the engine **is** in `README.md`, how each
   capability is **used** on its own page under `docs/` with an index beside
   them, and the lookup tables on a reference page per family, listed in
   `REFERENCE.md` — stamping the tag and the SHA256 of both sources into
   **each** generated file. Which page a section belongs to is written in this
   document, not in the script: a `x3:doc` marker opens a page, and everything
   up to the next marker is that page. A `x3:ref` marker moves the table under
   it to the reference page of the page it sits in, so the grouping is
   **derived** rather than declared a second time — a sub-page joins its main
   family, `arch-prose` writing into the `arch` reference. Nothing is written
   by hand;
4. it re-reads what it just wrote and runs the staleness gate against it.

**The staleness gate** recomputes the SHA256 of this document and of the template
and compares them with the stamp in **every** published document. Either changing
after the last release run turns the step **red**: the binaries do one thing and
the documents describe another. Because the publication is a *set*, the gate reads
the set from the same list that gives each document its cap — a page whose
generation was skipped, or one left over from an earlier run, is named and red.
A publish directory that is not configured, or configured and missing, is red as
well and says `NOT GENERATED` — deliberately not a green skip, because "nobody
has published yet" and "the publication is current" are not the same answer. The
step carries two control experiments: the same question with a deliberately wrong
document hash, and a copy of the publication with one page removed.

Three further gates stand between this document and the public repository, and
each one is the engine reading its own output. **The leak gate** reads every
generated document against a list of patterns the project keeps privately — real
module and directory names, project and customer names, local paths, account
names — and a single match stops the publication; the finding names the pattern
and masks the value, because a gate that printed what it found would be a second
leak. **The language gate** reads the same documents for a second language: the
template's maintainer notes are stripped before publication, but an HTML comment
ends at its first `-->`, so a stray one inside a note would end the block early
and publish the rest. What the strip cannot guarantee, the output side can, and a
foreign word or letter in a published document is red. **The size gate** is the
engine's own `freeze` cap, one per published document: it takes no debt, cannot
be lowered by `-update`, and a document above its cap is red — so splitting a
document is a way of keeping it measured, never a way around the measurement. All
three run in `check.ps1` as well, each with a two-way control experiment.

The publish directory is configuration and never a constant in the code:
`-DistDir` wins, then `X3_DIST_DIR`, then `dist.dir` in `x3.json`.

## Using x3 from another project

**Integration is by binary, not by import.** The consuming project does not add
x3 to its `go.mod`, does not use a `replace`, and does not put it in a `go.work`.
The directives are plain comments, so the consuming project's compiler never sees
them and its dependency graph never learns that x3 exists.

**Pin a version, and let the engine fetch itself.** The project writes the
version it requires into its own `x3.json`
([the minimum version gate](update.md#the-minimum-version-gate)) and calls
[`x3 update`](update.md#x3-update) to obtain that binary. Nothing else about x3 is
tracked: no downloader, no checksum file, no path.

This corrects earlier advice, and the reason is worth keeping. The first
integration had the project carry its own script to read a pinned version,
download the binary and verify the sum. That script was correct and still wrong:
every project using the engine would write the same one, each with its own bugs,
and the engine could fix none of them. A checked-in path is worse still — green
on the machine that wrote it, unmeasured everywhere else, and unable to say
*which* build ran.

**A missing or mismatched binary is red, not skipped.** The pilot's gate was
fail-open at first: no binary meant a warning and a normal start. That is the
failure this engine exists to prevent. **"The tool was not there" and "the tool
found nothing" must never produce the same colour.**

**The counts are the engine's job, not yours.** A scan of a tree with no
directives exits `0` and reports `0 red`, so a gate trusting the exit code alone
turns "delete the directives" into a way to go green. The first answer was to
have the project read the report and assert on it — which worked, and meant every
project wrote the same counter with its own bugs. The count now belongs to the
configuration: see [Expectations](scan.md#expectations).

**Directives arrive next to the existing tests, not instead of them.** Nothing is
migrated until its x3 equivalent has been seen to go red on a deliberately broken
input.

<!-- x3-dist version=v0.150.0 capabilities=162fe2ced0d891cd8733aba17d14fcabc3618c79fd93d1547dabbbcdc64d0fcb template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
