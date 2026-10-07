# JavaScript examples

[The pages](INDEX.md) - [what x3 is](../README.md)

## JavaScript examples, run by `x3 case`

**What it is for:** the pure logic of a web front end — rounding an amount,
splitting a total, the bounds of a retry delay, mapping a key — is otherwise
measured by hand in a browser, or not at all. The same rounding rule often lives
in a Go package that carries examples and in a JavaScript module that carries
none; only one side was ever measured.

An ES module (`.js`, `.mjs`) carries **the same line** a Go file does, above an
**exported** declaration. There is no second example language: the grammar is
the Go one (`given=(...)`, `in=(...)`, `out=`, `then=(...)`), the expressions
inside it are JavaScript, and `out0` names the returned value.

```js
import { half } from "./lib/half.mjs";
import { scale } from "/shared/scale.js";   // a browser path, mapped below

//x3:case: in=(125, 10) out=130
//x3:case: in=(-125, 10) out=-130
//x3:case: in=(124, 10) then=(out0 === 120)
export function roundTo(value, step) { ... }

//x3:case: in=(100, [1, 1, 1]) out=[34, 33, 33]
export const allocate = (total, weights) => { ... };

//x3:case: then=(ROUNDING.length === 2, ROUNDING[0] === "half-up")
export const ROUNDING = ["half-up", "half-even"];

//x3:case: in=(3) out=6
export async function doubled(n) { return scale(n, 2); }
```

- `in=(...)` **calls** the function below it (an `async` result is awaited);
  `out=` compares the value deeply and strictly (`util.isDeepStrictEqual`),
  so `[34, 33, 33]` is compared element by element and `0` is not `-0`.
- Without `in=(`, the line **measures a value** — a constant, a table, a
  class — with `then=(...)` alone. Every export of the module is in scope by
  its name, so a proposition may read any of them.
- Only comments may stand between the line and the declaration; a blank line
  or any other line breaks the tie (`not_a_function`), as in Go. A function
  returns one value, so `out=` names one.

`x3 case` finds these files in the same walk that finds Go files; a project
writes nothing to switch it on. Every module that is not cached runs in **one**
`node` process, so a tree of many modules pays the start-up of one.

```yaml
case:
  js:
    node: node            # the default; a path also works
    paths:                # a root-absolute import, as the browser writes it,
      /shared/: shared/   # read from this directory under the root
```

```
$ x3 case web/                 # a wrong expectation: red on its own line
amount.mjs:5 (roundTo): example_failed
	out[0] = 130, want 120
x3 case: 6 example(s) in 0 package(s) - 5 passed, 1 JavaScript module(s), 1 finding(s)
```

**What is red, and why none of it is skipped:**

| Code | When |
|---|---|
| `example_failed` | `out=` differs, a proposition is false, or the call threw |
| `malformed` | the line breaks the grammar, or its expressions do not compile as JavaScript |
| `needs_a_browser` | loading the module, or the call, reaches `window`, `document`, `localStorage`, `navigator`… |
| `does_not_build` | the module does not load in node (a missing import, a syntax error) |
| `no_runtime` | `node` is not found; the finding names `case.js.node` as the remedy |
| `crashed` | node ended, or reached `case.timeout`, before the example reported |

A module that needs a browser is **unmeasurable, and says so**. Node defines a
`navigator` without `onLine`, so `navigator.onLine !== false` would pass there
and prove nothing; the runner removes `navigator`, `localStorage` and
`sessionStorage` before loading any module, and the read becomes a red
`needs_a_browser`. Keep the pure logic in a module that needs no browser, and
measure that module.

**The cache follows the imports.** A module's answer is stored only when every
example passed, under a key that carries the runner, the `node` binary, the
`paths` setting, and the content of the module **and of every file it imports**,
transitively (relative and mapped imports are read; a package name or `node:`
module is keyed by its name). A key on the module alone would show the old
green after a helper it imports changed. Measured on this engine's fixtures:

```
$ x3 case web/                          # cold: one node process
x3 case: 6 example(s) in 0 package(s) - 6 passed, 1 JavaScript module(s), 0 finding(s)      151 ms
$ x3 case web/                          # nothing changed: node is not started
x3 case: ... 1 JavaScript module(s) (1 unchanged since the last run), 0 finding(s)          86 ms
$ (lib/half.mjs, imported by amount.mjs, changed)
amount.mjs:6 (roundTo): example_failed
	out[0] = -120, want -130
```

Control experiment: the five `case JavaScript control experiment` trials of the
`inline examples` gate step (green, wrong expectation, browser API, malformed,
no node), and two examples on `Check` (cached when nothing changed; measured
again when an imported file changed). Each arm was broken once and turned red:
a key that ignored imports kept the changed tree green; a runner that did not
remove `navigator` passed `online()`.

<!-- x3-dist version=v0.340.0 capabilities=7f596239b33d07b08fcfa0550f33f7a3d4eadd50b8ad99f6fce73dc731635008 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
