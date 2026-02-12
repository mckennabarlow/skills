# Testing Skills

A collection of [Copilot Skills](https://docs.github.com/en/copilot/copilot-extensions/copilot-skills) for reviewing and comparing unit test output from the .NET Testing Agent and GH Copilot Agent Mode. These skills extend GitHub Copilot CLI with capabilities for collecting test artifacts, analyzing test quality, and producing structured evaluation reports.

> **⚠️ Prototype Notice:** All skills in this category are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## Recommended Workflow

The skills in this category follow a natural pipeline. Use them in this order:

```
            ┌─────────────────────────┐
            │  0. PRE-RUN ANALYSIS    │
            │                         │
            │  pre-run-analysis       │
            └───────────┬─────────────┘
                        │
            ┌───────────┴───────────────────┐
            ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  1. COLLECT ARTIFACTS   │     │  1. COLLECT ARTIFACTS   │
│                         │     │                         │
│  collect-test-          │     │  collect-test-          │
│  testingagent-logs      │     │  copilot-logs           │
└───────────┬─────────────┘     └───────────┬─────────────┘
            │                               │
            ▼                               ▼
┌─────────────────────────┐     ┌─────────────────────────┐
│  2. REVIEW INDIVIDUAL   │     │  2. REVIEW INDIVIDUAL   │
│                         │     │                         │
│  testing-agent-review   │     │  copilot-test-review    │
└───────────┬─────────────┘     └───────────┬─────────────┘
            │                               │
            └───────────┬───────────────────┘
                        ▼
            ┌─────────────────────────┐
            │  3. COMPARE SIDE-BY-SIDE│
            │                         │
            │  unit-test-comparison    │
            └───────────┬─────────────┘
                        │
                        ▼
            ┌─────────────────────────┐
            │  4. EXTRACT & PRIORITIZE│
            │                         │
            │  extract-issues          │
            └─────────────────────────┘
```

| Step | What to do | Skill |
|------|-----------|-------|
| **0. Analyze** | Before running any tool, analyze the target source for blockers, map the testable surface, and set a quality bar | `pre-run-analysis` |
| **1. Collect** | Run your test generation tool (Testing Agent or Copilot), then collect all artifacts into a timestamped folder | `collect-test-testingagent-logs` or `collect-test-copilot-logs` |
| **2. Review** | Analyze a single run — get a scored report with test quality assessment and suggested improvements | `testing-agent-review` or `copilot-test-review` |
| **3. Compare** _(optional)_ | If you ran both tools on the same source, compare them side-by-side with a unified scoring table | `unit-test-comparison` |
| **4. Extract** _(optional)_ | Pull all issues from review/comparison reports into two prioritized, deduplicated backlogs | `extract-issues` |

> **💡 Tip:** Step 0 can be run independently before any test generation. Steps 1 and 2 can be run independently. Step 3 requires artifacts from both tools targeting the same source file.

## Skills

### Pre-Run Analysis

| Skill | Description |
|-------|-------------|
| [`pre-run-analysis`](./pre-run-analysis/) | Analyze a .NET project before running any test generation tool — detects blockers (sealed classes, CPM, build failures), maps the testable surface, generates tailored prompts, and sets a quality bar. Works in VS Code, Visual Studio, and Copilot CLI. |

### Log Collection

| Skill | Description |
|-------|-------------|
| [`collect-test-copilot-logs`](./collect-test-copilot-logs/) | Collects logs, source files, generated test files from a .NET repo after a Copilot prompt — captures TRX results, console output, Cobertura code coverage, and Copilot diagnostic logs |
| [`collect-test-testingagent-logs`](./collect-test-testingagent-logs/) | Collects logs, source files, generated test files from a .NET repo after a Testing Agent prompt — captures TRX results, console output, Cobertura code coverage, Copilot diagnostic logs, and Testing Agent session logs |

### Review

| Skill | Description |
|-------|-------------|
| [`testing-agent-review`](./testing-agent-review/) | Analyze a single .NET Testing Agent run — classifies test quality, scores across five dimensions, and produces a detailed evaluation report |
| [`copilot-test-review`](./copilot-test-review/) | Analyze a single GH Copilot Agent Mode test generation run — reviews the log, assesses generated test quality, scores across five dimensions, and produces a detailed evaluation report |

### Comparison

| Skill | Description |
|-------|-------------|
| [`unit-test-comparison`](./unit-test-comparison/) | Run a structured case study review comparing the .NET Testing Agent and GH Copilot Agent Mode — produces individual evaluations plus a side-by-side comparison |

### Issue Extraction

| Skill | Description |
|-------|-------------|
| [`extract-issues`](./extract-issues/) | Extract, deduplicate, and prioritize issues from review and comparison reports into two sorted backlog markdowns |

## Skill Details

### [`pre-run-analysis`](./pre-run-analysis/)

**Analyze a .NET project before running test generation — detect blockers, map the testable surface, and set a quality bar.**

This skill is the "step 0" of the pipeline. Run it before launching either the Testing Agent or Copilot Agent Mode. It:

- **Detects blockers** — sealed classes, static dependencies, CPM conflicts, build failures
- **Flags warnings** — no test project, large file count, missing mock framework, high existing coverage
- **Maps the testable surface** — public methods, branches, edge cases, exception paths → a scenario table defining what "complete" looks like
- **Generates tailored prompts** — ready-to-use prompts for both tools, incorporating constraints and known gotchas
- **Sets a quality bar** — expected test count, behavioral %, negative assertions, coverage targets
- **Works everywhere** — VS Code, Visual Studio, and Copilot CLI (no environment-specific APIs)

#### Quick Start

```
Pre-run analysis for C:\path\to\MyService.cs
```

```
Analyze before testing C:\path\to\src\Services\
```

---

### [`collect-test-copilot-logs`](./collect-test-copilot-logs/)

**Capture GitHub Copilot and testing logs after a test run.**

This skill automates the collection of test artifacts and Copilot diagnostic logs from a .NET repository. It will:

- **Run all tests** in the repo with TRX logging and code coverage collection
- **Capture console output** — redirects all test output to a file
- **Collect Copilot logs** — automatically copies the most recent log from `%TEMP%\VSGitHubCopilotLogs`
- **Collect code coverage** — locates and copies the Cobertura XML report
- **Prompt for manual confirmation** — reminds the user to verify the Copilot log matches the current session

All artifacts are stored under `./artifacts/test-runs-copilot/<YYYYMMDD-HHMMSS>/`.

#### Quick Start

```
Collect Copilot and test logs for this repo
```

---

### [`collect-test-testingagent-logs`](./collect-test-testingagent-logs/)

**Capture .NET Testing Agent and testing logs after a test run.**

This skill automates the collection of test artifacts, Copilot diagnostic logs, and .NET Testing Agent logs. It will:

- **Run all tests** in the repo with TRX logging and code coverage collection
- **Capture console output** — redirects all test output to a file
- **Collect Copilot logs** — automatically copies the most recent log from `%TEMP%\VSGitHubCopilotLogs`
- **Collect Testing Agent logs** — copies the most recently created subfolder from `%TEMP%\VSCodeTestingAgentLogs`
- **Collect code coverage** — locates and copies the Cobertura XML report
- **Prompt for manual confirmation** — reminds the user to verify the Copilot log matches the current session

All artifacts are stored under `./artifacts/test-runs-testingagent/<YYYYMMDD-HHMMSS>/`.

#### Quick Start

```
Collect Testing Agent and test logs for this repo
```

---

### [`testing-agent-review`](./testing-agent-review/)

**Analyze and evaluate a .NET Code Testing Agent run.**

This skill reviews the output of a .NET Testing Agent run and produces a detailed markdown evaluation report. It:

- **Reconstructs the run timeline** — extracts key events, durations, and milestones from the agent log
- **Analyzes compilation errors** — catalogs C# error codes and explains root causes
- **Tracks deleted tests** — identifies tests the agent generated but later removed during fix iterations
- **Classifies test quality** — categorizes every test method as Behavioral, Trivial, or Redundant
- **Scores the run** — hybrid rubric + LLM-adjusted score (0–100) across five dimensions
- **Suggests actionable issues** — up to 10 concrete, ROI-prioritized improvements

#### Quick Start

```
Review the testing agent run in C:\path\to\TestingAgentFolder, source file is C:\path\to\MyClass.cs
```

---

### [`copilot-test-review`](./copilot-test-review/)

**Analyze and evaluate a GH Copilot Agent Mode test generation run.**

This skill reviews the output of a Copilot Agent Mode test generation run and produces a detailed markdown evaluation report. It:

- **Reconstructs the run timeline** — extracts key events from the Copilot log
- **Analyzes errors & warnings** — catalogs any errors, auth issues, or LSP failures
- **Classifies test quality** — categorizes every test method as Behavioral, Trivial, or Redundant
- **Scores the run** — hybrid rubric + LLM-adjusted score (0–100) across five dimensions
- **Suggests actionable issues** — up to 10 concrete, ROI-prioritized improvements

#### Quick Start

```
Review the copilot test run in C:\path\to\CopilotFolder, source file is C:\path\to\MyClass.cs
```

---

### [`unit-test-comparison`](./unit-test-comparison/)

**Compare .NET Testing Agent and GH Copilot Agent Mode test generation runs side-by-side.**

This skill runs a structured case study review comparing both tools. You provide artifacts from both tool runs and it produces three reports: one for each tool's run plus a side-by-side comparison.

#### Quick Start

```
Compare testing agent run in C:\path\to\TestingAgentFolder against copilot run in C:\path\to\CopilotFolder
```

---

### [`extract-issues`](./extract-issues/)

**Extract, deduplicate, and prioritize issues from review reports into actionable backlogs.**

This skill reads all review and comparison reports, extracts every suggested issue, deduplicates by theme, and produces two prioritized backlog markdowns — one for the Testing Agent team and one for the Copilot Agent team.

Key features:
- **Deduplication** — groups issues that describe the same root problem across multiple reports
- **Recurrence tracking** — issues seen across more runs rank higher within the same ROI tier
- **Versioning** — archives previous backlogs to `history\` before overwriting
- **Re-sortable** — every time you run it, issues are re-sorted by ROI priority and recurrence

#### Quick Start

```
Extract issues from C:\path\to\Evaluations
```
