# SpecTrapBench

**A specification-fidelity benchmark for local LLMs.** Give a model one task
file and see whether it builds the exact thing the spec says — or a plausible,
working, wrong approximation.

Most coding benchmarks reward volume or algorithmic cleverness. SpecTrapBench
does neither. It hands the model a deliberately exact specification of a small
application (`recon`, a payment-reconciliation CLI, ~500–700 lines of Python
using only the standard library) and scores whether the *reading* was faithful.
The difficulty is concentrated in **eleven planted traps**, each with exactly one
defensible answer.

A careless reader produces code that runs, looks right, and is wrong in ways
that matter in production: floats on a money path, banker's rounding, guessed
date formats, idempotency keyed on the wrong column, a refund that settles an
invoice, an aging report that reads the clock. That gap — *working, plausible,
wrong* — is what this benchmark measures.

## Why local models

Small and self-hosted models (7B–70B, often quantized) are exactly where
specification fidelity breaks first:

- **Long specs get lossy.** Eleven requirements at 16k–64k context degrade into
  "build a CSV-to-SQLite CLI", and the details get regenerated from priors
  instead of read.
- **Tool-calling is shakier.** More failed turns, more re-reads, more
  self-reported success without actually running anything.
- **Priors win.** Local models have strong, familiar patterns for "money",
  "CSV", "dates", and "dedupe" — and every one of those priors is deliberately
  wrong here.

That makes the benchmark useful for choosing local weights, quantization levels,
or reasoning settings: the scores move on reading fidelity, not on luck.

## What a run looks like

One run = one model, one scratch directory, one task file, one prompt.

1. **Serve the model** with a client that can write files and run shell commands
   (see [Serving a local model](#serving-a-local-model)). Agentic clients —
   Cline, Roo Code, OpenHands, Aider, Continue — are the intended vehicle.
2. **Create an empty scratch directory**, one per model, e.g.
   `bench/my-model/`. Do not run from this repo folder.
3. **Copy only `bench-app-task.md`** into it. Nothing else.
4. **Send exactly one message** and nothing more:

   > Read bench-app-task.md and build what it specifies.

5. If the model stops early, reply with **"continue"** and nothing else. Every
   clarifying question it asks is a data point, not something to resolve.
6. **Record process metrics** (wall clock, turns, tool calls, failed turns,
   thinking blocks, whether it claimed success without running anything).
7. **Score it against `bench-app-rubric.md`** — never against the model's own
   passing test suite.

## Serving a local model

Any OpenAI-compatible endpoint works; the client matters more than the server.
Point the client at the endpoint, give it file-write and shell tools, and make
sure the context window is large enough to hold the task file plus the code it
is writing (16k is workable, 32k–64k is comfortable).

| Backend | Endpoint | Notes |
|---|---|---|
| **vLLM** | `http://localhost:8000/v1` | `vllm serve <model> --max-model-len 32768 --enable-auto-tool-choice --tool-call-parser hermes` for tool use |
| **llama.cpp** | `http://localhost:8080/v1` | `llama-server -m model.gguf --port 8080 --jinja` (template must emit tool calls) |
| **Ollama** | `http://localhost:11434/v1` | `ollama serve`; pick a tool-capable model tag |
| **LM Studio** | `http://localhost:1234/v1` | enable the local server; tool use varies by model |
| **LiteLLM proxy** | `http://localhost:4000/v1` | one config, many models; useful when comparing several at once |

Settings that matter, and belong in the run record:

- **Context / max tokens** — a truncated run is a void run; raise the cap.
- **Temperature** — 0 or near it; this is not a creativity test.
- **Reasoning effort** — where the backend supports it (`low`/`medium`/`high`).
- **Thinking blocks** — some transports expose them, some drop them. Note
  which; it changes wall-clock comparisons.
- **Quantization** — name the weights in the result (`Q4_K_M`, FP8, …).

Before launching, confirm the server is up: `curl http://localhost:8000/v1/models`.

## What it measures

Every run is scored out of **28**:

| Part | What | Points |
|---|---|---|
| A | Gate: does it run at all | Pass/fail — never points |
| B | 11 requirements × 2 | 22 |
| C | 6 quality checks × 1 | 6 |

Part A is checked **last**: `python -m unittest discover` runs clean,
`recon import` succeeds on the fixtures, `reconcile` produces output, and there
are zero third-party imports. A model that passes Part B on code that cannot
read the specified input is a failure, and the score says so.

Part B is scored **per requirement** against fixed expected values, not against
the model's tests. Score 2 = correct, 1 = defensible but wrong in an edge case,
0 = wrong or absent.

Two checks catch defects that a model's own green test suite hides:

- **Cross a process boundary.** Run `import`, `reconcile`, and `report` as three
  separate invocations and see whether they agree. Scoring matching from one
  in-process session is the exact blind spot that let one run's 47/47 suite pass
  while `reconcile` persisted nothing.
- **Check that correct code is reachable.** Grep each requirement's helper for
  call sites. One run implemented the correct most-recent-rate rule and never
  called it.

Also inspect **what the model's tests feed the parser**: a run can pass eleven
requirements while being unable to read the specified CSV shape, because its own
tests invented a format it likes better.

## The eleven traps and what they expose

Each trap has exactly one defensible answer. This is the part that matters for
local models — every row is a place where a "sensible default" is wrong.

| # | Requirement | Expected | What it exposes |
|---|---|---|---|
| 1 | Integer minor units | 1000 rows of `0.01` → exactly `10.00` | Float instinct on a money path; `0.01 * 1000 != 10.00` |
| 2 | Half-up rounding + allocation residual | `100.00` over 3 → `33.34, 33.33, 33.33`; `-2.345` → `-2.35` | Python's banker's `round()`; allocations that don't sum |
| 3 | CSV edges | BOM + CRLF + `"+1,234"` parses; field stays literal | "Clean the data" reflexes: BOM in header name, `=` coerced or stripped |
| 4 | Flag over inference | `03/04/2026` under `iso` is a hard error; `2026-02-29` names file+line+value | Date guessing (`dateutil`-style) and silent coercion to Mar 1 |
| 5 | Rate as of *issued*, most recent on/before | invoice `03-10`, rates `03-01`/`03-15` → uses `03-01` | Exact-date lookup; using the transaction date instead |
| 6 | Deterministic order + subset-sum + tie-breaks | reversed file order gives identical output; 2-invoice solution beats 3-invoice | File-order greedy; single-invoice-only matching |
| 7 | Refund reverses, never settles | refund of a settled transaction returns invoices to open | The "refund = negative payment" prior; normalizing text the spec says must match exactly |
| 8 | Content-keyed dedup | import twice → `0 inserted`; two identical rows in one file stay two | Dedup on `id` or row order; collapsing duplicates within a file |
| 9 | Atomic import + atomic write | `--fail-after 3` leaves the row count unchanged | Per-row commits; in-place report writes |
| 10 | `--as-of` is the only clock | `due = as-of - 30` → `0-30`; `-31` → `31-60` | `date.today()` scattered through the code; tests that only pass today |
| 11 | Exit codes + stream split | unmatched present → exit `1`, report on stdout only | Logging to stdout; exiting `0` always |

Requirements 6 and 7 interact on purpose: a refund returns invoices to open, and
whether later transactions can claim them depends on processing order. Handling
both in isolation but not together is the most interesting failure this task
produces.

## The fixtures

The fixture set is kept **outside** this repo and outside every scratch
directory, then copied in, so every model sees byte-identical input. Build it
once, deterministically — explicit line endings, explicit BOM — and reuse it.
`bench-app-rubric.md` lists the nine cases and the exact expected outputs.

The short version: `penny.csv`, `alloc.csv`, `dirty.csv`, `baddate.csv`,
`rates.csv`, `subset.csv`, `refund.csv`, `dupes.csv`, `aging.csv`, with the two
input shapes from the task (`transactions.csv`, `invoices.csv`) and a
`rates.csv` of `date,currency,rate_to_usd`.

## Publishing your own results

This repo doesn't publish a model ranking. The score is a property of *your*
weights, quantization, backend, context cap, and client settings — not of the
benchmark — so the useful result is the one you measure yourself. Record it in
this shape:

| Run | Model + weights | Backend / client | Context cap | Part B | Part C | Total | Gate |
|---|---|---|---|---|---|---|---|
| 1 | *(model id, quantization)* | *(vLLM / Cline, …)* | *(tokens)* | /22 | /6 | /28 | passed / failed |

Include the failing checks and quote the evidence (the wrong `33.34` split, the
`reconcile` that printed nothing in a new process, the helper that exists but is
never called). A failed gate is a legitimate result — report it beside the
score, not folded into it.

Two cautions when comparing your own runs:

- Expect **~2 points of run-to-run noise** on the same weights and settings.
  Anything smaller is not a difference.
- Compare **points per turn**, not wall-clock seconds. Transports differ in
  whether thinking blocks are preserved, and local hardware varies by an order
  of magnitude in decode speed.

## Repo layout

| File | Show to the model? | What it is |
|---|---|---|
| [`bench-app-task.md`](bench-app-task.md) | **Yes — and only this** | The task spec: build `recon`, Python 3.11+, standard library only |
| [`bench-app-prompt.md`](bench-app-prompt.md) | No | Operator runbook: how to launch a run, what to record, why each trap discriminates |
| [`bench-app-rubric.md`](bench-app-rubric.md) | No | Grading sheet: fixed expected values, per-requirement checks, results template |
| [`bench-app-grading.md`](bench-app-grading.md) | No | Isolation setup: folder layout, operator grading, grading with a second model |
| `README.md` | No | This overview |

**Only `bench-app-task.md` goes to the model.** Every other file in this repo —
including this README, the runbook, the rubric, and the grading guide — names
intended traps or expected values and is operator/grading material. A model that
reads any of them is **contaminated** and the run is void.

## Keeping operator files away from the model

The tested model gets **one folder, one file**: an empty run directory
containing only `bench-app-task.md`. Everything else — this README, the runbook,
the rubric, the fixtures, and the probe harness — stays outside the run
directory and is opened only after the run finishes (or, for grading, in a
separate grading folder).

Two supported setups, both described in
[`bench-app-grading.md`](bench-app-grading.md):

- **Operator grading** — a human scores the finished code against the rubric.
- **Second-model grading** — copy the produced tree into `grading/<model>/` and
  launch a grader agent there with the rubric and fixtures. The tested model
  never sees them; a different model for the grader avoids self-preference bias.

Practical rules: don't run the benchmark inside this repo folder (parent
directories and repo-wide instruction files get auto-read by some clients);
disable network access for the tested model; and never paste rubric contents
into the chat.

## Contamination

Before scoring, inspect the model's tool calls. If it read any repo file other
than `bench-app-task.md` — the runbook, the rubric, this README, or the grading
guide — discard the run. Keeping each scratch directory to *only*
`bench-app-task.md` is the cheapest prevention — do not run the benchmark from
this repo folder.

A model that announces it is "setting up benchmark infrastructure" has read the
wrong file: it saw operator material, not the task. Stop and restart clean.

## Contributing a run

A useful run report includes: model id and weights (quantization), backend and
client, context cap, temperature, reasoning effort, the recorded process
metrics, and the Part A/B/C scores with the failing checks quoted. Runs with a
failed gate are welcome — label the gate result beside the score. See
`bench-app-rubric.md` for the format.

## Why this name

The kit was originally filed as "Logic Benchmark". That name described the wrong
thing: this test scores **specification fidelity under deliberate traps** —
careful reading, exact arithmetic, correct edge cases — not formal logical
reasoning. It also collided with [LogicBench](https://arxiv.org/abs/2404.15522),
an existing ACL 2024 logical-reasoning benchmark.

**SpecTrapBench** names both halves of what is measured: the exact specification
and the eleven planted traps. The score moves precisely on whether a model read
the spec or inferred around it.
