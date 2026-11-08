# Grading sheet — SpecTrapBench

The model never sees this file — or any repo file other than
`bench-app-task.md` (the README, runbook, and grading guide also give away traps
and expected values). Score each row independently; don't let a model's own
passing test suite substitute for a check here.

Total is out of **28**: Part A is a gate (pass/fail, never points), Part B is
11 requirements × 2 = 22, Part C is 6 quality checks × 1 = 6, Part D is recorded
but not scored.

## Part A — does it run at all (gate)

Check this **last**, after Part B.

| Check | Run A | Run B |
|---|---|---|
| `python -m unittest discover` runs without import/syntax error | | |
| `recon import ...` completes on the fixtures below | | |
| `recon reconcile` produces output | | |
| Zero third-party imports (`grep -rE '^\s*(import\|from) '` and reject non-stdlib) | | |

A model failing the gate still gets scored on Part B by reading the code — but
the gate failure is reported beside the score, never folded into it.

## Part B — the 11 traps, 2 points each

Score **2** = correct, **1** = attempted and defensible but wrong in an edge
case, **0** = wrong or absent. Max 22.

| # | Exact check | Expected | Run A | Run B |
|---|---|---|---|---|
| 1 | 1000 rows of `0.01` | exactly `10.00`; no `float` in any value path | | |
| 2 | `100.00` over 3 equal invoices | `33.34, 33.33, 33.33`, sum `100.00` | | |
| 2b | `-2.345` | `-2.35` (not `-2.34`, not banker's) | | |
| 3 | BOM + CRLF + `="+1,234"` field | parses; field stays literal text | | |
| 4 | `03/04/2026` under `--date-format iso` | hard error, not 3 Apr | | |
| 4b | `2026-02-29` | error naming file + line + value | | |
| 5 | invoice issued `2026-03-10`, rates only on `03-01` and `03-15` | uses `03-01` rate | | |
| 6 | file order reversed vs `(date, amount, id)` order | identical result both ways | | |
| 6b | subset-sum case with a 2- and a 3-invoice solution | picks the 2 | | |
| 7 | refund matching a settled transaction | invoices return to open; refund settles nothing | | |
| 8 | same file imported twice | `0 inserted`; two identical rows in one file stay 2 | | |
| 9 | `--fail-after 3` mid-import | row count unchanged from before | | |
| 10 | `due = as-of - 30` and `-31` | `0-30` and `31-60` | | |
| 11 | unmatched present | exit `1`; report on stdout only | | |

(Rows 2b/4b/6b are the same requirement split where one number can't cover both
halves; weight each at 1 point so the requirement still totals 2.)

## Part C — quality, 1 point each (max 6)

| Check | Run A | Run B |
|---|---|---|
| "Design decisions" section actually lists real ambiguities it resolved | | |
| Line count within 500–700 excluding tests | | |
| Module boundaries as specified (matching logic not in `cli.py`) | | |
| Errors name file + line + value, not just a traceback | | |
| Determinism: two identical runs byte-identical (incl. `--json`) | | |
| No dead code, no unused flags beyond `--fail-after` | | |

## Part D — process (record, don't score)

| | Run A | Run B |
|---|---|---|
| Wall clock | | |
| Turns / tool calls | | |
| Failed or retried turns | | |
| Thinking blocks present | | |
| Asked clarifying questions despite being told not to | | |
| Claimed something worked without running it | | |

Note the asymmetry before comparing wall clock: transports differ in whether
thinking blocks are preserved, and decode speed varies by orders of magnitude
across local hardware and quantization. A raw time comparison says nothing about
quality — the interesting number is **points per turn**, not points per minute.
Record turns for every run so that number exists.

## Part A is not a formality — check it last, after Part B

Part B can be scored high on a program that doesn't work. One run scored 16/22 on
traps while `reconcile` persisted nothing: it printed `Matched: 1`, and `report`
in a *new process* called the same transaction unmatched. Two checks catch this
class of defect and neither is a trap:

- **Cross a process boundary.** Run `import`, `reconcile`, then `report` as three
  separate invocations and ask whether they agree. Never score matching from a
  single in-process session — that is exactly the blind spot that made one run's
  own 47/47 suite green.
- **Check that correct code is reachable.** One run implemented trap 5's
  most-recent-rate rule correctly in `Store.get_exchange_rate` and never called
  it. Grep each requirement's helper for call sites; existence isn't use.

**Add a gate row for the input contract.** One run passed 11 requirements while
being unable to read the specified file shape, because its own tests fed it the
format it invented. Check that `import` succeeds on a fixture *before* scoring
anything, and check what the model's tests actually feed the parser.

## Fixture set

This kit ships no fixture files — build them from these descriptions, once,
deterministically (explicit line endings, explicit BOM). Keep them outside every
scratch directory and copy them in so each run sees byte-identical input; don't
hand-edit them. Scoring is done by you (or a grader model) against this sheet;
there is no scoring script in the repo.

```
penny.csv        1000 transactions of 0.01, same date
alloc.csv        1 transaction 100.00 + 3 invoices of 33.34/33.33/33.33 target
dirty.csv        BOM, CRLF, trailing blank line, "a,b" quoted, ="+1,234", ' padded '
baddate.csv      one 2026-02-29 row, one 03/04/2026 row
rates.csv        USD implicit; EUR on 2026-03-01 and 2026-03-15 only
subset.csv       transaction matching both a 2- and a 3-invoice sum
refund.csv       a payment, its invoices, then an exact-description refund
dupes.csv        two rows identical under the dedup key, differing ids
aging.csv        invoices due at as-of -30, -31, -60, -61, -90, -91
```

Each case is an import pair — `transactions.csv` and `invoices.csv` — plus
`rates.csv` where FX is involved. Keep the file names stable across models.

## Scoring method — settled 2026-08-11

- **Part B is 11 requirements × 2 = 22, scored per requirement.** The `2b`/`4b`/`6b`
  rows are evidence *inside* their parent requirement, not extra cells —
  otherwise requirement 6 is worth 4 points when split and 2 when not.
- **Part C is 6 rows × 1 = 6.** Total out of 28.
- **Part A is a gate: pass/fail, never points.** Report it beside the score. A
  gate penalty folded silently into the total hides the most important fact:
  a high traps score on code that cannot read the specified input. Write
  **gate FAILED** next to the score and let the two numbers say different things.
- **Score the fixed expected values, not the model's tests.** A model that
  writes a green suite around a wrong reading gets no credit for the suite.

## Recording your own results

This sheet deliberately publishes no ranking. Scores are specific to the weights,
quantization, backend, context cap, and client you tested — so publish the run
you measured:

| Run | Model + weights | Backend / client | Context cap | Temperature | Part B | Part C | Total | Gate |
|---|---|---|---|---|---|---|---|---|
| 1 | | | | | /22 | /6 | /28 | passed / failed |

Add the process row from Part D beside it, and quote the failing check's
evidence. Two comparisons worth keeping in mind:

- **Noise floor.** Expect roughly 2 points of run-to-run spread on identical
  weights and settings. Anything under that is not a difference.
- **Points per turn, not points per minute.** Transports differ in whether
  thinking blocks survive, and local decode speed varies enormously across
  hardware and quantization. Record turns for every run so the number exists.

## Harness notes

**Budget for a fresh probe per run.** Table and column names differ every run
(one run `amount_minor` plus a `match_summary` view, another `matches.amount` /
`rates.rate`, another with no `matches` table at all — that last one is by
design, not a defect). Only the scoring driver and the fixtures genuinely port.

- Repoint the code path at the run being scored; keep probe helper imports in the
  harness directory, not the run directory.
- Check `main()`'s signature before scoring. One run's `main()` takes **no argv**
  and reads `sys.argv` directly; a runner that passes a list fails in a way that
  looks like the app is broken.
- Prefer black-box probes (subprocess CLI calls) over importing the model's
  internal API; internal probes have needed rewriting every run and can hide the
  defects the fixtures exist to catch.
