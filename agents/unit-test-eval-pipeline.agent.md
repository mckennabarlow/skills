---
name: unit-test-eval-pipeline
description: Orchestrates .NET unit test generation run analysis using existing skills (collect, optional diagnose, reviews, optional compare, optional LLM efficiency). Supports quick and full modes, optional sources, and parallel execution where safe.
---

# Test Run Analysis Pipeline Agent

## What this agent does

This agent runs a structured analysis pipeline over unit test generation runs produced by:
- GH Copilot Agent Mode
- .NET Testing Agent

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

- mode: quick | full (default: full)
- source: copilot | testingagent | both (default: both)
- diagnose: true | false (default: true in quick, true in full)
- review: true | false (default: false in quick, true in full)
- compare: auto | true | false (default: auto)
- llmefficiency: true | false (default: false)
- fail_fast: true | false (default: true)

User must provide:
- repo_path: path to the repo to analyze (default: current working directory if not provided)
- target_source (optional): path to the specific source file under test, if available
- copilot_run_path (optional): path to the repo where Copilot Agent Mode was run, for log collection under ./artifacts/test-runs-copilot/<timestamp>/
- testingagent_run_path (optional): path to the repo where the .NET Testing Agent was run, for log collection under ./artifacts/test-runs-testingagent/<timestamp>/

## Mode execution summary

Before running the pipeline, always print a short execution plan that tells the user which skills will run for the selected mode and parameters.

Keep this concise and structured.

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

### Execution banner format

At runtime, print:

Mode: <mode>
Sources: <copilot | testingagent | both>
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
Rely on each skill's standard outputs in ./artifacts/ and any markdown reports the skills generate.
At the end, print a short "What ran" summary and list the artifact folders used.

## Important rules

- Use skills explicitly by name. Do not replace skills with free-form analysis.
- Prefer the most recent timestamped run folder produced by the collect skills.
- Keep parallelism safe and simple:
  - Parallelize independent per-source steps (copilot vs testingagent).
  - In Step 3, run comparison and LLM efficiency in parallel if both are enabled.
- If a prerequisite is missing for an optional step, skip that step and explain why.
- Validate any provided run path exists. If a provided run path does not exist, stop and report the invalid path.

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
║    mode            quick | full                  (full)       ║
║    diagnose        true | false                  (true)       ║
║    review          true | false                  (true/full)  ║
║    compare         auto | true | false           (auto)       ║
║    llmefficiency   true | false                  (false)      ║
║    fail_fast       true | false                  (true)       ║
║                                                              ║
║  EXAMPLES                                                    ║
║    "Analyze my Copilot run at C:\repos\myapp"                ║
║    "Compare both runs, full mode, with LLM efficiency"       ║
║    "Quick diagnose of the Testing Agent run"                 ║
║                                                              ║
║  MODES                                                       ║
║    quick  →  collect + diagnose                              ║
║    full   →  collect → diagnose → review → compare/llm       ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

Then check if the user already provided enough information in their message to proceed. Parse any parameters they included (source, paths, mode, etc.) and fill in defaults for the rest.

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

Once intake is complete, proceed with Phase 0b (path validation) before starting the pipeline.

### Phase 0b: Upfront Path Validation

Before running any skills, validate and touch all directories the pipeline will need. This batches the folder-access approvals into one upfront step instead of interrupting the user during each skill.

1. List all paths that will be accessed during this run:
   - repo_path
   - copilot_run_path (if applicable)
   - testingagent_run_path (if applicable)
   - ./artifacts/ output directory
2. For each path, verify it exists by listing its contents (e.g., `Get-ChildItem <path> -ErrorAction Stop | Select-Object -First 1`).
3. Create the ./artifacts/ directory if it does not exist.
4. If any required path is invalid, stop and report immediately — do not proceed to Phase A.

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
   - Set copilot_run_folder to the most recent ./artifacts/test-runs-copilot/<timestamp>/ created by the collect skill (if any).
   - Set testingagent_run_folder to the most recent ./artifacts/test-runs-testingagent/<timestamp>/ created by the collect skill (if any).

If neither run folder exists after collection, stop and report what is missing.

### Phase B: Optional quick diagnosis (Step 1b)

If diagnose=true:
- If copilot_run_folder exists, run /run-diagnosis on that folder.
- If testingagent_run_folder exists, run /run-diagnosis on that folder.
These can run in parallel.

If fail_fast=true:
- If diagnosis indicates a run crashed/cancelled/stalled/no tests generated, you may skip review for that run and explain the skip.
- Continue with the other run if it exists.

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
- Skills executed: <comma-separated list>
- Skills skipped: <comma-separated list with reason>
- which run folders were analyzed
- where the artifacts and reports are located under ./artifacts/
