# Gaps in what a work list can say

[The pages](INDEX.md) - [what x3 is](../README.md)

## Gaps in what a work list can say

"Gaps we know about" bounds what a matcher can READ. These bound what a list of
open work, and the examples run beside it, can SAY.

**`boxes` measures evidence, not completion.** A command criterion runs where the
gate runs, and a move is trusted once its target exists.

**"Could not measure" is declared, not inferred.** The engine has no idea which
sentence a runner prints when it skips; a project that declares none keeps the
older reading, where an unmeasurable criterion and an unproven one share one code.
And `summary.unmeasuredBoxes` counts every box carrying such a criterion, which is
more than the number of `box_unmeasured` findings: a box that also fails a
criterion it *did* measure is counted here and reported as `box_unproven`.

**A name a declared place mentions cannot be weighed.** A criterion carries a
place to compare against; a runner *is* the place. Such a record is `SUSPECT`
and its entry `unsure` - kept on the not-free side, never counted as a bond.

**Examples are one call, not a scenario.** An example cannot expect a panic and
cannot read a value it mutated.

**A declared build tag is not checked against the build constraints.** The
engine asks whether a run carries a tag an example asks for; it does not verify
that the tag actually gates a file. It cannot: a tag may gate a file in a
*dependency* of the package being run, so "no file here is constrained on it"
would call a working declaration dead. A tag nothing is constrained on is
therefore not red on its own — its examples simply stay deferred, named on
every run until someone carries it.

**The language gate speaks one language** — `en` is the only embedded dictionary.

<!-- x3-dist version=v0.82.0 capabilities=11edd916bf4f7054d8961098862c218884f6184a237e77ee3193ebf2a0e9dc05 template=c40035a911f18207838ce450f42d40bb4e85fe48362e4b316bf413025402ab83 -->
