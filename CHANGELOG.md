# Changelog

All notable changes to snapshot-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Changed

- The header reader and the update-mode parser are `match` expressions, and the private searches for the value offset and the common indent use an optional rather than `-1`; no public signature or behaviour changes.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-12

**The implementation.** Every body under `src/` landed against the
interface's own suites, which were not asked to move: a body is the
answer the assertions in `tests/` were written for, and the one
assertion that cannot be satisfied is left red and named under `###
Known` rather than edited. `stability` moves to `experimental`, the
version to `0.1.0`, and both `README.md` and `novo.toml` lose the
interface-release text the bodies made false.

The package turned out to be a **value** snapshot model and not a
filesystem one. It records the text a caller produced — a header, two
delimiter lines, then the value verbatim — and no attribute of the
source file is written down: not its mtime, not its size, not a hash,
not its permission bits. Nothing in the header takes part in the
comparison either. Both facts are pinned in the new suite, because they
are what decides whether a snapshot survives a checkout, a copy or a
reformat, and because the one place the filesystem's clock is read at
all — accepting a `.new` over a snapshot written after it — is a hazard
worth naming rather than a comparison rule.

A rename is *not* a kind of verdict here. The five cases are matched,
recorded, absent, differed and faulted, and the difference is a line
diff, so a file that moved is a removal and an addition. That is pinned
as such, so that a later change starting to pair a delete with an add
has to say so.

### Added

- `tests/snapshot_nv_edge_tests.nv` — the one suite that holds files,
  and the four statements the other five have no place for: the three
  diff cases over one pair (nothing changed, one thing changed, and one
  gone with one arrived so that the line count, the byte count and the
  number of members are all unchanged and a comparison that counted
  instead of comparing would answer that nothing happened); a rename
  reported as a delete and an add; that nothing in the header and no
  attribute of the source file takes part in the comparison, and that
  the comparison itself is exact — no trailing space, no case and no
  final newline normalised away, because a redaction is the only
  transformation there is; that an empty value and a later reading of
  an unchanged tree are both clean; the four update modes on real
  files; and the pending-edit file through a real disk, including every
  malformed record being refused at its own line.
- `3` `cov: skip` regions under `src/`, two of them excusing the
  reporting path of an inline assertion — which runs only when an
  assertion fails, and so cannot be reached by a passing suite — and
  one excusing `snapinline.ends_blank`'s empty-list branch, which
  `str.split_lines` makes unreachable by answering at least one line for
  every text. Everything else under `src/` is covered by a suite.

### Changed

- `novo.toml` — `0.1.0`, `stability = "experimental"`, the `novo` floor
  to `>= 0.20.2` (the version verified against), the interface-release
  tail dropped from `description`, and the `diff-nv` comment corrected:
  the registry's diff-nv 0.1.0 has bodies in all five of its modules, so
  the three calls into it answer rather than panic, and the rendering
  here is exercised rather than only type-checked.
- `README.md` — the `**Status: NOT IMPLEMENTED**` block and the
  implementation-status table are gone, with nothing written in their
  place, and the three sentences the bodies made false are rewritten:
  the "`novo test` fails on purpose" paragraph, the "interface
  depending on an interface" line about diff-nv, and the closing
  paragraph about tests that "turn green one at a time".
- **`snapassert.no_stored` recorded nothing for a missing snapshot under
  a policy that leaves the evidence.** README rule 3's table and
  `SnapNew`'s own doc both say the default writes the `.new` beside the
  snapshot when it is missing, "so a person can look at what it would
  have been"; the body answered the absence and wrote nothing.  The
  verdict is still the absence — `SnapAbsent` carries the snapshot's
  path and has no rejected one to carry — and the value now lands
  beside it.
- **`snapinline.record_pending` created a headerless file.** Appending
  to a path that was not there wrote the record with no header, and
  `parse_pending` refuses a first line that is not the header, so a file
  this package wrote could only be answered a fault by this package's
  own `pending_edits` — and a review walk that had told a tool to open
  it was pointing at something unreadable.  A first append now writes
  the header too.
- `22` `redundant-bool` sites in the released suites, all of them
  pre-existing (`git show HEAD:<file>` counts the same sites before this
  release), rewritten to the linter's own suggestion: `x == false` to
  `not x`.  No predicate changed.

### Known

- **One assertion in the released suite cannot be satisfied.**
  `tests/snapassert_tests.nv`'s
  `test_the_reporting_wrapper_is_the_only_io_in_the_module` calls
  `assert_snapshot` against
  `tests/snapshots/render_tests__a_wide_table.snap`, and no such file is
  in the release: the tarball carries `src/`, `README.md`,
  `CHANGELOG.md` and `LICENSE`, never `tests/`.  The assertion can only
  pass in a working tree where a run has recorded that snapshot, so it
  is left red with the diagnosis here rather than an edit to the
  assertion.  Measured after the bodies landed:
  `novo test --isolate tests/snapassert_tests.nv` is 9 passed and 1
  failed, on `no snapshot at
  'tests/snapshots/render_tests__a_wide_table.snap': re-run with
  UPDATE_SNAPSHOTS=auto to record it`, and `novo test src/snapassert.nv`
  compiles 17 doc examples with 16 passing and the same file missing
  from the one that fails.
- **The publish dry-run is green and its own report is not all green.**
  `novo pkg publish --dry-run --allow-dirty` exits `0`, and the coverage
  line it prints is `99% of 491 instrumented line(s) under src/, over 6
  suite(s); 3 region(s) excused by cov: skip`, naming the three regions
  above at `src/snapinline.nv:135`, `:431` and `:454`. It also prints
  `the shard rows that did not pass: test`, and `test` is that
  assertion, not the format, the coverage or the linting.
- **`novo test --review` and `--update-snapshots` are named and not
  filed.** Everything they need except the terminal is in this package,
  and the pending-edit file now round-trips end to end, which is what
  the review half was missing.

## 0.0.3 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- diff-nv: `^0.0.1` to `^0.1.0`.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md).

### Fixed

- The README's example stands on its own.  It opened `use tblrender` —
  a module of table-nv, which this package does not depend on and has
  no reason to — and then called `fixture()` and `style()`, which were
  declared nowhere.  `novo doc` could not resolve the `use`, so the
  block failed to compile and `novo pkg publish` refused the release
  over it.  The value under test is now a `Str` the example produces
  itself, which is also the more accurate picture: this package never
  sees anything but the text the code under test returned.  No
  dependency was added — the closure is still unicode-nv and diff-nv.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `snappolicy` — `SnapUpdate` (never, new, auto, always) and
  `SnapConfig`; `policy_from_env` as the package's one environment
  read, `policy_named` as its pure half, and the three predicates
  (`overwrites`, `records_when_missing`, `leaves_rejected`) that keep
  the dangerous mode named in one place.
- `snapfile` — `SnapHeader`, `SnapStored`, `SnapError` and
  `impl Error for SnapError`; `render`, `render_into` and `parse` at
  `[]`; `snapshot_path`, `new_path`, `is_rejected`; `read`, `write`,
  `write_rejected`, `clear_rejected`, `rejected_under`,
  `accept_rejected`, `stored_for`; and `unused_names`, the arithmetic
  half of the unused-snapshot check.
- `snapredact` — `SnapRedaction` as a label and a named function;
  `apply_all` in order; and the named redactions `timestamps`, `uuids`,
  `hex_runs`, `path_prefix`, `field`, `json_member`, `matching`,
  `unix_newlines`, `trailing_space`, with `cli_defaults` as the list a
  tool's output usually needs.
- `snapassert` — `SnapVerdict` and `SnapRequest`; `check` at `[fs]`,
  `agrees` at `[]`, `assert_snapshot` at `[fs, io]`; `render_diff` and
  `diff_summary` over diff-nv; and `summarise` / `failure_count` for a
  runner's report.
- `snapinline` — `SnapInline` and `SnapPendingEdit`;
  `normalise_inline`, `indent_of` and `render_literal` as the round
  trip; `check_inline`, `agrees_inline`, `assert_inline`; and the
  pending file — `record_pending`, `pending_edits`, `clear_pending`,
  `render_pending`, `parse_pending`, `pending_under`.

### Known

- **The assertion answers a `SnapVerdict`.** A library whose only entry
  point fails a test cannot be tested by itself, forces a review tool
  into a second code path, and cannot produce a per-run report.
- **The update policy is an argument, not an environment variable.**
  `policy_from_env` is one function at the edge; `check` is `[fs]`
  because of it, and one run can hold two policies.
- **The stored file is a two-delimiter header and then the value
  verbatim**, so a snapshot containing `---` or `key: value` lines
  round-trips, and the trailing newline is preserved exactly.
- **Redactions run before the value is stored**, so the file on disk is
  stable, the comparison has no rules of its own, and adding a
  redaction shows up as a diff in review.
- **Inline snapshots are two steps** — this package records a pending
  edit, a tool applies it — because novo-lang has no macro that can see
  its call site.
- **`std.test` cannot tell a running test its own name or file**, so
  `SnapRequest` takes both as text. `test.current_name()` and
  `test.current_source()` are what would remove the repetition, and
  without them a renamed test silently keeps its old snapshot.
- **`novo test --review` and `--update-snapshots` are named and not
  filed.** Everything they need except the terminal is in this package.
- **One dependency, diff-nv**, for the mismatch rendering only.
