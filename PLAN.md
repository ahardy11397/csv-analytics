# PLAN — csv-analytics: a Memelang graduate-level demo

Build a CSV query/analytics CLI entirely in Memelang. Beta demonstration
program: the data-engineering assignment — parsing, joins, group-by,
aggregation.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang. All numeric values integer (Int) for aggregation; floats
  only at print time via `format <x> with 2 decimals` for averages.
- CSV dialect: RFC-4180-lite — comma-separated, optional double-quoted
  fields (escaped `""`), \r\n or \n line endings, no header quoting
  requirements. Parse it YOURSELF in-language (string verbs), do not
  assume simple split.
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Domain and CLI (thin argv-router main + logic modules; unknown command →
usage + exit 1; every error = one stderr-style line + exit 1)

    csv load <file> <name>               # register a table for this session;
                                         # persisted into data/tables.json
    csv tables                           # registered tables: name, rows, cols
    csv head <name> [n]                  # first n rows (default 5)
    csv count <name> [column value]      # row count, or rows where column=value
    csv sum|avg|min|max <name> <column> [where column value ...]
                                         # aggregation; avg printed with 2 decimals
    csv groupby <name> <group-col> <agg>(<col>)   # e.g. groupby sales region sum(amount)
    csv join <a> <b> <column>            # inner join on equal column values,
                                         # prints combined rows
    csv where <name> <column> <value>    # filtered rows to stdout (CSV out)

Rules: ragged rows are rejected at load (one line, exit 1); quoted fields
may contain commas/newlines; aggregations on non-integer cell values fail
with a clean error; group-by output sorted by group key; join column must
exist in both tables; `where` chains are single-condition only (keep it
simple).

Suggested model: `Table` record (name, header List<String>,
rows List<List<String>>); cell→Int conversion via the safe parse (`integer
value of ... or else nothing`); group maps as `Map<String, Int>`.

## Tests — tests/meme/*.meme, `meme test`

At minimum: quoted-field parse (comma + escaped quotes + embedded newline
in a quoted field); CRLF handling; ragged-row rejection; count/sum/min/max
on fixed data with hand-computed answers; avg with 2-decimal rendering;
group-by with sorted keys; join on a shared column incl. rows with no
match dropped; where filter; non-numeric aggregation rejection; corrupt
data tolerance. Unique scratch files per test, cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o csv_bin` — end-to-end demo: load a small sales
   CSV + a regions CSV, join, group-by sum, where-filter, aggregates.
   Transcript in your final summary.
4. README.md: CSV dialect, CLI usage, semantics, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.
