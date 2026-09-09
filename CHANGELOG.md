# Changelog

All notable changes to csv-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-09

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `dialect` — the eight decisions that separate one comma-separated
  file from another, as one value: delimiter, quote, escape byte,
  comment byte, header, trimming, strictness and a single-field bound.
  `rfc4180()` and `tsv()` are the two starting points, ten `with_*`
  builders change one field each, and `check` is the one call that
  refuses a combination the scanner could not read.
- `reader` — the feed-and-drain state machine, which is this package's
  load-bearing interface. `feed` takes a chunk and returns the reader
  to feed next plus every record those bytes completed; `finish` is
  end of stream. A chunk may split a field, a quoted field containing
  a newline, or a CRLF between the CR and the LF.
  `read_all<S: Read[e]>` is the same machine with the pump inside it,
  charged whatever the caller's stream costs (SPEC § 5.6); `parse` is
  the whole document in one call.
- `record` — `Row` and `Header`, the field lookups by index and by
  header name, and the typed reads (`int_at`, `float_at`, `bool_at`
  and their by-name twins) that a caller uses to type a column.
- `writer` — records to bytes, terminated CRLF, with quoting applied
  where the dialect requires it and nowhere else. `needs_quote` and
  `quote_field` expose the rule for a caller producing a line this
  package does not.
- `csverror` — ten reasons, each carrying the line and the byte column
  a person has to go and look at, or `-1` where there is no such place.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented: csv-nv.<fn>`.
  Run it with `--isolate` for one verdict per test naming the function
  it stopped at.
