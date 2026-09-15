# What a project changes about a rule it did not write

[The pages](INDEX.md) - [what x3 is](../README.md)

## Overriding, and switching off

A called rule is not a rule the project has lost hold of. Three ways to differ,
and each one leaves a trace in `x3 profile`:

| | Written as | What it does |
|---|---|---|
| change | `override: { "<rule>": { ... } }` | replaces the fields named, keeps the rest |
| change one | `override: { "<check>": { ... } }` | the same, for a single check an `each` rule grew |
| switch off | `off: { "<rule>": "<reason>" }` | the rule does not run, and the sentence says why |
| add | the section's own list | the project's own rules, beside the called ones |

A rule measured once per item is not one rule to a project: each item becomes a
check with its own name, and the threshold, source or exemption of one of them
need not be those of its siblings. So an override may name **the check** rather
than the rule - the filled-in name, `every-published-migration-under-db/one-is-sealed`
rather than `every-published-migration-under-{migrations}-is-sealed` - and it
reaches that one check alone. `x3 profile` prints the deviation beside the check
it touched. Without this, one library rule with three thresholds is three copies
of the rule's text in the project, and a copied rule stops following the library.

**An override replaces; it does not add.** This is deliberately the opposite of
the merge law that joins the parts of a split configuration, and for the opposite
reason: there, two files of one project are peers and a list that silently lost
to another would be a setting its author believes is in force. Here the project
is overruling a library, and a `deny` list that grew by addition would be a list
nobody wrote. An override may not set a field to `null` either — a rule that
should not run is switched off, with a reason.

### Nothing goes stale quietly

A profile version is pinned, so its rules do not move under a project. What can
move is the project: a rule renamed or dropped in a later version leaves an
override or an `off` line pointing at nothing. Such a line **stops the run**,
in the same family as every other dead declaration this engine reports, and for
the same reason as an `include` pattern that matches no file: a setting that
changes nothing reads exactly like a setting that works.

| Refused, exit `2` | Why |
|---|---|
| `override` names neither a carried rule nor a check this call sets up | the line does nothing and reads as if it did; the message names both lists |
| `off` names a rule no set in use carries | the same |
| a rule is switched `off` and the project declares its name itself | `x3 profile` would print it off while every command that measures it ran the project's copy |
| `with` fills a token no rule in force asks for | so does that one, and the message says whether a switched-off rule is the reason |
| `off` with a blank reason | a gate quietly silenced |
| the same rule overridden and switched off | one of the two lines does nothing |
| the project declares a rule by a carried rule's name | a copy stops following the library the day the library changes |
| a set called without a version, or one the engine does not carry | the message names what it does carry |

### What is not here

The library is carried **inside** the engine, so a call means the same thing on
every machine; a project cannot add a set of its own to it, and cannot switch off
one instance of an `each` rule without switching off all of them - it can change
one, but not silence one. `each` belongs
to called rules only: a project's own rule is still written once per item. And a
called rule has no file of its own, so `x3 placement` — which asks which settings
file a rule was declared in — does not weigh it.

<!-- x3-dist version=v0.243.2 capabilities=da726653351dbbe2db81062b11393d6c4505651eec6367273b074422dc95a457 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
