# snapshot-nv

A **snapshot test** stores the output of something in a file beside the
test, and every later run compares against the stored copy. When the
output changes, the test fails and a person reviews the difference and
either accepts it or fixes the code. The technique and its vocabulary
come from [insta](https://insta.rs) in Rust and
[syrupy](https://github.com/syrupy-project/syrupy) in Python. This
package brings them to novo-lang, over
[diff-nv](https://novo-lang.org/packages/diff-nv) for the rendering of a
mismatch.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A **snapshot** is a file holding one recorded value. It lives in a
`snapshots` directory beside the test source file, and its name is built
from that file and the name of the assertion, so there is one snapshot
per assertion name per source file. A **rejected snapshot** is a
`.snap.new` file written beside a stored one when the value differed; it
is what a person looks at during review.

The file is a header of two delimiter lines and then the value,
verbatim.

```
--- snapshot-nv 1
source: tests/render_tests.nv
assertion: a_wide_table
expression: tblrender.render(t, style)
redactions: timestamps, uuids
---
┌────────┬───────┐
│ name   │ cells │
└────────┴───────┘
```

The value runs from the byte after the second `---` line to the end of
the file. It is not escaped, quoted or length-prefixed, and its final
newline is kept exactly as it was. The header exists for a person
reading the file in a code review and takes no part in the comparison.

A **redaction** is a replacement applied to a value before it is
compared and before it is stored: a timestamp, a generated identifier,
a path with somebody's home directory in it. It is a label and a
function from the whole value to the whole value, so a redaction can be
structural — dropping a line, sorting a set — and not only textual.
Because redactions run before the write, the file on disk already reads
`created: [redacted]`, and adding one to a test shows up as a difference
in review.

The **update policy** says what a check may do when a snapshot is
missing or has changed: compare only, write a rejected file, record a
missing snapshot, or overwrite. It is a value the caller passes in, not
a variable read inside the assertion.

The comparison itself answers rather than fails. `snapassert.check`
reads the stored file, applies the redactions, compares, writes at most
one file according to the policy, and returns a **verdict** saying what
happened. Failing a test is a separate nine-line wrapper. That is what
lets one run hold two policies, lets a review tool use the same call as
the test does, and lets this package's own suite assert on the
interesting cases without catching a panic.

## Install

```
novo pkg add snapshot-nv
```

## Example

```novo
use std.test
use snapassert
use snapredact
use tblrender

@test
fn test_a_wide_table_renders() [fs, io]
    // The source file and the assertion name are written out, because
    // a running test cannot ask what it is called. See rule 9.
    let r = snapassert.with_redactions(
              snapassert.request("tests/render_tests.nv", "a_wide_table"),
              // Timestamps, identifiers and absolute paths, replaced
              // before the value is stored.
              snapredact.cli_defaults())

    // Compares against tests/snapshots/render_tests__a_wide_table.snap
    // and reports through std.test if it differs.
    snapassert.assert_snapshot(r, tblrender.render(fixture(), style()))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: snapshot-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `snappolicy` | The configuration: the four update modes, the directory, the two filename suffixes, the diff ceiling, the one function that reads the environment, and the three predicates that name what a mode allows. |
| `snapfile` | The stored format and the files: the header, the parser and the renderer, the path rules, reading and writing a snapshot and a rejected one, listing what is pending, accepting one, and the arithmetic of the unused-snapshot check. |
| `snapredact` | The replacements: the redaction type, the constructor a caller writes their own with, the nine named ones, the list a command-line tool usually needs, and the placeholder they all write. |
| `snapassert` | The comparison: the request, the five verdicts, the check that answers, the pure comparison under it, the assertion that reports, the readers over a verdict, and the two renderings over diff-nv. |
| `snapinline` | The expectation written in the test: the literal, the normalisation both ways, the three inline entry points, and the pending-edit file a review tool applies. |

## How to choose an entry point

**`snapassert.assert_snapshot` is what a test calls.** It runs the check
and reports through `std.test` on anything but a match.

**`snapassert.check` is the primitive and answers a verdict.** Take it
in a runner, in a review tool, or anywhere the consequence of a mismatch
is not "fail this test".

**`snapassert.agrees` is the comparison rule with no file in it.** It
takes the stored value and the actual one, applies the redactions and
compares. Use it when the stored value came from somewhere other than a
snapshot file, and to test comparison behaviour without a filesystem.

**`snapinline.assert_inline`, `.check_inline` and `.agrees_inline` are
the same three for an expectation written in the test itself.** See
rule 8.

**`snappolicy.config` builds the default configuration and the `with_*`
functions change one field each.** `snappolicy.policy_from_env` reads
the environment variable, and a runner calls it once. An assertion never
does.

## The rules a user needs

1. **The check answers; it does not fail anything.**
   `snapassert.check` returns a `SnapVerdict` with five cases:
   `SnapMatched`, `SnapRecorded(path)`, `SnapAbsent(path)`,
   `SnapDiffered(path, rejected, diff)` and `SnapFaulted(why)`.
   `snapassert.passed` is true for the first two, because recording a
   first snapshot is a pass.
2. **"Never recorded" and "changed" are different verdicts.**
   `SnapAbsent` is a snapshot that has never existed and `SnapDiffered`
   is one that moved. The update policy treats them differently, and a
   report that called both "failed" would make a first run
   indistinguishable from a regression.
3. **The update policy is an argument.** It lives on `SnapConfig`, which
   travels down from the caller, so one process can compare one suite
   and accept another.

   | `SnapUpdate` | Missing snapshot | Changed value |
   | --- | --- | --- |
   | `SnapNever` | fails, writes nothing | fails, writes nothing |
   | `SnapNew` (the default) | fails, writes the rejected file | fails, writes the rejected file |
   | `SnapAuto` | records it and passes | fails, writes the rejected file |
   | `SnapAlways` | records it and passes | overwrites it and passes |

   `SnapNever` is what continuous integration runs, so a snapshot
   somebody forgot to commit fails rather than appears. `SnapAlways`
   turns the whole suite into a recording; the only honest use is a bulk
   rename or a formatting change whose difference is reviewed in version
   control instead.
4. **One function in the package reads the environment.**
   `snappolicy.policy_from_env` reads the variable named by
   `snappolicy.env_name`, and it is the only `[io]` function outside the
   two assertions. Everything else takes the policy it was given.
5. **The value is stored verbatim, and a snapshot containing `---`
   round-trips.** The header ends at the second delimiter line and the
   rest of the file is the value, so a snapshot of a YAML document, of
   `key: value` lines or of another snapshot survives unchanged. The
   trailing newline is preserved exactly, because "the output grew a
   blank line" is a real regression.
6. **Nothing in the header takes part in the comparison.** Changing the
   expression text or adding a redaction label does not make a snapshot
   differ. Only the value does.
7. **Redactions run before the comparison and before the write, and in
   order.** `snapredact.apply_all` is that fold. The consequence is that
   the comparison is a plain string compare with no rules of its own, so
   there is exactly one place a redaction can be wrong.
   `snapredact.placeholder` is the one spelling they all write,
   `[redacted]`, so that a snapshot's own difference stays readable.
   `snapredact.matching` is the regular-expression escape hatch, and its
   label says less to a reader than the named ones do.
8. **An inline snapshot is two steps, and the second is a tool's.**
   novo-lang has no macro that can see its own call site, so no package
   can rewrite the source it was called from. `snapinline.check_inline`
   answers the same verdict and appends the edit to a `.pending-snap`
   file beside the test; a tool with an editor's rights applies it.
   Nothing here writes a `.nv` file.
9. **An inline literal is normalised before it is compared, and
   `snapinline.normalise_inline` is published because that is usually
   the answer.** It strips the newline after the opening delimiter, then
   the longest leading whitespace common to every non-empty line, then a
   trailing whitespace-only line. `snapinline.render_literal` is the
   inverse, and `snapinline.indent_of` is the column count a tool
   re-indents with.
10. **A running test cannot ask its own name or file, so the caller
    writes both.** `snapassert.request` takes the source path and the
    assertion name as text. Until `std.test` can answer them, a renamed
    test silently keeps its old snapshot.
11. **The assertion name becomes a filename, so it is checked.**
    `snapfile.name_ok` is false for an empty name and for anything
    carrying a path separator, and `snapfile.snapshot_path` answers
    `SnapBadName` for one.
12. **A file whose format version is newer than this build is refused,
    not parsed.** `SnapVersionAhead` carries the version.
    `snapfile.format_version` answers 1. A snapshot silently misread is
    a test that passes for the wrong reason.
13. **The rejected suffix is a field, not a derived string.** A
    repository's ignore rules name `.snap.new` literally, so
    `SnapConfig.new_suffix` is separate from `suffix` and
    `snapfile.is_rejected` is the predicate both a review tool and those
    rules need.
14. **A failure message carries at most `max_diff_lines` lines of
    diff**, 120 by default, with the count of what was cut on the end. A
    thousand-line difference in a test log buries every other failure in
    the run.
15. **The unused-snapshot check needs the whole run's set of names, so
    it is off by default.** Only a runner sees every name a run asked
    for. `snapfile.unused_names` is the arithmetic, waiting for that
    list, and `SnapConfig.require_all_used` is the flag a runner turns
    on.
16. **Accepting a rejected file over a snapshot that has changed
    underneath it is named.** `SnapStale` is a review tool's one real
    hazard, reported rather than resolved silently.
17. **Only the file operations touch a disk.** `snapassert.check` is one
    read and at most one write, with no environment, no clock and no
    terminal. `snapassert.agrees`, the whole file format, every render
    and every redaction declare no effects.

## What would have to change elsewhere

Three things this package needs are not in the toolchain yet, and are
named here rather than worked around.

`std.test` gives a running test no way to learn its own name or source
file. The equivalents of insta's macro would be `test.current_name()`
and `test.current_source()`, and they are what would remove the
repetition in every call site and stop a renamed test from keeping its
old snapshot.

`novo test --review` would walk the rejected files
(`snapfile.rejected_under`) and the pending inline edits
(`snapinline.pending_under`), show each difference and accept or reject
it (`snapfile.accept_rejected`, `snapfile.clear_rejected`). Everything
except the terminal is in this package already.

`novo test --update-snapshots` would set `SnapConfig.update` rather than
export an environment variable, and would hand the runner the set of
names the run used, which is the missing half of the unused-snapshot
check.

## What is not included

- **A diff algorithm.** diff-nv tokenises both sides by line, walks them
  and renders the unified form. `snapassert.render_diff` is three calls
  into it.
- **Rewriting a test's source.** See rule 8.
- **A test runner.** `std.test` is the harness.
- **Reading the environment inside an assertion.** See rule 4.
- **A serialisation format.** A value reaches this package as a `Str`.
  Rendering a structure into text is the caller's business, and it is
  what the header's `expression` field records.
- **Running on a microcontroller.** No such claim is made. The package
  reads and writes files.

## Related packages

- [diff-nv](https://novo-lang.org/packages/diff-nv) renders the
  difference between two texts. It is the only dependency, and the
  relationship is one-directional: a snapshot is text and a difference
  is arithmetic over text, so nothing here belongs there and nothing
  there needs a file. It is an interface release too, so this is an
  interface depending on an interface.
- [matchers-nv](https://novo-lang.org/packages/matchers-nv) judges a
  value against a named property and says what it wanted. Take it when
  the test is about one property of the value; take this package when
  the expected value is too large to write out.
- [httpmock-nv](https://novo-lang.org/packages/httpmock-nv) draws the
  same split between a call that answers and a wrapper that reports,
  for the same reason.
- [tempdir-nv](https://novo-lang.org/packages/tempdir-nv) is where a
  test that needs a directory of its own gets one, including a test of
  this package.
- `std.test` in the standard library is where `assert_snapshot` and
  `assert_inline` report. They are the only two functions here that
  touch it.

## Tests

```bash
novo test --isolate tests/snapinline_tests.nv   # 13 tests: the literal, the round trip, the pending file
novo test --isolate tests/snapfile_tests.nv     # 12 tests: the format, the paths and the refusals
novo test --isolate tests/snapredact_tests.nv   # 11 tests: the named redactions and the order
novo test --isolate tests/snapassert_tests.nv   # 10 tests: the verdicts and the comparison
novo test --isolate tests/snappolicy_tests.nv   #  7 tests: the four modes and the predicates
```

The behaviour asserted is insta's, for the file format, the update modes
and the review flow, and syrupy's, for the unused-snapshot check and the
per-run report.

Most of the suite needs no disk, which is the point of the verdict.
`snapassert_tests.nv` asserts through `agrees` and over verdict values:
that recording counts as a pass, that an absent snapshot and a changed
one are different answers, and that a diff is cut at the configured
ceiling. `snapfile_tests.nv` asserts the format by rendering and parsing
in memory, including a value that contains the delimiter and a value
whose final newline must survive. `snapredact_tests.nv` asserts that the
named redactions replace what they claim and that `apply_all` is ordered.
`snapinline_tests.nv` asserts the round trip: normalise what
`render_literal` produced and the value comes back.

The tests compile today and fail at run, each on the
`not implemented: snapshot-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `snappolicy.config`, `.with_update`, `.with_dir`, `.with_max_diff_lines`, `.with_require_all_used` | no |
| `snappolicy.policy_from_env`, `.policy_named`, `.update_name`, `.env_name` | no |
| `snappolicy.overwrites`, `.records_when_missing`, `.leaves_rejected` | no |
| `snapfile.format_version`, `.header`, `.with_expression`, `.with_redactions` | no |
| `snapfile.render`, `.render_into`, `.parse`, `.value_of` | no |
| `snapfile.snapshot_path`, `.new_path`, `.is_rejected`, `.name_ok` | no |
| `snapfile.read`, `.write`, `.write_rejected`, `.clear_rejected` | no |
| `snapfile.rejected_under`, `.accept_rejected`, `.stored_for`, `.unused_names` | no |
| `snapfile.SnapError.message` | no |
| `snapredact.placeholder`, `.redaction`, `.apply_all`, `.labels` | no |
| `snapredact.timestamps`, `.uuids`, `.hex_runs`, `.path_prefix`, `.field`, `.json_member` | no |
| `snapredact.matching`, `.pattern_ok`, `.unix_newlines`, `.trailing_space`, `.cli_defaults` | no |
| `snapassert.request`, `.with_config`, `.with_redactions`, `.with_expression`, `.named` | no |
| `snapassert.check`, `.agrees`, `.assert_snapshot` | no |
| `snapassert.passed`, `.verdict_name`, `.message`, `.diff_of`, `.path_of` | no |
| `snapassert.render_diff`, `.diff_summary`, `.summarise`, `.failure_count` | no |
| `snapinline.inline`, `.normalise_inline`, `.indent_of`, `.render_literal` | no |
| `snapinline.check_inline`, `.agrees_inline`, `.assert_inline` | no |
| `snapinline.pending_suffix`, `.pending_path`, `.record_pending`, `.pending_edits` | no |
| `snapinline.clear_pending`, `.render_pending`, `.parse_pending`, `.pending_under` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
