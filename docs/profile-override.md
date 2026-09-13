# What a project changes about a rule it did not write

[The pages](INDEX.md) - [what x3 is](../README.md)

## Overriding, and switching off

A called rule is not a rule the project has lost hold of. Three ways to differ,
and each one leaves a trace in `x3 profile`:

| | Written as | What it does |
|---|---|---|
| change | `override: { "<rule>": { ... } }` | replaces the fields named, keeps the rest |
| switch off | `off: { "<rule>": "<reason>" }` | the rule does not run, and the sentence says why |
| add | the section's own list | the project's own rules, beside the called ones |

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
| `override` or `off` names a rule no set in use carries | the line does nothing and reads as if it did |
| `with` fills a token no rule in force asks for | so does that one, and the message says whether a switched-off rule is the reason |
| `off` with a blank reason | a gate quietly silenced |
| the same rule overridden and switched off | one of the two lines does nothing |
| the project declares a rule by a carried rule's name | a copy stops following the library the day the library changes |
| a set called without a version, or one the engine does not carry | the message names what it does carry |

### What is not here

The library is carried **inside** the engine, so a call means the same thing on
every machine; a project cannot add a set of its own to it, and cannot switch off
one instance of an `each` rule without switching off all of them. `each` belongs
to called rules only: a project's own rule is still written once per item. And a
called rule has no file of its own, so `x3 placement` — which asks which settings
file a rule was declared in — does not weigh it.

<!-- x3-dist version=v0.172.0 capabilities=aeb308bbc6445b368dba196cbd2cc9d24d585cf0fd3d47313e197f2672ef0a92 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
