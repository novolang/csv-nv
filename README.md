# csv-nv

## What this is

RFC 4180 comma-separated values, read and written by a package that
performs nothing itself. The reader is a feed-and-drain state machine:
you hand it whatever bytes you have, it hands back the records those
bytes completed and the reader to feed next. The writer turns records
into bytes and you decide where they go. A `Dialect` value carries the
eight decisions that separate one comma-separated file from another —
semicolons, tabs, backslash escapes, comment lines — and every error
names the line and the column a person has to go and look at.

It is for the program that reads a data file before it can do anything
else: a report tool, an importer, a fixture loader, an embedded logger
replaying its own output.

```
novo pkg add csv-nv
novo pkg build
novo test
```

## The one example

```novo
use reader
use record
use dialect

fn summarise(text: Str) -> Result<Float, CsvError>
    let rows = reader.parse(text, dialect.rfc4180())!
    var total = 0.0
    for row in rows
        total = total + record.float_at(row, 1)!
    Ok(total)
```

The header row is consumed by the dialect, so `rows` is the data. A
field that is not a number stops the fold with a `NotAFloat` that
carries the line, the column and the text that was there.

## The layer, and why

`core` — no effects at all, on a package whose whole subject is a
file.

That is not a contradiction, it is the design. A CSV scanner is
arithmetic over bytes the caller already holds: nothing is opened,
nothing is waited for, and the state a record carries between chunks is
a few integers and two strings. The host owns the file; this package
owns the grammar.

The one function that meets a stream stays inside the budget by
**binding** its cost rather than spending one:

```novo
pub fn read_all<S: Read[e]>(src: S, d: Dialect) -> Result<[Row], CsvError> [e]
```

`S: Read[e]` binds the effect parameter of the standard library's
`Read` trait and the clause uses it, so the row means *whatever the
impl behind `S` supplies*. A file charges its caller `[io]`; an
in-memory buffer charges nothing; `csv-nv` is charged neither, because
the impl that supplies the effect is declared where the host is.

## The load-bearing interface

```novo
pub fn reader(d: Dialect) -> Reader
pub fn feed(r: Reader, chunk: Bytes) -> (Reader, [Row])
pub fn finish(r: Reader) -> Result<[Row], CsvError>
```

Three calls, and everything else in the package is either a value they
take, a value they return, or a convenience written in terms of them.
`feed` returns a NEW reader rather than mutating one because `[mutate]`
is a host effect and this package has none — which is also what makes a
reader safe to keep, fork, or feed from two places.

A chunk may split anything: a field, a quoted field containing a
newline, a CRLF between the CR and the LF. Whatever a chunk could not
finish is carried in the reader and completed by the next one.
`read_all` is this loop written once, inside the package, for the
callers who just have a file.

## The reference implementation

Rust's `csv` crate for the reader's shape — its `Reader`, its
`StringRecord`, its `ByteRecord`, and above all its insistence that a
CSV reader is a state machine you can drive a byte at a time. Python's
`csv` module for the `Dialect` value and for the names of its fields.
RFC 4180 is the oracle, and `std.csv` — which this package does not
replace so much as give a position, a dialect and a stream to — is the
in-tree reference for what the same input should produce.

Deliberately not ported: `csv`'s Serde integration (`serde-nv` is where
that belongs), its `Trim` enum beyond the one boolean, and its
byte-record/string-record split — this package never decodes, so there
is one record type and its fields are bytes that were not checked.

## Status

Implemented, and `experimental`: the API tests were written against
the signatures before any body existed, so the suite is a description
of the interface that the bodies now have to agree with. `novo test`
runs it — 33 assertions over four suites — and `tests/differential.nv`
runs a thirty-document corpus through both engines so that `novo run
--interp` and the compiled binary can be compared byte for byte.

| module | functions | implemented |
| --- | --- | --- |
| `csverror` | 2 | yes |
| `dialect` | 11 | yes |
| `record` | 12 | yes |
| `reader` | 8 | yes |
| `writer` | 6 | yes |

### What it decides where RFC 4180 does not

The RFC describes a well-formed file and says nothing about the files
people have. `reader.nv`'s module comment is the full list; the short
version is that a blank line is not a record, a file may end without a
terminator, a lone CR ends a line, a bare quote in an unquoted field is
data under a lenient dialect and `BareQuote` under a strict one, and an
empty document is zero records rather than one empty one.
