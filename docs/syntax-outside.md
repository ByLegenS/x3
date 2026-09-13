# Where the pattern does not look

[The pages](INDEX.md) - [what x3 is](../README.md)

## Where the pattern does not look

An [`ignore`](syntax-ignore.md#the-word-another-language-owns) reads the **text** a pattern
matched. Some rules are not about that text at all: a manual entry is allowed
when it sits inside a container that declares itself the fallback, and the line
that matched says nothing about where that container begins or ends. `outside`
names the container, and the check does not look inside it.

```json
{ "name": "an-identity-box-nobody-declared", "sources": ["ui/**/*.html"],
  "deny": "<(?:input|textarea)\\b[^>]*\\b(?:account_id|tenant_id)\\b[^>]*>",
  "reads": "file",
  "outside": { "open": "<(\\w+)\\b[^>]*data-manual-fallback[^>]*>",
               "close": "</${1}>", "nest": "<${1}\\b" },
  "reason": "an identity the system learns is not one to type by hand; a fallback is declared, not assumed" }
```

| Field | Meaning |
|---|---|
| `open` | the pattern a container begins with |
| `close` | the pattern it ends with; `${1}`..`${9}` carry `open`'s captures as literal text |
| `nest` | optional - what raises the depth, so an inner container does not close the outer one |

**Not an exclusion, and that is the point.** A dead `ignore` is red, because a
stale exclusion is how a gate goes blind. A container nobody has declared yet is
normal: the rule is written before the first fallback exists. So `outside`
claims nothing about the tree and carries no dead-declaration law - what it
carries is a count, `summary.outside`, saying how many matches fell inside one.

**An opening that never closes is only itself.** Taking the rest of the file
would turn one unclosed tag into a way to silence the whole check; it also lets
a self-contained element carry the marker itself
(`<input data-manual-fallback ...>`).

### What the pattern reads at once

`reads` is `line` by default: the pattern is searched line by line and a line is
at most one finding. A tag written one attribute per line is then invisible -
the opening and the identity are never on the same line - and the check goes
narrower than it reads without saying so.

`"reads": "file"` searches the whole text. Every match is a finding (two tags on
one line are two violations) and the line reported is the one the match begins
on. The default does not change: every gate written so far was measured at one
finding per line.

<!-- x3-dist version=v0.190.0 capabilities=0baf44f282fed2c99d88b24a56fbcc96dc3aaa6477ac5d753eddf755e2ee5fc1 template=8b180c04f72b592ba2c6db66547668cfa8fbdb9f09c6051bca2ba17e518cab2b -->
