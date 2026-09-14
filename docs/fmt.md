# The settings, written one way

[The pages](INDEX.md) - [what x3 is](../README.md)

## `x3 fmt`

**What it catches:** settings that drift into two writing styles, so a reader
cannot tell a rule from a habit.

```
x3 fmt [-config <file>] [-check] [file ...]
```

```yaml
fmt: { indent: 2 }
```

With no file named it formats the root configuration **and every part the root
declares**, so one command covers the whole configuration. `-check` writes
nothing: it names the files that are not formatted and exits `1`, which is what
a gate step runs.

### Nothing is lost, and that is enforced

A formatter that silently drops something is worse than none, because the loss
becomes invisible on the second run — the second run already works on top of it.
So before any file is written, the output is read back and compared with the
input two ways: **every value** must still be there, and **every comment** must
still be there, counted, so that dropping one of two identical sentences is a
loss too. A file that fails either check is **left untouched** and named.

A comment may still MOVE — a block written after the last key of a mapping is
re-indented under the list above it, and that is the writer's own ambiguity, not
a loss. What is refused is a comment that disappears.

A file that does not parse is refused with exit `2`, not passed over: calling an
unreadable configuration "formatted" is how a gate goes blind.

**Formatting is stable.** Running it twice produces the same bytes; the gate's
own experiment proves it with the formatter's output as a fixture.

<!-- x3-dist version=v0.215.0 capabilities=cd8fa8e546bc67b7323831ec5fdf30a6b1f6686c528907907a45a93054264111 template=43e4718d5f123011abedb1d713cc25a94efd0cee223243fab09b278510dd84c7 -->
