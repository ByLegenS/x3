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

<!-- x3-dist version=v0.158.0 capabilities=e84674ef730d4c01bd28143e856bf5a927aed61a6b4eead5472b98cb6d4baaa2 template=dc09b1bbb2d660b8d4128f8b3c106398aba2584e6aea0ed894d6a3e548e0fdf0 -->
