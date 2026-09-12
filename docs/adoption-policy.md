# Binding the gate without a permanent red

[The pages](INDEX.md) - [what x3 is](../README.md)

## One known debt, without a permanently red gate

Every project adopting a gate has one finding it already knows about and is
already working on. With a single word the choice is between a gate that can
stop nothing and a gate that is red until the work lands — and a gate that is
always red is read by nobody. So `policy` also takes an **object**:

```json
{ "policy": {
    "*": "block",
    "tests_remain": {
      "policy": "warn",
      "reason": "the migration to inline examples is under way; the counter must fall, not stop the gate" } } }
```

Everything not named takes `*`. **Nothing is silenced**: the finding is still
measured, still counted, still printed with its own message — it is marked
`ALLOW` instead of `BLOCK`, its reason is printed beside it, and the run prints
a table of every exception with **how many findings each one touched**.

An exception is the same promise an exemption is, so it carries the same laws:

| The exception | What happens |
|---|---|
| carries a reason | required; without one the run exits `2` |
| written as a bare word (`"warn"`) | exit `2` — it has nowhere to put the reason |
| names a code this gate cannot produce | exit `2`; an exception for nothing hides the day the code changed |
| the object has no `"*"` | exit `2`; a gate whose unnamed codes must be guessed is trusted by nobody |
| matches **no finding in this run** | **red** — `dead_policy`, whatever `*` says |
| names `dead_policy` itself | exit `2`; the audit of exceptions cannot be excepted from itself |

The last two rows are the whole point. An exception that has stopped being true
is the quiet way a gate goes blind, so `dead_policy` is **always** a block: a
project that could soften the audit of its own softening would have written a
permanent exemption in two lines.

<!-- x3-dist version=v0.149.0 capabilities=aa2b3d5359a52c0465529a4d78500da0ece5c1d342d9261b163f39c08cf09ce1 template=d6bc32c7a50d63dff3c2e3a215156b8f0c9ba214600907d4b4b40c9d4169d73d -->
