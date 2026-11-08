# Task: build `recon`

Build a command-line application called `recon` — a double-entry payment
reconciliation engine. Python 3.11+, **standard library only**, no third-party
packages.

Your deliverable is the `recon` application itself, described below. Create it in
the current directory. Do not build test harnesses, benchmarks, or tooling for
grading this task; build only what this file specifies.

## Deliverables

```
recon/__init__.py
recon/cli.py          argument parsing, exit codes, stdout/stderr routing
recon/money.py        money type + rounding + allocation
recon/parsing.py      CSV + date ingestion
recon/store.py        SQLite persistence, idempotent import
recon/matching.py     the matching engine
recon/report.py       text + JSON reporting
tests/                unittest, runnable via `python -m unittest discover`
README.md             usage, and a "Design decisions" section
```

## Data model

Two CSV inputs.

`transactions.csv` — a bank feed: `id,date,amount,currency,description`
`invoices.csv` — the ledger: `id,issued,due,amount,currency,customer,memo`

A *transaction* is money that moved. An *invoice* is money owed. Reconciliation
decides which transactions settle which invoices.

## Commands

- `recon import --transactions FILE --invoices FILE [--date-format {iso,dmy}] [--rates FILE]`
- `recon reconcile [--as-of YYYY-MM-DD] [--json]`
- `recon report [--as-of YYYY-MM-DD] [--json] [--out FILE]`

State persists in `./recon.db` (SQLite). `--rates` is a CSV of
`date,currency,rate_to_usd`.

## Requirements — these are exact, not suggestions

**1. Money.** All amounts are stored and computed as integer minor units. Binary
floating point must not appear anywhere in a value that reaches output. Summing
one thousand rows of `0.01` must produce exactly `10.00`.

**2. Rounding and allocation.** Rounding is half-up away from zero
(`2.345 -> 2.35`, `-2.345 -> -2.35`). When one payment is allocated across N
invoices pro rata, allocations must sum to the payment exactly; distribute
leftover minor units one each to the largest allocations, breaking ties by
lowest invoice `id` (lexicographic). Allocating `100.00` across three equal
invoices yields `33.34, 33.33, 33.33`.

**3. CSV robustness.** Accept a UTF-8 BOM, CRLF or LF line endings, a trailing
blank line, quoted fields containing commas and escaped double-quotes, and
fields beginning with `=` or `+` (treated as literal text, never as a formula
or a number). Leading/trailing whitespace around values is insignificant;
whitespace inside a quoted field is significant.

**4. Dates.** `--date-format iso` means `YYYY-MM-DD`; `dmy` means `DD/MM/YYYY`.
Default `iso`. Never infer the format from the data — `03/04/2026` must obey the
flag. An impossible date such as `2026-02-29` is a hard input error naming the
file, line number and offending value; it must not be coerced to a nearby valid
date. All dates are naive calendar dates; no timezone conversion anywhere.

**5. Currency.** Amounts may be in a currency other than USD. Convert using the
rate effective **on the invoice's `issued` date** — meaning the most recent rate
on or before that date — not today's rate and not the transaction's date. A
missing rate is a hard error naming the currency and the date that needed a
rate. USD needs no rate.

**6. Matching.** Process transactions in ascending order of
`(date, amount_minor, id)` — not file order. For each unsettled transaction, in
this precedence:

  a. a single open invoice whose converted amount equals the transaction amount
     and whose `issued` date is within ±3 calendar days inclusive of the
     transaction date;
  b. failing that, a **subset of 2 to 4** open invoices, all within the same
     ±3-day window, whose converted amounts sum exactly to the transaction
     amount;
  c. failing that, the transaction is unmatched.

Prefer the smallest subset. Among candidates of equal size, prefer the one whose
invoice dates are closest to the transaction date (smallest total absolute day
difference); break remaining ties by the sorted invoice-`id` sequence,
lexicographically smallest. An invoice may be settled only once.

**7. Refunds.** A negative transaction amount is a refund. A refund never
settles an invoice; it reverses a previously matched transaction of the same
absolute amount, same currency, and identical `description`, and returns that
transaction's invoices to open. If no such transaction exists, the refund is
unmatched.

**8. Idempotent import.** Importing the same file twice inserts nothing the
second time and reports `0 inserted`. Identity is
`(date, amount_minor, currency, normalized_description)` where normalization is
casefold plus internal-whitespace collapse — **not** the `id` column and not row
order. Two rows identical under that key within a single file are two distinct
transactions, not one.

**9. Atomicity.** An import is all-or-nothing: one SQLite transaction, and a
failure part-way leaves the database exactly as it was. `--out` writes
atomically via temp file plus rename, never a partially-written report. Provide
a hidden `--fail-after N` flag that raises after N rows, so this is testable.

**10. Aging.** `report` buckets unmatched invoices by age from `due` to
`--as-of`: `0-30`, `31-60`, `61-90`, `90+`, in whole days, boundaries inclusive
on the lower end — exactly 30 days is `0-30`, exactly 31 is `31-60`. `--as-of`
defaults to today, and no other part of the program may read the clock.

**11. Contract.** Report to stdout, diagnostics to stderr, never mixed. Exit `0`
if every transaction matched, `1` if any transaction or invoice is unmatched,
`2` on an input or configuration error. Output ordering is fully deterministic
for identical input. `--json` emits one JSON object with the same information as
the text report.

## Testing

Include unit tests covering each numbered requirement above, plus fixture CSVs
under `tests/fixtures/`. Tests must pass via `python -m unittest discover` with
no network access and no environment setup.

## Constraints

Aim for roughly 500-700 lines of Python excluding tests. Prefer clear code over
clever code. When a requirement above is ambiguous, choose an interpretation,
implement it, and record the choice under "Design decisions" in `README.md`
rather than asking.
