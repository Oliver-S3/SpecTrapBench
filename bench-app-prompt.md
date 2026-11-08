# Operator runbook — running a local model against SpecTrapBench

**Operator file. Never show this to a model under test, and never show
`bench-app-rubric.md`.** The task itself is `bench-app-task.md`, which is the
only file safe to hand over.

This runbook is about local / self-hosted models: how to stand one up behind an
agentic client, how to launch the run so it is comparable, what to record, and
why each trap discriminates. It assumes you already have weights you can serve
(vLLM, llama.cpp, Ollama, LM Studio, or a LiteLLM proxy in front of several).

## Files

| File | Show to model? | What it is |
|---|---|---|
| `bench-app-task.md` | **yes** | the task, and nothing else |
| `bench-app-prompt.md` | no | this runbook — names every intended trap |
| `bench-app-rubric.md` | no | grading sheet with expected values |
| `bench-app-grading.md` | no | isolation setup — folder layout, operator and second-model grading |

## Rules of the run

- Create an **empty scratch directory per model** before launching, e.g.
  `bench/model-a/`, `bench/model-b-q4/`. Never run from the kit repo folder.
- Copy **only** `bench-app-task.md` into that directory.
- Keep the rubric, runbook, fixtures and harness in a separate `grading/`
  folder and open them only **after** the run finishes. See
  `bench-app-grading.md` for the folder layout and the second-model grading
  setup.
- Give **exactly** this as the first message and nothing more:

  > Read bench-app-task.md and build what it specifies.

- Answer nothing afterwards except **"continue"** if it stops early. Every
  clarifying question it asks is a data point, not something to resolve.
- Record: wall clock, turn count, tool-call count, any turn that failed or
  retried, whether thinking blocks appeared, and whether it claimed something
  worked without running it.
- Do **not** run the model's own tests as the score. Score against the rubric's
  fixed expected values — a model that writes its own passing tests around a
  wrong reading of the spec must not get credit for it.

## Contamination

**Only `bench-app-task.md` may reach the model.** Every other repo file — this
runbook, the rubric, the README, the grading guide — names intended traps or
expected values, so if a model reads any of them the **run is void**. Check its
tool calls for reads outside the run directory before scoring. Keeping each run
in a scratch directory that contains *only* `bench-app-task.md` is the cheapest
prevention.

A model that announces it is "setting up benchmark infrastructure" has read the
wrong file — it saw operator material, not the task. Stop and restart clean.

Also watch for contamination through the agentic client itself: a client that
loads repo-wide context files, workspace instructions, or auto-reads the parent
directory can leak the rubric without the model visibly opening it.

## Serving the model

The client matters as much as the weights. Use an agentic client that can write
files and run shell commands (Cline, Roo Code, OpenHands, Aider, Continue),
pointed at an OpenAI-compatible endpoint.

| Backend | Endpoint | Notes |
|---|---|---|
| **vLLM** | `http://localhost:8000/v1` | add `--enable-auto-tool-choice --tool-call-parser hermes` (or the parser your model's template expects) for tool use |
| **llama.cpp** | `http://localhost:8080/v1` | `llama-server -m model.gguf --port 8080 --jinja`; the chat template must emit tool calls |
| **Ollama** | `http://localhost:11434/v1` | `ollama serve`; use a tool-capable tag |
| **LM Studio** | `http://localhost:1234/v1` | enable the local server; tool-call support depends on the model |
| **LiteLLM proxy** | `http://localhost:4000/v1` | one config, several models — useful for A/B runs on the same harness |

Record these settings per run, because they change the score:

- **Context / max tokens.** 16k is workable, 32k–64k is comfortable. A truncated
  run is not a fair run; log the cap.
- **Temperature.** 0 or near it. This is a fidelity test, not a creativity one.
- **Reasoning effort**, where supported (`low` / `medium` / `high`).
- **Transports that keep or drop thinking blocks.** Anthropic-style
  pass-throughs often keep them; plain OpenAI-compatible ones drop them — which
  makes wall-clock numbers incomparable across transports.
- **Quantization.** Say it in the result (`Q4_K_M`, FP8, …).

Before launching, check the server is alive:
`curl http://localhost:8000/v1/models`.

## Size budget (why the task is shaped this way)

Deliberately capped at ~700 lines of output so a small local model can finish it
in a reasonable context, rather than timing out or truncating. The difficulty is
concentrated in **11 specification traps** that each have exactly one defensible
answer — a model that reads carelessly produces working, plausible, wrong code.
That discriminates far better than volume, and it is much cheaper to run than a
multi-module application.

## Why each trap discriminates a local model

| # | Trap | What it catches |
|---|---|---|
| 1 | integer minor units | `float` amounts; `0.01 * 1000 != 10.00` |
| 2 | half-up + residual placement | banker's rounding via `round()`; allocations that don't sum |
| 3 | CSV edges | BOM left in the first header name; `=` field coerced or stripped by a "cleanup" reflex |
| 4 | format flag over inference | `dateutil`-style guessing; `2026-02-29` silently becoming Mar 1 |
| 5 | rate as of *issued*, most-recent-on-or-before | exact-date lookup only; uses the transaction date |
| 6 | ordering + subset-sum + tie-breaks | file-order greedy; single-invoice matching only |
| 7 | refund reverses, never settles | refund matched as a payment against an invoice; normalizing text that must match identically |
| 8 | dedup key, and duplicates *within* a file collapsing | dedup on `id`; second import re-inserts |
| 9 | one transaction, atomic write | per-row commits; report written in place |
| 10 | `--as-of` is the only clock | `date.today()` sprinkled through, tests unreproducible |
| 11 | exit codes + stream split | everything on stdout; exit 0 always |

Requirements 6 and 7 interact deliberately: a refund reversal returns invoices
to open, and whether later transactions can then claim them depends on the
processing order in 6. Handling both in isolation but not together is the most
interesting failure this task produces.

## The fixture set

This kit ships no fixtures or scoring harness — you provide both. Build the nine
cases below once, deterministically (explicit line endings, explicit BOM), from
the descriptions in the rubric; then keep them **outside** every scratch
directory and copy them in, so each run sees byte-identical input. Grading is
done by you (or a grader model) against `bench-app-rubric.md`, not by a script
in this repo.

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
`rates.csv` where FX is involved. Copy one case at a time into the scratch
directory and keep the file names stable across models.

## Scoring sequence

Score Part B **first**, then Part A (the gate), then Part C, then record Part D.

1. Copy one fixture case into the scratch directory.
2. Run `import`, then `reconcile`, then `report` as **three separate
   processes** and check they agree. Never score matching from a single
   in-process session.
3. Check the expected value for the requirement under test, not the model's own
   assertion. Grep each requirement's helper for call sites — existence is not
   use.
4. Inspect what the model's tests feed the parser. A model can pass requirements
   while being unable to read the specified file shape, because its tests feed
   it a format it invented.
5. Run the gate checks last: `python -m unittest discover`, `import` on the
   fixtures, `reconcile` produces output, zero third-party imports.

### Grading with a second model

If you don't want to grade by hand, use a grader model — preferably a different
model than the one under test. Copy the finished tree into
`grading/<model>/produced/`, copy the fixtures in alongside it, and give the
grader agent `bench-app-rubric.md` plus this instruction:

> Score the code in `./produced` against `bench-app-rubric.md`. Use the fixtures
> in `./fixtures`. Score Part B per requirement against the fixed expected
> values, run `import` / `reconcile` / `report` as separate processes, check that
> each requirement's implementation is actually called, then run the Part A gate
> and report it as pass/fail beside the total. Output the completed tables with
> evidence.

The grader lives in the grading folder; the tested model never sees it. Full
setup and the contamination checklist are in `bench-app-grading.md`.

## Harness notes

**Budget for a fresh probe per run.** Table and column names differ every run
(one run had `amount_minor` plus a `match_summary` view, another had
`matches.amount` / `rates.rate`, and one with no `matches` table at all —
that last one is by design, not a defect). Only the scoring driver and the
fixtures genuinely port.

- Repoint **only** the code path at the run being scored. Keep the probe's
  helper imports in the harness directory, not the run directory.
- Check `main()`'s signature before scoring. One run's `main()` took no `argv`
  and read `sys.argv` directly; a runner passing a list then looked like a
  broken app.
- Prefer black-box probes (subprocess calls to the CLI) over importing the
  model's internal API — internal probes have needed rewriting every time and
  can hide the very defects the fixtures are for.

## Publishing a run

Report the model id and weights, backend + client, context cap, temperature,
reasoning effort, process metrics, and Part A/B/C scores with the failing checks
quoted. Publish the gate result next to the score — a 24/28 with a failed gate
is a different animal from a 24/28 that runs.
