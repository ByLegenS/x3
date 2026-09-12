# Which files the check is about

[The pages](INDEX.md) - [what x3 is](../README.md)

## Which files the check is about

A path glob says where a file **sits**; a rule usually asks what it **does**.
Two files in the same directory, one assembling a report and one not, are the
same to `sources` and different to the rule — and the line that separates them
is inside them. `holds` and `lacks` bind the subject set to the text:

```json
{ "name": "a-report-breakdown-carries-no-cost-of-ours", "sources": ["**/*.go"],
  "holds": "report[.]Definition[{]",
  "lacks": "requireSystem[(]",
  "deny": "cost_micros|provider_model",
  "reason": "a breakdown a customer can open must not carry what we paid" }
```

| Field | A file is a subject when |
|---|---|
| `holds` | its text matches the pattern |
| `lacks` | its text does not match the pattern |

The second one is how a gate marker inside the file takes it out of scope: the
screens that are allowed to show a cost are the ones that check for the right
to see it, and they say so in their own code.

**Not an exemption, and not an exclusion either**, exactly as
[`subjects`](arch-pairing-subjects.md#pairing--which-files-is-the-rule-about) in an `arch` rule.
An exemption overrides a measurement, so it needs a reason; `exclude` names a
**path** and so claims the path exists, which is why a dead exclusion is red.
These two name a **property** and claim nothing about the tree, so there is no
dead-declaration law to run here.

**The elimination is counted.** `summary.eliminated` says how many files the
condition took out, and if it takes them **all** the check is `empty_scope`,
red, with the condition written into the message. A condition nobody can see
work is the quietest way to turn a gate off. The file is read once: the same
bytes the condition read are the ones the check reads.

<!-- x3-dist version=v0.160.0 capabilities=e88f95261480eb59757c7a6380cf3f12994a6220aec98dc46bd65ac5e6967e82 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
