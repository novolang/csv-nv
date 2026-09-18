# csv-nv

Comma-separated values (CSV) is a text format in which each line of a
file is one record and the fields of a record are separated by commas.
[RFC 4180](https://www.rfc-editor.org/rfc/rfc4180) is the specification
for it. This package reads and writes the format in novo-lang, with no
dependencies. Nothing here opens a file: the caller holds the bytes and
this package holds the grammar.

## What it is

A CSV **record** is one line of the file. Its **fields** are the pieces
between the separators. A field that contains the separator, a quote or
a line break is written inside double quotes, and a quote inside a
quoted field is written twice. RFC 4180 section 2 is the whole grammar,
and it is three paragraphs long.

The first record may be a **header**: one field name per column. RFC
4180 makes it optional, and a reader has no way to detect one, so it is
told.

RFC 4180 describes one format and the world ships a dozen. The
difference between them is small: a semicolon in place of a comma, a
backslash in place of a doubled quote, a `#` line the producer considers
a comment. This package puts those differences in one value, a
**dialect**, which is built once and handed to the reader and the
writer.

| Dialect field | Default | What it is |
| --- | --- | --- |
| `delim` | `0x2C` | The byte between fields |
| `quote` | `0x22` | The byte that opens and closes a quoted field |
| `escape` | `-1` | The byte that escapes a quote, or `-1` for RFC 4180's doubling |
| `comment` | `-1` | A byte that makes a whole line a comment, or `-1` for none |
| `has_header` | `true` | Whether the first record names the columns |
| `trim` | `false` | Whether to drop spaces and tabs around an unquoted field |
| `strict` | `false` | Whether to refuse what RFC 4180 forbids rather than read it |
| `max_field` | `1048576` | The largest single field in bytes, or `0` for unbounded |

Every byte in a dialect is an `Int` and not a one-character string. A
separator is one byte, and `-1` is the value that turns a feature off.

The reader is a **feed-and-drain state machine**. The caller hands it
whatever bytes it has, and it hands back the records those bytes
completed together with the reader to feed next. A chunk may split
anything: a field, a quoted field containing a newline, a CRLF between
its two bytes. Whatever a chunk could not finish is carried in the
reader and completed by the next one.

Where RFC 4180 leaves a choice, this package makes these ones.

| Decision | This package's answer |
| --- | --- |
| Record terminator written | CRLF, always |
| Record terminators read | CRLF, LF and a lone CR |
| Default field separator | `,` (`0x2C`) |
| Default quote byte | `"` (`0x22`) |
| Default quote escape | a doubled quote, RFC 4180 section 2.7 |
| Default field size bound | 1 048 576 bytes |
| Line numbers in an error | 1-based |
| Column numbers in an error | 1-based |

## Install

```
novo pkg add csv-nv
```

## Example

```novo
use std.bytes
use dialect
use reader
use record
use writer

fn main() [io]
    // A comma, double quotes, and a header row: RFC 4180 as written.
    let d = dialect.rfc4180()

    // Read a document that is already in memory. The header record is
    // consumed by the dialect, so `rows` holds the two data records.
    match reader.parse("item,price\nbolt,0.25\nnut,0.10\n", d)
        Err(e)   => println(e.message())
        Ok(rows) =>
            var total = 0.0
            for row in rows
                // Read the second field as a number. A field that is
                // not one answers an error naming its line and column.
                match record.float_at(row, 1)
                    Err(e2) => println(e2.message())
                    Ok(p)   => total = total + p
            println("${total}")

            // Write one record back out. The writer answers bytes and
            // the caller decides where they go.
            let (w, line) = writer.write_row(writer.writer(d), ["total", "${total}"])
            println(bytes.to_str(line))
```

It prints `0.35` and then `total,0.35`.

Build and test with:

```bash
novo pkg build                  # the package compiles
novo test tests/reader_tests.nv # and the other three suites
```

## What the package contains

| Module | Contents |
| --- | --- |
| `dialect` | The eight decisions that separate one comma-separated file from another, as one value, with a constructor for RFC 4180 and one for tab-separated files. |
| `reader` | The scanner: a reader fed a chunk at a time, a whole-stream call, and a whole-string call. |
| `record` | A record and a header, and the reads over them: by index, by column name, as text, as an integer, as a number, as a boolean. |
| `writer` | Records to bytes: one record, a header, a list of records, and the quoting rule on its own. |
| `csverror` | Every way a file can refuse to be read, each with the line and the column to send a person to. |

## How to choose an entry point

**`reader.parse` takes the whole document as text.** It answers the
records or the first error. Use it when the file is already in memory.

**`reader.read_all` takes a stream.** It is declared
`read_all<S: Read[e]>(src: S, d: Dialect) -> Result<[Row], CsvError> [e]`,
so it costs the caller whatever the caller's stream costs: a file
charges `[io]`, an in-memory buffer charges nothing. Use it when you
have opened something and want the records.

**`reader.reader`, `reader.feed` and `reader.finish` take chunks.** The
caller pumps bytes in and takes records out. Use it when the bytes
arrive from somewhere the other two do not fit: a frame decoder, a
decompressor, a network read at a time.

**`writer.write_row` and `writer.write_rows` answer bytes.** The caller
writes them wherever they go. `writer.write_header` puts the column
names back at the top, and `writer.quote_field` spells one field the way
the file needs it.

## The rules a user needs

1. **The header is consumed and not returned.** Under a dialect with
   `has_header` true the first record becomes the header.
   `reader.header_row` answers it, `record.header` turns a record into
   one, and `record.names` lists the column names. RFC 4180 section 2.3.
2. **A file may end without a terminator, and `finish` is where that
   last record comes out.** RFC 4180 section 2.2 makes the final CRLF
   optional. A caller that stops feeding without calling `finish` loses
   the last record.
3. **A blank line is not a record.** A line whose terminator arrives
   with nothing open and nothing finished is skipped. A line holding one
   empty quoted field is a record of one empty field, because the quotes
   say so. An empty document is zero records.
4. **A lone carriage return ends a line.** The terminator in RFC 4180
   section 2.1 is CRLF, and a file that lost its line feeds is still a
   file.
5. **A bare quote inside an unquoted field is data under a lenient
   dialect and an error under a strict one.** RFC 4180 section 2.5
   forbids it and says nothing about what to do. Text after a closing
   quote is the same defect from the other side, and `strict` reports
   both as `BareQuote`.
6. **A record of the wrong width is an error only under a strict
   dialect.** `RaggedRow` carries the header's width and this record's.
   Leniently, the widths vary and `record.width` is how a caller checks.
7. **A field is never decoded.** The bytes come out of the file with the
   quoting removed and are handed back as a `Str` without being checked
   as UTF-8. A file in any byte encoding therefore reads, and a caller
   who needs UTF-8 validation does it on the fields it cares about.
8. **Types are asked for, one column at a time.** Nothing is inferred.
   `record.int_at`, `float_at` and `bool_at` take an index; `int_of`,
   `float_of` and `bool_of` take a header and a column name.
   `bool_at` accepts `true`, `t`, `yes`, `y` and `1`, and the matching
   five for false, in any case.
9. **A field is bounded at 1 MiB by default.** The bound is what keeps
   one unclosed quote in a hostile file from accumulating the whole file
   into one field. Past it the reader answers `FieldTooLong`.
   `dialect.with_max_field` moves the line and `0` removes it.
10. **The column in an error means one of two things.** For the
    scanner's errors, `UnterminatedQuote`, `BareQuote` and
    `FieldTooLong`, it is a 1-based byte offset into the line. For the
    typed reads it is the 1-based field number, because a record does
    not carry the text of its line. `csverror.line_of` and `col_of`
    answer `-1` where there is no position.
11. **`UnterminatedQuote` reports the opening quote.** That is the byte
    a person has to go and fix, and it is the one position a scanner
    cannot recompute after reading past it.
12. **`feed` answers a new reader rather than changing the one you
    gave it.** The state is a value, so a reader can be kept, copied or
    fed from two places. The writer is threaded the same way.
13. **A reader that has failed stays failed.** `reader.error` answers
    the reason, and every later `feed` emits nothing.
14. **A contradictory dialect is refused before a byte is read.**
    `dialect.check` names the combinations that make the scanner
    ambiguous: the separator equal to the quote, the escape or the
    comment byte, the quote equal to the escape byte, any of them
    outside `-1` to `255`, or a negative field bound. `reader.reader`
    and `writer.writer` both call it, and a reader built from a bad
    dialect is already failed.
15. **The writer always terminates a record with CRLF.** RFC 4180
    section 2.1. The line ending is not a dialect field, so the output
    is predictable from the dialect a caller was handed.
16. **A field is quoted when it holds the separator, the quote byte, a
    carriage return or a line feed, and not otherwise.**
    `writer.needs_quote` is that rule on its own.
    `reader.parse` over the writer's output, under the same dialect,
    gives back the records that went in.
17. **A writer that cannot write emits nothing.** Three cases answer an
    empty byte string and an unchanged writer: a dialect `check`
    refuses, a header after the first record, and, under a strict
    dialect, a record of a different width from the first. There is no
    error channel on the pair a writer answers, so a refusal shows as a
    writer whose record count did not move.
18. **`read_all` stops at the end of the stream, and discards what it
    read if the stream failed.** An empty read is the end, and a source
    reporting itself closed is the end. Any other failure is
    `Transport`, which carries the host's own `IoError` so a caller can
    still tell a timeout from a truncated file. A caller who wants the
    prefix of a stream that failed drives `feed` and keeps what it was
    handed.

## What is not included

- **Any input or output.** No function here opens, reads or writes
  anything. The reader takes bytes and the writer answers them.
- **Decoding.** See rule 7. A caller who needs UTF-8 validation does it
  on the fields it cares about.
- **Type inference.** See rule 8.
- **A line-ending choice for the writer.** See rule 15.
- **A separate byte record and text record.** Rust's `csv` crate has
  both. This package never decodes, so there is one record type and its
  fields are bytes that were not checked.
- **Trimming beyond one boolean.** `trim` drops spaces and tabs from an
  unquoted field. RFC 4180 section 2.4 says the space is part of the
  field, so the default is to keep it.
- **A second index for a repeated column name.** `record.column`
  answers the first index a name appears at. A caller who needs a later
  one reads `record.names` and searches it.
- **Serialization of novo-lang structs.**
  [serde-nv](https://novo-lang.org/packages/serde-nv) is where that
  belongs.

## Related packages

- `std.csv` in the standard library reads a whole document in one call.
  It has no dialect, no position in its errors and no way to take a
  stream a chunk at a time. This package is the one to reach for when
  the file is not quite RFC 4180, when a person has to be told where the
  fault is, or when the document does not fit in memory.
- [serde-nv](https://novo-lang.org/packages/serde-nv) serializes
  novo-lang structs and enums to JSON, MessagePack and CBOR. A record
  here is a list of fields, which is what a tabular file has; a struct
  is what serde-nv has.
- [toml-nv](https://novo-lang.org/packages/toml-nv),
  [yaml-nv](https://novo-lang.org/packages/yaml-nv) and
  [ini-nv](https://novo-lang.org/packages/ini-nv) are the configuration
  formats on the registry. They carry nesting and types; a
  comma-separated file is a table of text.

## Tests

```bash
novo test tests/dialect_tests.nv     #  7 tests: the value and its refusals
novo test tests/reader_tests.nv      # 11 tests: the scanner and the chunk boundaries
novo test tests/record_tests.nv      #  8 tests: the reads over a record
novo test tests/writer_tests.nv      #  7 tests: the quoting and the round trip
```

RFC 4180 is the oracle. Rust's `csv` crate is the reference for the
reader's shape and Python's `csv` module for the dialect's field names.

The suite asserts that a chunk may split a quoted field, a quoted
newline and a CRLF between its two bytes, that a blank line is not a
record, that a file may end without a terminator, that a lone carriage
return ends a line, that a bare quote is data leniently and an error
strictly, that a field past the bound is refused, that every error
carries the position the module comment promises, and that reading the
writer's output under the same dialect gives back the records that went
in.

`tests/differential.nv` is a program rather than a suite. It prints a
corpus of thirty documents, so that the interpreter and the compiled
binary can be compared byte for byte:

```bash
novo run --interp tests/differential.nv > a.txt
novo run          tests/differential.nv > b.txt
cmp a.txt b.txt
```

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
