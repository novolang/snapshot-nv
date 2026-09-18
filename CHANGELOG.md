# Changelog

All notable changes to snapshot-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
