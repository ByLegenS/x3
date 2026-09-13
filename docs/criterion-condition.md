# Work that is not needed yet

[The pages](INDEX.md) - [what x3 is](../README.md)

## Work that is not needed yet

Some work is not late, it is **not needed yet**: the migration the next release
will want, the cleanup that starts when a dependency ships. Its box is open, its
proof may even already stand, and closing it would be a lie while opening a
finding against it would be noise.

A condition is written beside the box, in the same words as its criteria:

```json
"criterion": { "key": "criterion", "whenKey": "condition", "kinds": { "...": {} } }
```

```markdown
- [ ] the migration the next release needs
      condition: exists docs/trigger.md
      criterion: exists docs/done.md
```

The vocabulary is shared on purpose: a condition and a criterion ask the same
tree the same question, and only the key says which judgement the answer serves.
The two keys may not be the same word. A condition that does not hold parks the
box — neither direction is asked of it — and one that holds gives the box back to
both.

**Parking is not free.** A condition that never holds would take its box out of
every law and say nothing, which is the quietest way a list stops being
measured. So each parked box is printed by name under `box_waiting`, the run
counts `conditions` and `held`, and a run that wrote conditions and held none of
them adds `condition_never_held`. All of it is a warning: waiting is legitimate,
hiding is not. A condition that could not be **measured** postpones nothing.

<!-- x3-dist version=v0.169.0 capabilities=8add3c844e8497d1ba6e332b2d1943eed701ede05258f37463f9c99f5684e33f template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
