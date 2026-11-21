# csv — a CSV query/analytics CLI written in Memelang

A beta demonstration of [Memelang](https://github.com/...) doing a classic
data-engineering assignment — CSV parsing, joins, group-by, aggregation —
100% in Memelang (Structured English), compiled offline with the `heuristic`
provider. All aggregation is integer-only; averages are rendered with 2
decimals at print time.

## Layout

| File | Purpose |
|---|---|
| `main.meme` | thin argv router (`function main` + command handlers) |
| `csvparse.meme` | RFC-4180-lite CSV parser and row renderer |
| `csvstore.meme` | table registry: `Table(name, content)` persisted to `data/tables.json` |
| `csvquery.meme` | count/sum/avg/min/max, group-by, join, where, column lookup |
| `tests/meme/*.meme` | test blocks run with `meme test` (41 cases) |

Test files import the root modules with relative parent paths
(`use ../../csvparse`, `../../csvstore`, `../../csvquery`) — Memelang
resolves `use` relative to the importing file's directory, so no symlinks
are needed.

## Build & run

```bash
uv run --project /path/to/meme meme build main.meme -o csv_bin   # native binary (C backend)
./csv_bin load demo/sales.csv sales
uv run --project /path/to/meme meme compile main.meme --target python -o csv.py
python3 csv.py tables
```

## CLI

```
csv load <file> <name>                 # register a table (persisted in data/tables.json)
csv tables                             # registered tables: name, rows, cols (sorted)
csv head <name> [n]                    # header + first n data rows (default 5)
csv count <name> [column value]        # row count, or rows where column=value
csv sum|avg|min|max <name> <column> [where column value]
csv groupby <name> <group-col> <agg>(<col>)   # agg: sum|avg|min|max|count
csv join <a> <b> <column>              # inner join on equal column values
csv where <name> <column> <value>      # filtered rows (header + rows, CSV out)
```

Unknown command: full usage, exit 1. Every error is a single `Error: ...`
line on stderr followed by exit code 1 (usage on failed invocation is also
printed to stderr). Successful commands exit 0.

## Semantics

- **CSV dialect (RFC-4180-lite).** Comma-separated; fields may be double-quoted;
  a quoted field may contain commas, escaped quotes (`""`), and embedded
  newlines. Both `\r\n` and `\n` line endings are accepted; a lone `\r` also
  ends a line. Completely blank lines are skipped. The parser is hand-written
  in-language (a character state machine), not a naive split. Tolerance rules:
  a quote only opens a quoted field at the *start* of a field — a mid-field
  quote is kept literally (so a stray inch-mark never merges rows); text after
  a closing quote is kept; an unterminated quote at end of input is closed
  silently. Control characters below U+0020 (except `\t`) are rejected at load
  with a clean error, so the registry can never hold JSON-breaking bytes.
- **Load.** Ragged rows (rows whose width differs from the header) are rejected
  with one error line and exit 1. Re-loading a name replaces the stored table.
  Table names may not contain `/`.
- **Persistence.** `data/tables.json` is a JSON array of
  `{"name": ..., "content": ...}` objects; `content` is the raw CSV text.
  Rows are parsed in-language on demand. A corrupt or missing registry is
  tolerated: it reads back as no tables (and the next `load` rewrites it).
  (Memelang's JSON record serialization cannot store list fields, and its
  writer fails silently on string fields containing a literal `"` — hence the
  hand-written JSON with an in-language escaper and the built-in JSON reader.)
- **Aggregation.** Cells must be integer text (optional `-`, digits, surrounding
  whitespace); anything else fails with `Error: column '<x>' has non-integer
  value '<cell>'`. `sum` over zero matching rows is `0`; `min`/`max` over zero
  rows are an error; `avg` over zero rows is an error. `avg` prints two decimals
  (`7.80`), rounded half away from zero via the built-in
  `format float of <mean> with 2 decimals`. A negative mean whose magnitude
  rounds to zero prints `0.00` (no negative zero; the old integer formatter
  printed `-0.00` in that corner).
- **where chains are single-condition only.** `sum sales amount where region
  east`; the keyword `where` is required. Equality is exact string equality.
- **group-by.** Output is `<key> <value>`, one group per line, sorted by group
  key (Memelang map iteration is sorted, giving deterministic output across all
  six compiler targets). `avg` groups print two decimals.
- **join.** Inner join on equal values of the join column, which must exist in
  both tables. Output header is `a`'s full header followed by `b`'s header minus
  the join column; rows keep that order. Unmatched rows are dropped.
- **Output rendering.** Table rows are re-rendered as CSV: a field containing
  a comma, quote, or newline is quoted with `""` escapes.

## Tests

```bash
uv run --project /path/to/meme meme test tests/meme --provider heuristic --target python
# also: --target rust, cpp, go, c, ts  (all six must be green)
```

41 test cases across `test_csvparse.meme` (19), `test_csvstore.meme` (8), and
`test_csvquery.meme` (14): quoted-field parsing (commas, escaped quotes,
embedded newlines), mid-field/post-close quote tolerance, unterminated quotes,
trailing-comma rows, lone-`\r` lines, control-character detection, CRLF
handling, blank lines, rectangularity, registry round-trip incl.
quotes/backslashes/newlines, corrupt-registry tolerance, upsert/find,
hand-computed count/sum/min/max, 2-decimal avg rendering (including negatives
and half-away rounding), filtered aggregates, sorted group-by keys (incl. a
group whose first value is 0), join with dropped non-matches, where filter,
and non-integer rejection. Tests use unique scratch files under `tests/tmp/`
and clean them up. (Note: `meme test` blocks cannot assert process exit codes
or stderr routing, so the CLI's print-error-and-exit-1 paths are verified by
the demo/smoke battery, not by test blocks.)
