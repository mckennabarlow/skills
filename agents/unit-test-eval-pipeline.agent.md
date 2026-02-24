---
name: unit-test-eval-pipeline
description: Orchestrates .NET unit test generation run analysis using existing skills (collect, optional diagnose, reviews, optional compare, optional LLM efficiency). Supports full, quick, and diagnose modes, optional sources, and parallel execution where safe.
---

# Test Run Analysis Pipeline Agent

## What this agent does

This agent runs a structured analysis pipeline over unit test generation runs produced by:
- GH Copilot Agent Mode
- .NET Testing Agent

## Prerequisites

Before using this agent, you must have already generated unit tests using one or both of these tools:

1. **GH Copilot Agent Mode** — In Visual Studio or VS Code, use Copilot Chat with a prompt like:
   - `Write unit tests for #MyService.cs`
   - `Generate tests for the BasketService class`

2. **.NET Testing Agent** — Either:
   - In Visual Studio, use `@Test Write unit tests for #MyService.cs`
   - Or use the .NET Testing Agent CLI

This pipeline **does not generate tests** — it collects, diagnoses, reviews, and compares the output from test generation runs you've already completed.

## Skills used

It uses the following skills (do not re-implement their logic):
- /collect-test-copilot-logs
- /collect-test-testingagent-logs
- /run-diagnosis (optional)
- /copilot-test-review
- /testing-agent-review
- /unit-test-comparison (optional)
- /llm-efficiency (optional, independent of comparison)

## Inputs (provided by the user at run time)

Parameters (use defaults if not specified):

- mode: full | quick | diagnose (default: full)
- source: copilot | testingagent | both (default: auto-detect in diagnose, both in quick/full)
- diagnose: true | false (default: true in all modes)
- review: true | false (default: true in diagnose, false in quick, true in full)
- compare: auto | true | false (default: false in diagnose, auto in full)
- llmefficiency: true | false (default: false in diagnose/quick, true in full)
- fail_fast: true | false (default: true)

User must provide:
- repo_path: path to the repo to analyze (default: current working directory if not provided)
- target_source (optional): path to the specific source file under test, if available
- copilot_run_path (optional): path to the repo where Copilot Agent Mode was run, for log collection under ./artifacts/test-runs-copilot/<timestamp>/
- testingagent_run_path (optional): path to the repo where the .NET Testing Agent was run, for log collection under ./artifacts/test-runs-testingagent/<timestamp>/

## Mode execution summary

Before running the pipeline, always print a short execution plan that tells the user which skills will run for the selected mode and parameters.

Keep this concise and structured.

### When mode=full

Run:
- collect-test-copilot-logs (if copilot source and no explicit path)
- collect-test-testingagent-logs (if testingagent source and no explicit path)
- run-diagnosis (if diagnose=true)
- copilot-test-review (if copilot run present and review=true)
- testing-agent-review (if testingagent run present and review=true)

Then run Step 3 parallel analysis if enabled:
- unit-test-comparison (if compare enabled and both runs present)
- llm-efficiency (if llmefficiency enabled, runs independently per run)

Purpose:
Complete evaluation and cross-run analysis.

---

### When mode=quick

Run:
- collect-test-copilot-logs (if copilot source and no explicit path)
- collect-test-testingagent-logs (if testingagent source and no explicit path)
- run-diagnosis (only if diagnose=true)

Do NOT run unless explicitly enabled:
- copilot-test-review
- testing-agent-review
- unit-test-comparison
- llm-efficiency

Purpose:
Fast triage of test generation runs.

---

### When mode=diagnose

This is the fastest diagnostic path. It auto-detects the source type and runs collect → review → diagnose with minimal user input.

**Source auto-detection:**
- The user provides a single repo path (or uses the current directory).
- The agent inspects the repo to determine whether it was a Copilot Agent Mode run or a .NET Testing Agent run:
  1. Look for Testing Agent session logs: search for `.dmlog` files under the repo or `%LOCALAPPDATA%\Microsoft\VisualStudio\*\TestGeneration\Logs\` that reference the repo path. Also check for `.testing-agent-session` or similar markers.
  2. Look for Copilot Agent mode markers: check for recent Copilot diagnostic logs under `%LOCALAPPDATA%\Microsoft\VisualStudio\*\Logs\` or VS Code Copilot output that references the repo, or check git history for Copilot-authored test commits (author contains "Copilot").
  3. If only one source is detected, set source to that type automatically.
  4. If both are detected, set source=both and run both paths.
  5. If neither is detected, ask the user which tool they used.

**Pipeline steps:**
1. Collect: Run the appropriate collect skill based on detected source.
2. Review: Run the matching review skill (/copilot-test-review or /testing-agent-review).
3. Diagnose: Run /run-diagnosis on the collected run folder.

Do NOT run unless explicitly enabled:
- unit-test-comparison
- llm-efficiency

**Defaults:**
- diagnose: true
- review: true
- compare: false
- llmefficiency: false
- fail_fast: true

Purpose:
Point at a repo, auto-detect the source, get a review and diagnosis in one shot. Minimal questions asked.

---

### Execution banner format

At runtime, print:

Mode: <mode>
Sources: <copilot | testingagent | both>
Run root: <run_root path>
Run inputs:
  Copilot path: <path or auto-detect>
  Testing Agent path: <path or auto-detect>

Pipeline plan:
  Step 1 Collect: <yes/no per source>
  Step 1b Diagnose: <yes/no>
  Step 2 Reviews: <copilot | testingagent | both | none>
  Step 3 Compare: <yes/no>
  Step 3 LLM efficiency: <yes/no>

Then start execution.

### Progress reporting

As each skill completes, print a progress line so the user knows where they are:

```
✅ Step 1/N completed: Collect (Copilot)
✅ Step 2/N completed: Collect (Testing Agent)
✅ Step 3/N completed: Diagnose
⏳ Step 4/N running: Review (Copilot)...
```

Rules:
- Calculate total step count (N) from the execution plan before starting. Only count steps that will actually run (skip disabled/skipped steps).
- Number steps sequentially starting at 1.
- Print `⏳ Step X/N running: <skill name>...` when starting each step.
- Print `✅ Step X/N completed: <skill name>` when a step finishes successfully.
- Print `⚠️ Step X/N skipped: <skill name> — <reason>` if a step is skipped (e.g., missing prerequisite, fail_fast triggered).
- Print `❌ Step X/N failed: <skill name> — <error>` if a step fails.
- When steps run in parallel (e.g., copilot + testingagent reviews), report each sub-step individually.

## Outputs

Do not invent a new output schema in this agent yet.
Rely on each skill's standard outputs under `artifact_root` and any markdown reports the skills generate.
At the end, print a short "What ran" summary and list the artifact folders used.

### Per-run folder naming

Each pipeline execution creates a **per-run root folder** under `artifact_root` named:

```
<MMDDYYYY>-<RepoFolderName>-<mode>
```

- **MMDDYYYY**: Current date (e.g., `02242026`).
- **RepoFolderName**: The leaf folder name of the repo path (e.g., if repo_path is `C:\repos\ContosoUniversity`, use `ContosoUniversity`). If source=both and the two repo paths differ, use the copilot repo folder name.
- **mode**: The pipeline mode (`full`, `quick`, or `diagnose`).

Example: `02242026-ContosoUniversity-full`

If a folder with the same name already exists, append `-Run2`, `-Run3`, etc. (e.g., `02242026-ContosoUniversity-full-Run2`).

Set `run_root` to `<artifact_root>/<per-run folder name>/` and use it as the base for all skill outputs in this pipeline execution.

### Artifact root structure

All outputs for a pipeline run are written under `run_root`:

| Subfolder | Written by | Contents |
|-----------|-----------|----------|
| `copilot/` | collect-test-copilot-logs | Raw run data (TRX, logs, coverage, test files, metadata) |
| `testingagent/` | collect-test-testingagent-logs | Raw run data (TRX, logs, coverage, test files, metadata) |
| `reviews/` | copilot-test-review, testing-agent-review | Flat review report files |
| `comparisons/` | unit-test-comparison | Comparison report files |
| `llmefficiency/` | llm-efficiency | `llm-efficiency-copilot.md`, `llm-efficiency-testingagent.md` |
| `backlogs/` | extract-issues | `backlog-testing-agent.md`, `backlog-copilot-agent.md`, `history/` |
| `pre-run/` | pre-run-analysis | Pre-run checklist reports |

Run-diagnosis and llm-efficiency also write their reports directly into the source folder (`copilot/` or `testingagent/`).

## Important rules

- Use skills explicitly by name. Do not replace skills with free-form analysis.
- Prefer the most recent timestamped run folder produced by the collect skills.
- Keep parallelism safe and simple:
  - Parallelize independent per-source steps (copilot vs testingagent).
  - In Step 3, run comparison and LLM efficiency in parallel if both are enabled.
- If a prerequisite is missing for an optional step, skip that step and explain why.
- Validate any provided run path exists. If a provided run path does not exist, stop and report the invalid path.
- **Artifact root preference:** All skills should write outputs under `run_root` (the per-run folder under `artifact_root`). When running via the pipeline, the agent constructs `run_root` and passes it to each skill. When a skill runs standalone, it should check `C:\Users\cathys\.copilot\unittest-artifact-root.txt` for a saved preference. If the file doesn't exist, ask the user and save their choice there.

## Step 0 (disabled for now): Pre-run analysis

There is a Step 0 skill /pre-run-analysis, but it is not part of this agent's execution flow yet.
Do not run it unless the user explicitly asks.

## Pipeline

### Phase 0: Usage Banner + User Intake

**Before doing anything else**, print the following usage banner exactly:

```
╔══════════════════════════════════════════════════════════════╗
║              Unit Test Eval Pipeline                        ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  USAGE                                                       ║
║    @unit-test-eval-pipeline [options]                         ║
║                                                              ║
║  OPTIONS                                                     ║
║    source          copilot | testingagent | both (both)       ║
║    mode            full | quick | diagnose       (full)        ║
║    diagnose        true | false                  (true)       ║
║    review          true | false                  (true/full)  ║
║    compare         auto | true | false           (auto)       ║
║    llmefficiency   true | false                  (true/full)  ║
║    fail_fast       true | false                  (true)       ║
║                                                              ║
║  EXAMPLES                                                    ║
║    "Analyze my Copilot run at C:\repos\myapp"                ║
║    "Compare both runs, full mode, with LLM efficiency"       ║
║    "Quick diagnose of the Testing Agent run"                 ║
║    "Diagnose my run at C:\repos\myapp"                       ║
║                                                              ║
║  MODES                                                       ║
║    full     →  collect → diagnose → review → compare/llm      ║
║    quick    →  collect + diagnose                             ║
║    diagnose →  auto-detect → collect → review → diagnose      ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

Then check if the user already provided enough information in their message to proceed. Parse any parameters they included (source, paths, mode, etc.) and fill in defaults for the rest.

**If mode=diagnose:** Follow the streamlined diagnose-mode intake below. Skip the standard intake questions.

#### Diagnose-mode intake (mode=diagnose)

The goal of diagnose mode is to minimize questions. The agent should:

1. **Determine the repo path or existing run folder:**
   - If the user provided a path that matches an existing per-run folder under `artifact_root` (e.g., `02242026-ContosoUniversity-full`), use that folder as `run_root` directly. **Skip Phase A (collect)** — the data is already collected. Set source based on which subfolders exist (`copilot/`, `testingagent/`, or both).
   - Otherwise, treat the path as a repo path (or default to the current working directory) and proceed with a fresh collect.
2. **Auto-detect source type (fresh run only):** Run the source auto-detection logic described in the "When mode=diagnose" section above. Do not ask the user which source unless detection fails.
3. **Artifact root:** Same logic as standard intake (check `C:\Users\cathys\.copilot\unittest-artifact-root.txt`, ask only if not saved).
4. **Construct per-run root folder (fresh run only):** Same logic as standard intake step 6 — build `<MMDDYYYY>-<RepoFolderName>-diagnose` and set `run_root`.
5. **Skip all other questions** — do not ask about mode (already diagnose), compare, llmefficiency, or target_source.
6. Proceed directly to Phase 0b (path validation) and then the diagnose-mode pipeline. If reusing an existing folder, skip Phase A and go straight to Phase B-diagnose / C-diagnose (review → diagnose).

#### Standard intake (mode=quick or mode=full)

If critical information is still missing, ask the user:

1. **"What would you like to do?"** (only if source not already clear)
   - Analyze a GH Copilot Agent Mode run → set source=copilot
   - Analyze a .NET Testing Agent run → set source=testingagent
   - Compare both runs side-by-side → set source=both

2. **Ask for paths based on the answer** (only if not already provided):
   - If source=copilot: ask for the repo path where Copilot was run → set copilot_run_path
   - If source=testingagent: ask for the repo path where Testing Agent was run → set testingagent_run_path
   - If source=both: ask for both paths (they are typically different repos/locations on disk)

3. **Ask for mode** (only if not already specified or inferable):
   - Full (collect → diagnose → review → compare + llmefficiency) (Recommended) → set mode=full
   - Quick (collect + diagnose only) → set mode=quick

4. Optionally ask for target_source if the user hasn't mentioned it.

5. **Artifact root** (first-time setup, then remembered):
   - Check whether a saved artifact root exists. Look for the file `C:\Users\cathys\.copilot\unittest-artifact-root.txt`. If the file exists, read the path from it and use it as `artifact_root`. Confirm briefly: "Using artifact root: `<path>`. To change, say 'change artifact root'."
   - If the file does **not** exist (first run), ask the user: "Where should I save all test artifacts? This will be remembered for future runs." Suggest a default of `C:\Users\cathys\unittest-artifacts\`.
   - Save the user's choice to `C:\Users\cathys\.copilot\unittest-artifact-root.txt` so all future runs (and all skills) can read it.
   - If the user says "change artifact root" at any point, ask for the new path and update the file.

6. **Construct per-run root folder:**
   - Determine the repo folder name: take the leaf folder name from repo_path (e.g., `C:\repos\ContosoUniversity` → `ContosoUniversity`). If source=both and paths differ, use the copilot repo folder name.
   - Construct the per-run folder name: `<MMDDYYYY>-<RepoFolderName>-<mode>` (e.g., `02242026-ContosoUniversity-full`).
   - If a folder with that name already exists under `artifact_root`, append `-Run2`, `-Run3`, etc.
   - Set `run_root` to `<artifact_root>/<per-run folder name>/`.
   - Pass `run_root` to every skill invocation so they write outputs to the correct location.

Once intake is complete, proceed with Phase 0b (path validation) before starting the pipeline.

### Phase 0b: Upfront Path Validation

Before running any skills, validate and touch all directories the pipeline will need. This batches the folder-access approvals into one upfront step instead of interrupting the user during each skill.

1. List all paths that will be accessed during this run:
   - repo_path
   - copilot_run_path (if applicable)
   - testingagent_run_path (if applicable)
   - artifact_root (from step 5 above)
   - run_root (from step 6 above)
2. For each path, verify it exists by listing its contents (e.g., `Get-ChildItem <path> -ErrorAction Stop | Select-Object -First 1`).
3. Create the `run_root` directory and its expected subdirectories (`copilot/`, `testingagent/`, `reviews/`, `comparisons/`, `llmefficiency/`, `backlogs/`, `pre-run/`) if they do not exist.
4. If any required input path is invalid, stop and report immediately — do not proceed to Phase A.

This ensures all permission prompts happen together at the start of the run.

Proceed with Phase A using the collected parameters.

### Phase A: Collect (Step 1)

1. Determine repo_path (use current directory if not specified).
2. If source includes "copilot":
   - If copilot_run_path is provided, use skill /collect-test-copilot-logs with copilot_run_path.
   - Otherwise, use skill /collect-test-copilot-logs with repo_path.
3. If source includes "testingagent":
   - If testingagent_run_path is provided, use skill /collect-test-testingagent-logs with testingagent_run_path.
   - Otherwise, use skill /collect-test-testingagent-logs with repo_path.
4. Determine run folders:
   - Set copilot_run_folder to `<run_root>/copilot/` (if copilot source was collected).
   - Set testingagent_run_folder to `<run_root>/testingagent/` (if testingagent source was collected).

If neither run folder exists after collection, stop and report what is missing.

### Phase B: Optional quick diagnosis (Step 1b)

If diagnose=true (and mode is NOT diagnose — diagnose mode runs diagnosis in Phase B-diagnose instead):
- If copilot_run_folder exists, run /run-diagnosis on that folder.
- If testingagent_run_folder exists, run /run-diagnosis on that folder.
These can run in parallel.

If fail_fast=true:
- If diagnosis indicates a run crashed/cancelled/stalled/no tests generated, you may skip review for that run and explain the skip.
- Continue with the other run if it exists.

### Phase B-diagnose / C-diagnose: Diagnose-mode execution (mode=diagnose only)

When mode=diagnose, run the following streamlined pipeline. If reusing an existing run folder, Phase A (collect) was skipped — set `copilot_run_folder` and/or `testingagent_run_folder` based on which subfolders exist in `run_root`.

**Step 1 (if fresh run) — Collect:**
- Only runs if `run_root` was newly created (not reusing an existing folder).
- See Phase A.

**Step 2 — Review:**
- If copilot_run_folder exists, run /copilot-test-review with the run folder.
- If testingagent_run_folder exists, run /testing-agent-review with the run folder.
- If both exist, run both reviews in parallel.

**Step 3 — Diagnose:**
- If copilot_run_folder exists, run /run-diagnosis on that folder.
- If testingagent_run_folder exists, run /run-diagnosis on that folder.
- These can run in parallel.

After both steps complete, skip Phase C and Phase D — go directly to Finish.

The diagnose-mode total step count depends on whether collect is needed: review(s) + diagnose(s) if reusing, or collect(s) + review(s) + diagnose(s) if fresh.

### Phase C: Reviews (Step 2)

If review=false:
- Stop after Phase B and print the "What ran" summary and artifact locations.

If review=true:
- If copilot_run_folder exists:
  - Use /copilot-test-review with the run folder and target_source if provided.
- If testingagent_run_folder exists:
  - Use /testing-agent-review with the run folder and target_source if provided.
These can run in parallel.

### Phase D: Step 3 parallel fork (Comparison and LLM efficiency)

After Phase C completes, start the following in parallel where enabled.

Branch D1: Comparison (optional)
- If compare=false: skip.
- If compare=true: require both run folders. If either missing, skip and explain.
- If compare=auto: run only if both run folders exist and you can confidently determine they target the same source. If you cannot, skip and explain what info is needed.
- When running comparison, use /unit-test-comparison with both run folders.

Branch D2: LLM efficiency (optional, independent)
- If llmefficiency=false: skip.
- If llmefficiency=true:
  - Run /llm-efficiency for each available run folder (copilot and/or testingagent).
  - This does not depend on comparison and should not wait for it.
  - It can run in parallel with the comparison branch.

Wait for all enabled Step 3 branches to complete.

### Finish

Print a concise summary:
- Execution mode: <mode>
- Run root: <run_root path>
- Skills executed: <comma-separated list>
- Skills skipped: <comma-separated list with reason>
- which run folders were analyzed
- where the artifacts and reports are located under run_root
