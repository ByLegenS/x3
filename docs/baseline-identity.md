# A baseline keyed by identity has no root

[The pages](INDEX.md) - [what x3 is](../README.md)

## A baseline keyed by identity has no root

**What it catches:** the root law locking a command out of its own subtree,
when nothing in its records was ever written against a root.

The law above rests on one thing: the records are **places**. `cmd/x3/main.go`
is a different file under a different root, so a run pointed at a subtree must
be refused before it mistakes a mismatch for a debt paid.

Not every command records a place. `boxes` keys its baseline by the box's
**identity** — a name minted into the box itself:

```yaml
findings:
  - id: 08359efec3ef
    rule: box_owner
    path: cccc55556666      # the box's identity, not a file
```

An identity means the same thing from any root. Applying the root law to such a
baseline buys nothing and costs everything: every narrowed run is refused, so
*"what is still open in this component"* cannot be asked at all. Measured
before this: `x3 boxes apps/whatsapp` exited `2` on a baseline whose every
record was an identity.

So a baseline keyed by identity carries no `root:` at all, and is read — and
refreshed — from any root. The exemption is the command's, not the file's: a
baseline whose records are places is still refused, in the same run, on the
same tree.

**One law replaces the other.** A narrowed run reads the whole baseline but
measures only part of it, so every record outside its subtree would look dead.
Such a run is therefore barred from the dead-record judgment, exactly as a run
narrowed to named rules already is. Without that bar, measured: a run from one
component reported another component's debt as `dead_baseline`.

**A declaration written in the configuration is read where it was declared.**
`boxes.holds.sources` names the files a test runner is driven from — a gate
configuration, a procedure document — and those sit at the project's root. A
narrowed run reads them there, from the configuration's own anchor, not from
under the subtree: a component holds no gate configuration and is not at fault
for it. The guard that refuses a declared place reading nothing is unchanged
and fires from every root.

**An empty answer is an answer, once the question is narrow enough.** A list
that holds no box is fatal from the root: a pattern that quietly stops matching
reports zero, and zero reads as *"nothing is owed"*. Narrowed to one component
the same sentence is wrong — a component with no work left is not at fault, and
*"what is still open here"* has to be allowed to answer *"nothing"*. A run whose
root sits under the configuration's anchor therefore reports `0 box` and exits
`0`; from the anchor itself, or from above it, the block is unchanged. Measured
on one tree, one configuration: `x3 boxes` blocked with `empty_scope` and exited
`1`, `x3 boxes <component>` exited `0`.

<!-- x3-dist version=v0.279.0 capabilities=2d4dbaa4fd0947e09be46fbdc475aa6d2c15d07d65a13b06f643f87f8c988536 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
