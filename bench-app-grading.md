# Grading setup — keeping operator files away from the tested model

The one hard rule of SpecTrapBench: **the model under test may only see
`bench-app-task.md`.** The rubric, the runbook, the README, and this guide all
name planted traps and expected values, so a single read of any of them voids the
run. Grading material is for the grader — human or model — and it lives outside
the tested model's workspace.

This file describes two supported setups: operator grading, and grading with a
second model. Both rely on the same isolation: **one folder per tested model,
containing only the task file**, and a separate grading folder that the tested
model cannot see.

## The folder layout

Keep the kit, the fixtures, and the grading material in one place, and give the
tested model its own empty directory:

```
C:\Coding\bench\                     # or ~/bench on POSIX
├── kit\                             # this repo
│   ├── bench-app-task.md            # the only file the tested model may have
│   ├── bench-app-prompt.md          # operator runbook — never shown to the model
│   └── bench-app-rubric.md          # grading sheet — never shown to the model
├── fixtures\                        # generated once, byte-identical for every run
│   ├── penny\  alloc\  dirty\ ...   # transactions.csv + invoices.csv (+ rates.csv)
│   └── harness\                     # scoring driver + probes
├── runs\
│   └── model-a\              # tested model's working directory
│       └── bench-app-task.md        # <-- the ONLY file in here at launch
└── grading\
    └── model-a\              # grading workspace (rubric lives here)
        ├── bench-app-rubric.md
        ├── produced\                # copy of the model's finished tree
        └── fixtures\                # copy of the fixture set
```

`runs\<model>\` is the tested model's `cwd`. `grading\<model>\` is where the
rubric, the fixtures, and the probe harness live, and it is opened **after** the
run finishes. Nothing under `grading\` is ever visible from `runs\<model>\`.

## Preparing a run

1. Copy the task file into a fresh run directory — nothing else:

   ```powershell
   # PowerShell
   New-Item -ItemType Directory -Path "C:\Coding\bench\runs\model-a" -Force
   Copy-Item "C:\Coding\bench\kit\bench-app-task.md" "C:\Coding\bench\runs\model-a\"
   ```

   ```bash
   # POSIX
   mkdir -p ~/bench/runs/model-a
   cp ~/bench/kit/bench-app-task.md ~/bench/runs/model-a/
   ```

2. Launch the tested model with that directory as its workspace root, using a
   client that can write files and run shell commands (Cline, Roo Code,
   OpenHands, Aider, Continue). Configure the client so it does **not**:

   - add the kit repo or any parent folder as extra workspace context,
   - auto-read files outside the run directory,
   - load repo-wide instruction files (`.cursorrules`, `CLAUDE.md`,
     `AGENTS.md`, `.github/copilot-instructions.md`) — the run directory should
     contain none of these,
   - have network access to fetch files from GitHub.

   If your client supports a sandbox or a "workspace only" file policy, turn it
   on. The cleanest sandbox is a directory that contains nothing but
   `bench-app-task.md`.

3. Send exactly this one message:

   > Read bench-app-task.md and build what it specifies.

   Nothing else. If it stops early, reply only with "continue". Do not paste the
   task contents into the chat, do not describe requirements, and never paste
   the rubric or the runbook into the conversation.

## Operator grading

When the run finishes:

1. Copy the model's produced tree into the grading folder:

   ```powershell
   New-Item -ItemType Directory -Path "C:\Coding\bench\grading\model-a" -Force
   Copy-Item "C:\Coding\bench\runs\model-a\*" "C:\Coding\bench\grading\model-a\produced\" -Recurse -Exclude "bench-app-task.md"
   ```

2. Copy `bench-app-task.md`? **No** — not needed for grading. Copy the fixtures
   instead, one case at a time, into a scratch directory inside `grading\`.

3. Open `bench-app-rubric.md` with your editor and score Part B first, then
   Part A, then Part C, then record Part D. Run `import`, `reconcile`, and
   `report` as separate processes (never one in-process session), and grep each
   requirement's helper for call sites.

4. Delete or archive the run directory afterwards. Keep the grading folder for
   auditing; it contains the rubric and is not a benchmark workspace.

## Grading with a second model

This is the "one model grades another" setup. The tested model produced code in
`runs\<model>\`; a grader agent — ideally a **different model** than the one
under test, so self-preference bias can't creep in — works in
`grading\<model>\`, where the rubric and fixtures live. The grader never talks to
the tested model, and the tested model never sees the grader's workspace.

### Prepare the grading workspace

1. Copy the produced tree in, as above.
2. Copy the fixture set in:

   ```powershell
   New-Item -ItemType Directory -Path "C:\Coding\bench\grading\model-a\fixtures" -Force
   Copy-Item "C:\Coding\bench\fixtures\*" "C:\Coding\bench\grading\model-a\fixtures\" -Recurse
   ```

3. Copy `bench-app-rubric.md` into the grading workspace. It must **not** exist
   in the run directory, in the tested model's client configuration, or in any
   chat context.

### Grader prompt

Launch the grader agent with `grading\<model>\` as its workspace and give it
something like this (adapt the details, keep the rules):

> You are grading a coding run. The code is in `./produced`; the grading sheet
> is `bench-app-rubric.md`; fixtures are in `./fixtures`. Follow the rubric
> exactly.
>
> - Do not trust the model's own test suite. Score Part B against the rubric's
>   fixed expected values, one requirement at a time: 2 = correct, 1 = defensible
>   but wrong in an edge case, 0 = wrong or absent.
> - Run `import`, `reconcile`, and `report` as three separate processes for each
>   fixture case and check they agree. Do not grade matching from a single
>   in-process session.
> - Check that the code that implements each requirement is actually called;
>   existence is not use.
> - After Part B, run the Part A gate: `python -m unittest discover`, import on
>   the fixtures, reconcile output, and zero third-party imports. Report
>   pass/fail beside the score, never as points.
> - Then score Part C and record Part D.
> - Output the completed rubric tables with evidence, the failing checks quoted,
>   and the total out of 28 with the gate result.

### Why a separate model

- The tested model's context never includes the rubric, so no contamination is
  possible at runtime.
- A different grader model is less likely to rationalize the tested model's
  choices; the rubric's fixed values are the only authority.
- The grading run is reproducible: the same produced tree and the same fixtures
  can be re-graded by another model and compared.

## Contamination checks (run before you trust a score)

- Diff the run directory's contents: at launch it should be **only**
  `bench-app-task.md` (plus whatever the model created itself).
- Inspect the tested model's tool calls for reads of `bench-app-rubric.md`,
  `bench-app-prompt.md`, `README.md`, `bench-app-grading.md`, or any path
  containing `grading` / `kit` / `fixtures`.
- Inspect the client's own file reads: some clients auto-read a parent
  directory's `README.md`, `.git`, or instruction files. A run directory nested
  inside the kit repo is a contamination risk — even the kit's own README gives
  away the traps.
- If the model announces that it is "setting up benchmark infrastructure", it
  has seen operator material, not just the task. Discard the run.
- Network fetches count: a model that clones this repo or downloads files from
  GitHub has read the rubric even if it never opened a local copy.

## Quick checklist

- [ ] Run directory exists and contains exactly `bench-app-task.md` at launch.
- [ ] The README, runbook, rubric, and grading guide live outside the run directory.
- [ ] Client configured for workspace-only reads; no repo-wide instruction files.
- [ ] One launch message, then only "continue".
- [ ] Fixture set generated once, byte-identical per run, kept outside the run.
- [ ] Scoring done in a separate folder, by an operator or a second model.
- [ ] Part B before Part A; cross-process runs; reachability checked.
- [ ] Gate reported beside the score, not folded into it.
