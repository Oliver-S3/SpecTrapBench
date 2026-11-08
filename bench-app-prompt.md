# Head-to-head runbook — qwen FP8 vs DeepSeek V4

**Operator file. Never show this to a model under test, and never show
`bench-app-rubric.md`.** The task itself is `bench-app-task.md`, which is the
only file safe to hand over.

## Files

| File | Show to model? | What it is |
|---|---|---|
| `bench-app-task.md` | **yes** | the task, and nothing else |
| `bench-app-prompt.md` | no | this runbook — names every intended trap |
| `bench-app-rubric.md` | no | grading sheet with expected values |

## Rules of the run

- `cd` to an empty scratch dir before launching; one per model
  (`C:\Coding\bench\qwenfp8\`, `C:\Coding\bench\ds4\`).
- Copy `bench-app-task.md` into that dir, then give **exactly** this as the first
  message and nothing more:

  > Read bench-app-task.md and build what it specifies.

- Answer nothing afterwards except "continue" if it stops early. Every
  clarifying question it asks is a data point, not something to resolve.
- Record: wall clock, turn count, tool-call count, any turn that failed or
  retried (BUGS.md #1 lives here), and whether thinking blocks appeared.
- Do **not** run the model's own tests as the score. Score against the rubric's
  fixed expected values — a model that writes its own passing tests around a
  wrong reading of the spec must not get credit for it.

### Contamination

If a model reads this file or the rubric, **the run is void** — both name the
intended failure modes and the expected values. Check its tool calls for reads
of `bench-app-prompt.md` / `bench-app-rubric.md` before scoring. Keeping the two
graded runs in scratch dirs that contain *only* `bench-app-task.md` is the
cheapest way to prevent it; don't run the benchmark from this project folder.

A model that announces it is "setting up benchmark infrastructure" has read the
wrong file — it saw the runbook, not the task. Stop and restart clean.

## Size budget (why the task is shaped this way)

Deliberately capped at ~700 lines of output so DS4 finishes in roughly an hour
instead of five. The difficulty is concentrated in **11 specification traps**
that each have exactly one defensible answer — a model that reads carelessly
produces working, plausible, wrong code. That discriminates far better than
volume does.

## Why each trap discriminates

| # | Trap | Common failure |
|---|---|---|
| 1 | integer minor units | `float` amounts; `0.01 * 1000 != 10.00` |
| 2 | half-up + residual placement | banker's rounding via `round()`; allocations that don't sum |
| 3 | CSV edges | BOM in first header name; `=` field coerced or stripped |
| 4 | format flag over inference | `dateutil`-style guessing; `2026-02-29` silently becoming Mar 1 |
| 5 | rate as of *issued*, most-recent-on-or-before | exact-date lookup only; uses transaction date |
| 6 | ordering + subset-sum + tie-breaks | file-order greedy; single-invoice matching only |
| 7 | refund reverses, never settles | refund matched as a payment against an invoice |
| 8 | dedup key, and duplicates *within* a file collapsing | dedup on `id`; second import re-inserts |
| 9 | one transaction, atomic write | per-row commits; report written in place |
| 10 | `--as-of` is the only clock | `date.today()` sprinkled through, tests unreproducible |
| 11 | exit codes + stream split | everything on stdout; exit 0 always |

Requirements 6 and 7 interact deliberately: a refund reversal returns invoices
to open, and whether later transactions can then claim them depends on the
processing order in 6. Handling both in isolation but not together is the most
interesting failure this task produces.

## Before you run it

`ds4-plan` is down as of 2026-08-08 — check first, or you'll bench one model:

```powershell
curl -s -m 6 http://192.168.1.183:18004/v1/models
```
