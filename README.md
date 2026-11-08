# SpecTrapBench — the `recon` spec-fidelity benchmark

> **SpecTrapBench** scores how faithfully a model implements an exact written
> specification that contains deliberate traps — not formal logical reasoning.
> The name is deliberate: the difficulty is *reading*, not *reasoning*. See
> [Why this name](#why-this-name).

This repo holds the operator kit for a head-to-head benchmark: give a model one
task file and see whether it builds a small, working application whose behaviour
matches eleven exact requirements. Models that "read carelessly produce working,
plausible, wrong code" — that gap is what the benchmark measures.

Three files, three different audiences. Keeping them straight is the whole
point of the setup.

| File | Safe to show the model? | Purpose |
|---|---|---|
| [`bench-app-task.md`](bench-app-task.md) | **Yes — and only this** | The task spec. Build `recon`, a double-entry payment reconciliation CLI in Python 3.11+ using only the standard library. |
| [`bench-app-prompt.md`](bench-app-prompt.md) | No | Operator runbook. How to launch a run, what to record, and why each trap discriminates. |
| [`bench-app-rubric.md`](bench-app-rubric.md) | No | Grading sheet: fixed expected values, per-requirement checks, and results so far. |

The runbook names every intended trap and the rubric contains the expected
answers. A model that reads either is **contaminated** and the run is void.

## What the task asks for

One self-contained build: `recon`, with modules for money, parsing, SQLite
persistence, matching, and reporting, plus unit tests and a README. The full
contract is in [`bench-app-task.md`](bench-app-task.md); the short version:

- amounts in integer minor units (no floats anywhere on the value path);
- half-up-away-from-zero rounding, with pro-rata allocations that sum exactly;
- tolerant CSV ingestion (BOM, CRLF, quoted commas, `=`-prefixed literals);
- explicit date-format flag, impossible dates rejected with file/line/value;
- FX conversion using the most recent rate on or before the invoice `issued` date;
- deterministic matching in `(date, amount, id)` order, single invoice first,
  then subsets of 2–4 invoices inside a ±3-day window, with fixed tie-breaks;
- refunds that reverse a match and never settle an invoice;
- idempotent import keyed on content, not `id` or row order;
- atomic import and atomic `--out` writes;
- aging buckets from an explicit `--as-of` (the only clock in the program);
- clean stdout/stderr split, exit codes 0/1/2, deterministic output.

The size budget is deliberate: roughly 500–700 lines excluding tests, so the
difficulty sits in the eleven traps rather than volume.

## How to run a session

Follow [`bench-app-prompt.md`](bench-app-prompt.md) exactly:

1. `cd` into an empty scratch directory — **one per model**, e.g.
   `C:\Coding\bench\qwenfp8\` and `C:\Coding\bench\ds4\`. Do not run from this
   project folder.
2. Copy **only** `bench-app-task.md` into that directory.
3. Send exactly one first message and nothing else:

   > Read bench-app-task.md and build what it specifies.

4. Afterwards, answer nothing except "continue" if the model stops early. Every
   clarifying question it asks is recorded as a data point, not resolved.
5. Record wall clock, turn count, tool-call count, failed/retried turns, whether
   thinking blocks appeared, and whether it claimed success without running it.

**Contamination check:** before scoring, inspect the model's tool calls. If it
read `bench-app-prompt.md` or `bench-app-rubric.md`, discard the run. A model
that announces it is "setting up benchmark infrastructure" has read the wrong
file — stop and restart clean.

## How scoring works

Settled method (2026-08-11). Total is out of **28**.

| Part | What | Points |
|---|---|---|
| A | Gate: does it run at all | Pass/fail — **never points** |
| B | 11 requirements × 2 points each | 22 |
| C | 6 quality checks × 1 point each | 6 |

**Part A (gate):** `python -m unittest discover` runs without import/syntax
errors, `recon import` completes on the fixtures, `reconcile` produces output,
and there are zero third-party imports. A model that fails the gate still gets
read-and-scored on Part B, but the failure is reported beside the score. Part A
is checked **last**, after Part B, so a plausible-looking program that doesn't
actually work cannot hide behind its traps score.

**Part B** is scored per requirement, not per check. The rubric's `2b`/`4b`/`6b`
rows are evidence *inside* their parent requirement, weighted 1 point each so
the parent still totals 2. Score 2 = correct, 1 = defensible but wrong in an
edge case, 0 = wrong or absent.

**Part C:** "Design decisions" section lists real ambiguities, line count within
500–700 excluding tests, module boundaries respected, errors name
file + line + value, two identical runs are byte-identical (including `--json`),
and no dead code or unused flags beyond `--fail-after`.

Do **not** accept the model's own passing test suite as the score. Score against
the rubric's fixed expected values. Two checks that catch otherwise-invisible
defects:

- **Cross a process boundary.** Run `import`, `reconcile`, then `report` as three
  separate invocations and check they agree. Scoring matching from one in-process
  session is the exact blind spot that let one run's own 47/47 suite go green
  while `reconcile` persisted nothing.
- **Check that correct code is reachable.** Grep each requirement's helper for
  call sites. Existence is not use — one run implemented the correct
  most-recent-rate rule and never called it.

Also check what the model's tests *feed* the parser. One run scored 11
requirements while being unable to read the specified CSV shape, because its
tests used a format it invented.

## The eleven traps and their expected values

Each trap has exactly one defensible answer. Full fixtures and per-row checks
live in [`bench-app-rubric.md`](bench-app-rubric.md).

| # | Trap | Expected check | Common failure |
|---|---|---|---|
| 1 | integer minor units | 1000 rows of `0.01` → exactly `10.00` | `float`; `0.01 * 1000 != 10.00` |
| 2 | half-up + residual placement | `100.00` over 3 equal invoices → `33.34, 33.33, 33.33`; `-2.345` → `-2.35` | banker's `round()`; allocations that don't sum |
| 3 | CSV edges | BOM + CRLF + `="+1,234"` parses, field stays literal | BOM in header name; `=` coerced |
| 4 | flag over inference | `03/04/2026` under `iso` is a hard error; `2026-02-29` errors naming file+line+value | date guessing; silent coercion to Mar 1 |
| 5 | rate as of *issued* | invoice issued `03-10`, rates on `03-01`/`03-15` → uses `03-01` | exact-date lookup; transaction date |
| 6 | ordering + subset-sum + tie-breaks | reversed file order gives identical result; a 2-invoice solution beats a 3-invoice one | file-order greedy; single-invoice only |
| 7 | refund never settles | refund of a settled transaction returns invoices to open | refund matched as a payment |
| 8 | content-keyed dedup | import twice → `0 inserted`; two identical rows in one file stay two | dedup on `id`; re-insert on second import |
| 9 | atomic import / atomic write | `--fail-after 3` leaves the row count unchanged | per-row commits; in-place report |
| 10 | `--as-of` is the only clock | `due = as-of - 30` → `0-30`; `-31` → `31-60` | `date.today()` scattered through code |
| 11 | exit codes + stream split | unmatched present → exit `1`, report on stdout only | everything on stdout; exit 0 always |

Requirements 6 and 7 interact on purpose: a refund returns invoices to open, and
whether later transactions can claim them depends on processing order. Handling
both in isolation but not together is the most interesting failure this task
produces.

## Results so far

All scores out of 28. Runs restated on 2026-08-11; rankings did not change.

| Run | Model | Part B | Part C | Total | Gate |
|---|---|---|---|---|---|
| 3 | DS4 | 20 | 4 | 24 / 28 | passed |
| 8 | Qwen3.8-27B (NVFP4+MTP, medium, 65536 cap) | 20 | 4 | 24 / 28 | passed (`unittest discover` needs `-s tests`) |
| 5 | DS4 q2 + DSpark | 19 | 4 | 23 / 28 | passed |
| 6 | DS4 q2 + DSpark | 20 | 3 | 23 / 28 | 3 of 4 — invented a no-header CSV format |
| 7 | Qwen3.8-27B (NVFP4+MTP, xhigh) | 19 | 3 | 22 / 28 | FAILED — assumes integer ids |
| 1 | qwen FP8 | 14 | 4 | 18 / 28 | failed — `reconcile` persists nothing |
| 4 | qwen FP8 | 13 | 3 | 16 / 28 | 3 of 4 — `reconcile` prints nothing |

### Reading the results

- Runs 3, 5, and 6 are the **same model on the same weights** scoring 24, 23,
  23. Treat anything under ~2 points as no difference; that is the run-to-run
  noise floor. Medium effort is still untested (run 6's setting was never
  confirmed — thinking was 54% of output against run 5's 56%).
- Where points move is more informative than totals. Run 6 fixed trap 3 (raw
  description) and trap 8 (duplicate ids) to reach Part B 20/22, then lost the
  aging edge case, the Design-decisions section, and the gate.
- The one defect all three DS4 runs share is requirement 7. It surfaced in three
  different places (`store.py:210`, `parsing.py:97`, `matching.py:101`): the
  model normalizes before a comparison the spec says must be on identical text.
- Runs 3 and 4 are the controlled pair (same client, transport, and ceiling;
  only the model differs). DS4 wins by 9 points. Run 1's `hosted_vllm/` plumbing
  cost qwen ~2 points, not 11 — the gap is the model, not the transport.
- Do not compare raw wall clock across models. DS4 runs `anthropic/`
  pass-through and keeps thinking blocks; qwen runs `hosted_vllm/` and loses
  them. At 3.6 vs 46 tok/s, the meaningful number is **points per turn**, not
  points per minute. Turns were not captured for runs 3 and 4, so that number
  does not exist for them.

## Repo layout and external dependencies

This folder contains the docs only; fixtures and harness live outside so both
models see byte-identical input:

- `C:\Coding\bench\fixtures\make-fixtures.py` — generates all fixtures
  deterministically (explicit line endings and BOM). Regenerate, don't hand-edit.
- `C:\Coding\bench\score-qwen\score.py` plus `probe2/3/4.py` — scoring harness.
  Repoint only `CODE`; `PEEK` must stay on the harness directory.

Fixture set: `penny.csv`, `alloc.csv`, `dirty.csv`, `baddate.csv`, `rates.csv`,
`subset.csv`, `refund.csv`, `dupes.csv`, `aging.csv`.

**Harness cost is real:** probes have had to be rewritten for every run so far,
because table and column names differ each time. Only `score.py` and the
fixtures genuinely port. Expect a fresh `peekN.py` per run. Also check
`main()`'s signature before scoring — one run's `main()` took no `argv` and read
`sys.argv` directly, so a runner passing a list looked like a broken app.

**Before a run:** `ds4-plan` was down as of 2026-08-08 — check it first or you
will benchmark only one model:

```powershell
curl -s -m 6 http://192.168.1.183:18004/v1/models
```

## Why this name

The kit was originally filed as "Logic Benchmark". That name described the
wrong thing: this test scores **specification fidelity under deliberate traps**
— careful reading, exact arithmetic, correct edge cases — not formal logical
reasoning. It also collided with [LogicBench](https://arxiv.org/abs/2404.15522),
an existing ACL 2024 logical-reasoning benchmark.

**SpecTrapBench** (folder `spec-trap-bench`) names both halves of what is
measured: the exact specification and the eleven planted traps. The runbook
calls them traps throughout, and the score moves precisely on whether a model
read the spec or inferred around it.

Alternatives that were considered and rejected: *ReconBench* (names the
artifact but not the method), *ContractBench* (accurate but generic), and
*FidelityBench* (the metric, not the test).
