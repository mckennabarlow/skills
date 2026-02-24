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
- efficiency: true | false (default: false)
- fail_fast: true | false (default: true)

User must provide:
- repo_path: path to the repo to analyze (default: current working directory if not provided)
- target_source (optional): path to the specific source file under test, if available

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

## Step 0 (disabled for now): Pre-run analysis

There is a Step 0 skill /pre-run-analysis, but it is not part of this agent's execution flow yet.
Do not run it unless the user explicitly asks.

## Pipeline

### Phase A: Collect (Step 1)

1. Determine repo_path (use current directory if not specified).
2. If source includes "copilot", run in parallel with the testingagent collection:
   - Use skill /collect-test-copilot-logs with repo_path.
3. If source includes "testingagent", run in parallel with the copilot collection:
   - Use skill /collect-test-testingagent-logs with repo_path.
4. After both collection tasks finish (or the selected one finishes), identify:
   - copilot_run_folder (most recent ./artifacts/test-runs-copilot/<timestamp>/ if created)
   - testingagent_run_folder (most recent ./artifacts/test-runs-testingagent/<timestamp>/ if created)

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
- If efficiency=false: skip.
- If efficiency=true:
  - Run /llm-efficiency for each available run folder (copilot and/or testingagent).
  - This does not depend on comparison and should not wait for it.
  - It can run in parallel with the comparison branch.

Wait for all enabled Step 3 branches to complete.

### Finish

Print a concise summary:
- mode and parameters used
- which run folders were analyzed
- which skills ran (and which were skipped, and why)
- where the artifacts and reports are located under ./artifacts/
