# Changelog

All notable changes to csv-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.3 — 2026-09-18

The documentation and comments in plain prose; no signature changed.

## 0.1.2 — 2026-09-15

The README written to the package README style guide
(docs/writing-a-readme.md); no change to the API.

## 0.1.1 — 2026-09-10

- **Toolchain floor is 0.8.9.**  The package uses what 0.8.9 added, a
  bound effect parameter and the four layers, and the manifest says so
  rather than letting an older toolchain fail on an undefined function.
  No signature changed.

## [0.1.0] — 2026-09-09

Reading and writing RFC 4180 over bytes the caller already holds: the
dialect value, the feed-and-drain reader, the typed reads over a
record, the writer, and the ten reasons a file is refused.

### Added

- The reader is one feed-and-drain state machine written once in
  `feed_str`; `feed` is that function over `bytes.to_str`, `parse` is
  it plus `finish`, and `read_all` is it with the pump inside and the
  caller's effect row bound. A chunk may split a field, a quoted
  newline, a CRLF between its two bytes, or an escape byte from the
  byte it escapes, and the reader carries every one of those into the
  next call.
- What the reader does where RFC 4180 is silent, written down in
  `reader.nv`'s module comment: a blank line is not a record, a lone CR
  ends a line, a bare quote in an unquoted field is data leniently and
  `BareQuote` strictly, text after a closing quote is the same defect
  from the other side, and an empty document is zero records.
- `tests/differential.nv` — a program, not a suite: it prints a corpus
  of thirty documents and the writer round trip, so that `novo run
  --interp` and the compiled binary can be compared byte for byte.

### Changed

- `Reader` gained four fields, each because a documented answer cannot
  be derived without it: `row_line`, because a `Row` carries the line
  the record started on and a quoted newline has already moved `line`
  on; `field_line` and `field_col`, because `UnterminatedQuote` reports
  the opening quote, which a scanner cannot recompute once it has read
  past it; and `width`, which is `RaggedRow`'s `want` under a strict
  dialect that has no header to take it from.
- `Reader.state` has seven values rather than five. The two extra are
  the ones a chunk boundary forces into existence. One is a CR whose LF
  may be in the next chunk, and one is an escape byte whose escaped
  byte may be.
- `csverror`'s module comment now says what a `col` is. The scanner's
  errors carry a byte offset into the line; the typed reads carry the
  1-based field number, because a `Row` is its fields and the line it
  started on and does not carry the line's text.
- `writer.write_rows`'s doc comment said two two-field records are 8
  bytes. They are 10, which is what its own test has always asserted.

### Fixed

- `novo test` is green: 33 assertions over four suites, both as
  compiled tests and — through `tests/differential.nv` — under the
  interpreter.

## [0.0.1] — 2026-09-09

The declarations: every public type and function with its full
signature, its effect row and its doc comment, and no function bodies.

### Added

- `dialect` — the eight decisions that separate one comma-separated
  file from another, as one value: delimiter, quote, escape byte,
  comment byte, header, trimming, strictness and a single-field bound.
  `rfc4180()` and `tsv()` are the two starting points, ten `with_*`
  builders change one field each, and `check` is the one call that
  refuses a combination the scanner could not read.
- `reader` — the feed-and-drain state machine. `feed` takes a chunk
  and returns the reader to feed next plus every record those bytes
  completed; `finish` is end of stream. A chunk may split a field, a
  quoted field containing a newline, or a CRLF between the CR and the
  LF. `read_all<S: Read[e]>` is the same machine with the pump inside
  it, charged whatever the caller's stream costs (SPEC § 5.6); `parse`
  is the whole document in one call.
- `record` — `Row` and `Header`, the field lookups by index and by
  header name, and the typed reads (`int_at`, `float_at`, `bool_at`
  and their by-name twins) that a caller uses to type a column.
- `writer` — records to bytes, terminated CRLF, with quoting applied
  where the dialect requires it and nowhere else. `needs_quote` and
  `quote_field` expose the rule for a caller producing a line this
  package does not.
- `csverror` — ten reasons, each carrying the line and the byte column
  a person has to go and look at, or `-1` where there is no such place.
