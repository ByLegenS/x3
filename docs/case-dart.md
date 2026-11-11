# Dart examples

[The pages](INDEX.md) - [what x3 is](../README.md)

## Dart examples, run by `x3 case`

**What it is for:** the pure logic of a Flutter or Dart application - a price
rule, a rounding step, a parser, a formatter - is the part that can be wrong
without a screen showing it. It is usually measured by a separate test file
that nobody keeps beside the code. Keep that logic in plain Dart (a library that
imports no Flutter) and write the example **above the function it measures**.

A `.dart` file carries **the same line** a Go file does, above a **top-level**
declaration. There is no second example language: the grammar is the Go one
(`given=(...)`, `in=(...)`, `out=`, `then=(...)`), the expressions inside it are
Dart, and `out0` names the returned value. Named parameters are written as Dart
writes them.

```dart
/// Cuts a decimal amount to minor units: 1200.365 -> 120036.
//x3:case: in=(1200.365) out=120036
//x3:case: in=(0) out=0
int toMinor(num amount) => (amount * 1000000).round() ~/ 10000;

//x3:case: in=(amount: 2543.32, passengers: 3) out=762996
//x3:case: in=(amount: 1, passengers: 2, breakdown: [FareLine(price: 5, count: 1)]) then=(out0 == 200)
int offerTotal({required num amount, required int passengers, List<FareLine>? breakdown}) { ... }

//x3:case: in=(120036) out="1.200,36"
Future<String> formatMinor(int minor) async { ... }   // an async result is awaited

//x3:case: then=(minorStep == 1)
const int minorStep = 1;                              // a value, measured by then=(...) alone
```

- `in=(...)` **calls** the declaration below it. The example runs in the
  library's own scope: every name the library declares or exports **and every
  name it imports** (the generated file repeats the library's `import`
  directives with the same prefix, `show` and `hide`, relative paths turned into
  absolute addresses), so an example may build a value of a class the library
  only imports (`FareLine(price: 5, count: 1)`).
- A declaration may return a **record** (`(int, int)? bounds(...)`,
  `({Item? cheapest, Item? fastest}) pick(...)`); `out=(10, 30)` compares it.
- `out=` compares **deeply and strictly**: lists, maps and sets element by
  element, and `2` is not `2.0` (an `int` is not a `double`). A failing line
  names the value it got: `out[0] = 120036, want 120037`.
- Without `in=(`, the line **measures a value** - a constant, a getter, a
  class - with `then=(...)` alone.
- Only comments, doc comments and `@annotation` lines may stand between the
  example and the declaration; a blank line or any other line breaks the tie
  (`not_a_function`). A name that starts with `_` is private to its library and
  cannot be measured. A function returns one value, so `out=` names one.

`x3 case` finds these files in the same walk that finds Go files; a project
writes nothing to switch it on, but dart must be able to read the project's
package map (`dart pub get`, or `flutter pub get`, writes
`.dart_tool/package_config.json`).

```yaml
case:
  dart:
    bin: dart                                  # the default; a path also works
    packages: .dart_tool/package_config.json   # the default; relative to the root
```

**One process, no file in the tree.** Dart cannot compile an expression while
it runs, so the engine turns every example into source: one generated file per
library (it imports the library the way the package map names it, so a type has
one identity) and a `main` that calls them all. These files live in the
operating system's temporary directory and are removed after the run; nothing
is written to the project. Every library that is not cached is measured by **one**
`dart` process.

```
$ x3 case lib/                 # a wrong expectation: red on its own line
lib/core/money.dart:5 (toMinor): example_failed
	out[0] = 120036, want 120037
x3 case: 13 example(s) in 0 package(s) - 12 passed, 1 Dart file(s), 1 finding(s)
```

**What is red, and why none of it is skipped:**

| Code | When |
|---|---|
| `example_failed` | `out=` differs, a proposition is false, or the call threw |
| `malformed` | the line breaks the grammar, or its expressions do not compile as Dart; the finding carries the compiler's words, and only that example is red - the rest are compiled again and run |
| `needs_flutter` | the library (or one it imports) reaches `dart:ui`, `dart:html` or `package:flutter`, which plain dart does not define |
| `does_not_build` | the library does not load for another reason (a missing import, a syntax error) |
| `no_runtime` | `dart` is not found, or the package map cannot be read; the finding names `case.dart.bin` or `dart pub get` as the remedy |
| `crashed` | dart ended, or reached `case.timeout`, before the example reported |

A library that needs Flutter is **unmeasurable, and says so**. Keep the pure
logic in a library that imports no Flutter and measure that one; the widgets
that call it stay outside this check.

**The cache follows the imports.** A library's answer is stored only when every
example passed, under a key that carries the runner, the `dart` binary and the
SDK version beside it, the digest of the package map, and the content of the
library **and of every file under the root that it imports or exports**,
transitively (relative and `package:` imports are followed; a package outside
the root is keyed by the package map's digest, which names its version). A key
on the library alone would show the old green after a helper it imports
changed.

Control experiment: the seven `case Dart control experiment` trials of the
`inline examples` gate step - a pure library (positional and named arguments, a
helper it imports, a list, a map, a `$` inside a string) green; a wrong
expectation red with the value it got; a false proposition red with its
position; an example that does not compile red alone beside a neighbour that
still runs; a library reaching `dart:ui` red; no `dart` red with the remedy.
Each arm was broken once and turned red: a runner that compared `2` and `2.0` as
equal kept `out=2` green for a `double`; a loop that did not drop the rejected
example failed the whole library instead of one line.

<!-- x3-dist version=v0.351.0 capabilities=21a88ea5aef392e7afd5f044ec88c56b1892a1be5387606caddf925abc6212a2 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
