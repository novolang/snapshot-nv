# snapshot-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Snapshot assertions for novo-lang — [insta](https://insta.rs) and
[syrupy](https://github.com/syrupy-project/syrupy)'s idea: store the
output of something beside its test, compare against the stored copy,
and review the change when it differs.

- `snappolicy` — `SnapUpdate` and `SnapConfig`: what a mismatch is
  allowed to do, as a value;
- `snapfile` — the stored format and the four files it touches;
- `snapredact` — the volatile fields, replaced before they are stored;
- `snapassert` — `check`, which answers, and `assert_snapshot`, the
  nine lines over it;
- `snapinline` — the expectation written in the test, and the pending
  file a review tool applies.

```
novo pkg add snapshot-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use std.test
use snapassert
use snapredact
use tblrender

@test
fn test_a_wide_table_renders() [fs, io]
    let r = snapassert.with_redactions(
              snapassert.request("tests/render_tests.nv", "a_wide_table"),
              snapredact.cli_defaults())
    snapassert.assert_snapshot(r, tblrender.render(fixture(), style()))
```

The first run under `UPDATE_SNAPSHOTS=auto` records
`tests/snapshots/render_tests__a_wide_table.snap`. Every later run
compares against it, and a change leaves a `.snap.new` beside it with
the unified diff in the failure message.

## The load-bearing interface: `SnapVerdict`

```novo ignore
pub enum SnapVerdict
    SnapMatched
    SnapRecorded(path: Str)
    SnapAbsent(path: Str)
    SnapDiffered(path: Str, rejected: Str, diff: Str)
    SnapFaulted(why: snapfile.SnapError)
```

`snapassert.check` compares and **answers**. It fails no test, prints
nothing, and decides nothing about what a mismatch means.
`assert_snapshot` is the small wrapper that reports through `std.test`,
and it is one of only two `[io]` functions in the package.

Every other snapshot library makes the assertion the primitive, and
pays for it three times.

**The library cannot be tested by the library.** "A mismatch under
`SnapNever` writes nothing" is a statement about a verdict. Where the
only entry point fails the test, asserting that means catching a panic
and reading a message — so the interesting cases go untested in exactly
the library whose job is to make other people's output testable.

**A review tool needs a second code path.** `novo test --review` wants
the same comparison with a different consequence. With a verdict it is
the same call; without one it is a reimplementation that drifts.

**A runner cannot aggregate.** "Eleven snapshots changed, three are new"
is the report a person acts on, and it is unreachable if the first
mismatch ends the process.

### The other half: the policy is an argument

insta reads `INSTA_UPDATE` from the process environment **inside the
macro that does the assertion**. That makes every assertion `[io]` for
a variable that is constant for the whole run, makes it impossible to
compare one suite and accept another in one process, and makes a test
of the library depend on global state.

Here `SnapConfig.update` travels down from the caller.
`snappolicy.policy_from_env` is one `[io]` function at the edge that a
runner may call once; an assertion never calls it.

Together those two decisions make `snapassert.check` `[fs]` — a file
read, at most one file written, no environment and no terminal.

## The stored file

```
--- snapshot-nv 1
source: tests/render_tests.nv
assertion: renders_a_wide_table
expression: tblrender.render(t, style)
redactions: timestamps, uuids
---
┌────────┬───────┐
│ name   │ cells │
└────────┴───────┘
```

**Two delimiters and not one.** The value runs from the byte after the
second `---` line to the end of the file, unparsed and unescaped. A
snapshot whose value contains `---`, or `key: value` lines, or a whole
YAML document, round-trips — which matters because the things people
snapshot are exactly the things that look like other formats.

**A snapshot is read by people**, in a code review, next to the change
that altered it. That is why the value is in the clear: the diff in the
review *is* the diff of the thing under test. No escaping, no quoting,
no length prefix — and the trailing newline preserved exactly, because
"the output grew a blank line" is a real regression that a normalising
format would hide.

**The header is metadata and never part of the comparison.** Changing
the expression text or adding a redaction label does not make a
snapshot differ; only the value does.

## Redactions run before the value is stored

A snapshot of anything real contains something that changes — a
timestamp, a generated id, a path with somebody's home directory in it.
The redaction is applied **before** the value is compared and before it
is written, so the file on disk already reads `created: [redacted]`.

Three things follow. The stored file is stable, so a reader in review
sees the redaction rather than a value about to change. The comparison
is a plain string compare with no rules of its own, so there is exactly
one place a redaction can be wrong. And adding a redaction to a test
changes the stored file, which shows up in review — where a
compare-time rule would change what passes with no diff at all.

A `SnapRedaction` is a label and a **named function**
(`fn(Str) -> Str` in a struct field — matchers-nv's arrangement, for
the same reason: SPEC § 3.6's bounds carry an effect argument and never
a type one). The named ones — `timestamps`, `uuids`, `hex_runs`,
`path_prefix`, `field`, `json_member` — say what they are about in the
header; `matching` is the regular-expression escape hatch and is last
on purpose, because a header line reading `regex` tells a reader
nothing.

## Inline snapshots are two steps, and say so

insta rewrites the literal in the test with a procedural macro that can
see its own call site. **novo-lang has no macro that can see its call
site**, so a package cannot rewrite the source it was called from — and
should not want to.

So `snapinline.check_inline` answers the same `SnapVerdict` and
`record_pending` appends the edit to a `.pending-snap` file beside the
test; a tool with an editor's rights applies it. Nothing in this package
writes a `.nv` file.

The one thing that has to be normalised is the indentation — the
literal in the test is indented to the code around it and the value is
not — and `normalise_inline` is published as a function rather than
hidden in the comparison, because "why did my inline snapshot not
match" is almost always its answer.

## What `novo test` would have to expose

Named here, filed nowhere.

**A test's own identity.** `std.test` gives a running test no way to
learn its name or its source file, so `SnapRequest` takes both as text
and every call site repeats them. The equivalent of insta's macro would
be `test.current_name()` and `test.current_source()`. Without them, a
renamed test silently keeps its old snapshot.

**`novo test --review`** — walk the `.new` files
(`snapfile.rejected_under`) and the pending ones
(`snapinline.pending_under`), show each diff, accept or reject
(`snapfile.accept_rejected` / `clear_rejected`). Everything except the
terminal is in this package already.

**`novo test --update-snapshots[=never|new|auto|always]`**, setting
`SnapConfig.update` rather than exporting an environment variable, so
one run can hold two policies.

**The used-name set.** The unused-snapshot check needs every name the
run asked for, and only the runner sees all of them.
`snapfile.unused_names` is the arithmetic, waiting for the list.

## The layer

`host`, and the row that matters is the one that is not here.

| what | row |
| --- | --- |
| reading a snapshot, writing a `.new`, listing pending ones | `[fs]` |
| `snappolicy.policy_from_env` and the two `assert_*` reporters | `[io]` |
| the file format, every comparison, every render, every redaction | `[]` |

## One dependency

diff-nv, for the mismatch rendering and nothing else:
`snapassert.render_diff` tokenises both sides by line, walks them and
renders unified — three calls and no algorithm of this package's own.
diff-nv is `core`, which the layer order allows, and it is itself an
interface release today, so this is an interface depending on an
interface.

## Reference implementation

[insta](https://insta.rs) (Rust) for the file format, the update modes
and the review flow; [syrupy](https://github.com/syrupy-project/syrupy)
(Python) for the unused-snapshot check and the per-run report. The
departures from both are the verdict, the policy as an argument, and
inline snapshots as two explicit steps.

## Status

Interface only. Every body is `todo()`; `novo pkg build` type-checks and
effect-checks the whole surface, and `novo test tests` is red until the
bodies land.
