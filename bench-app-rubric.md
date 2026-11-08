# Grading sheet — `bench-app-prompt.md`

The model never sees this file. Score each row independently; don't let a model's
own passing test suite substitute for a check here.

## Part A — does it run at all (gate)

| Check | qwen FP8 | DS4 |
|---|---|---|
| `python -m unittest discover` runs without import/syntax error | | |
| `recon import ...` completes on the fixtures below | | |
| `recon reconcile` produces output | | |
| Zero third-party imports (`grep -rE '^\s*(import\|from) ' | reject non-stdlib`) | | |

A model failing the gate still gets scored on Part B by reading the code.

## Part B — the 11 traps, 2 points each

Score **2** = correct, **1** = attempted and defensible but wrong in an edge
case, **0** = wrong or absent. Max 22.

| # | Exact check | Expected | qwen | DS4 |
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

| Check | qwen | DS4 |
|---|---|---|
| "Design decisions" section actually lists real ambiguities it resolved | | |
| Line count within 500-700 excluding tests | | |
| Module boundaries as specified (matching logic not in `cli.py`) | | |
| Errors name file + line + value, not just a traceback | | |
| Determinism: two identical runs byte-identical (incl. `--json`) | | |
| No dead code, no unused flags beyond `--fail-after` | | |

## Part D — process (record, don't score)

|  | qwen | DS4 |
|---|---|---|
| Wall clock | | |
| Turns / tool calls | | |
| Failed or retried turns (BUGS.md #1) | | |
| Thinking blocks present (`anthropic/` yes, `hosted_vllm/` no) | | |
| Asked clarifying questions despite being told not to | | |
| Claimed something worked without running it | | |

Note the asymmetry before comparing wall clock: DS4 runs `anthropic/`
pass-through and keeps thinking blocks; qwen runs `hosted_vllm/` and loses them.
At 3.6 vs 46 tok/s a raw time comparison says nothing about quality — the
interesting number is **points per turn**, not points per minute.

## Part A is not a formality — check it last, after Part B

Part B can be scored high on a program that doesn't work. Run 1 scored 16/22 on
traps while `reconcile` persisted nothing: it printed `Matched: 1`, and `report`
in a *new process* called the same transaction unmatched. Two checks catch this
class of defect and neither is a trap:

- **Cross a process boundary.** Run `import`, `reconcile`, then `report` as three
  separate invocations and ask whether they agree. Never score matching from a
  single in-process session — that's exactly the blind spot that made run 1's own
  47/47 suite green.
- **Check that correct code is reachable.** Run 1 implemented trap 5's
  most-recent-rate rule correctly in `Store.get_exchange_rate` and never called
  it. Grep each requirement's helper for call sites; existence isn't use.

## Fixture set — BUILT, reuse it

`C:\Coding\bench\fixtures\make-fixtures.py` generates all of these
deterministically (explicit line endings, explicit BOM). Regenerate; don't
hand-edit. Harness: `C:\Coding\bench\score-qwen\score.py` plus `probe2/3/4.py`
— repoint `CODE` at the run being scored. A CLI-only pass is not sufficient.

Keep these outside both scratch dirs and copy them in, so the two runs see
byte-identical input:

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

Final, all runs on the reconciled method (see below) — `bench-results.md` has the
per-requirement tables. Run 2 was void and never produced code.

| Run | Model | Part B | Part C | Total | Gate |
|---|---|---|---|---|---|
| 3 | DS4 | 20 | 4 | **24 / 28** | passed |
| 8 | Qwen3.8-27B (NVFP4+MTP, medium, 65536 cap) | 20 | 4 | **24 / 28** | passed (caveat: `unittest discover` needs `-s tests`) |
| 5 | DS4 q2 + DSpark | 19 | 4 | **23 / 28** | passed |
| 6 | DS4 q2 + DSpark | 20 | 3 | **23 / 28** | **3 of 4** — invented a no-header CSV format; all 23 fixtures fail at import |
| 7 | Qwen3.8-27B (NVFP4+MTP, xhigh) | 19 | 3 | **22 / 28** | **FAILED** — `int(rec["id"])` assumes integer ids; every string-id fixture fails at import |
| 1 | qwen FP8 | 14 | 4 | **18 / 28** | **failed** — `reconcile` persists nothing |
| 4 | qwen FP8 | 13 | 3 | **16 / 28** | **3 of 4** — `reconcile` prints nothing |

**Runs 3, 5 and 6 are the same model on the same weights**: 24, 23, 23. Runs 5 and 6
are also the same *configuration* — run 6 was meant to be the `reasoning_effort:
medium` pair but the setting was never confirmed, and the log shows thinking at 54%
of output against run 5's 56%, i.e. **unchanged**. So this is a three-sample read on
the run-to-run noise floor: **treat anything under ~2 points as no difference.**
Medium effort remains **untested**.

**Where the points move is more informative than the totals.** Run 6 fixed the raw-
description line that cost runs 3 and 5 (trap 3) and run 5's duplicate-id bug (trap
8), reaching Part B 20/22 — then lost the aging edge case, the "Design decisions"
section, and the gate. The *only* defect all three DS4 runs share is requirement 7,
and it has surfaced in three different places (`store.py:210`, `parsing.py:97`,
`matching.py:101`): the model keeps normalizing before a comparison the spec says
must be on *identical* text.

**Add a gate row for the input contract.** Run 6 passed 11 requirements while being
unable to read the specified file shape, because its own tests fed it the format it
invented. Check that `import` succeeds on a fixture *before* scoring anything, and
check what the model's tests actually feed the parser.

**Runs 3 and 4 are the controlled pair**: both via Cline direct to the Spark at a
16384 ceiling, only the model differs. DS4 wins by 9 points. Run 1's `hosted_vllm/`
plumbing cost qwen ~2 points, not 11 — the gap is the model, not the transport.

Run 3 caveat: client + transport + ceiling all differ from run 1, so trap scores
are comparable there (same fixtures, same harness) but process metrics are not.
Neither Cline run captured a turn count, so **points-per-turn — called the most
important number above — does not exist for runs 3 or 4.**

**Scoring method — settled 2026-08-11 with run 5. Use this one.**

- **Part B is 11 requirements × 2 = 22, scored per requirement.** The `2b`/`4b`/`6b`
  rows above are evidence *inside* their parent requirement, not extra cells —
  otherwise requirement 6 is worth 4 points when split and 2 when not.
- **Part C is 6 rows × 1 = 6.** Total out of 28.
- **Part A is a gate: pass/fail, never points.** Report it beside the score. Run 1's
  published 13/28 could not be reproduced from its own tables because a ~7-point
  gate penalty was folded in silently; that number is now 18/28 with **gate FAILED**
  written next to it, which says the same thing and can be audited.

All four runs were restated on 2026-08-11; rankings did not change.

Harness note: **the probes have had to be rewritten for every run so far — budget
for it.** Run 1's `probe2/3/4.py` call qwen's *internal* API (`find_matches`,
`allocate_payment`, `CSVParser.parse_transaction_row`); `ds4-probe1..6.py` are
black-box but assume DS4's module layout; run 4 needed `qc-probe1..3.py`. Only
`score.py` and the fixtures genuinely port. Per-run checklist:

- Repoint **only** `CODE`. `PEEK` must keep pointing at the harness dir.
- Expect a fresh `peekN.py`: table and column names differ every run
  (run 1 `amount_minor` + a `match_summary` view, run 4 `matches.amount` /
  `rates.rate`, DS4 no `matches` table at all — that last one is by design, not a
  defect).
- Check `main()`'s signature. Run 4's takes **no argv** and reads `sys.argv`
  directly; a runner that passes a list fails in a way that looks like the app is
  broken. `score4.py`'s `RUNNER` inspects the signature and handles both.
