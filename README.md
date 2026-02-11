# .NET Unit Testing Skills

A collection of prototype [Copilot Skills](https://docs.github.com/en/copilot/copilot-extensions/copilot-skills) for reviewing and comparing unit test output from the .NET Testing Agent and GH Copilot Agent Mode. These skills extend GitHub Copilot CLI with capabilities for collecting test artifacts, analyzing test quality, and producing structured evaluation reports.

> **⚠️ Prototype Notice:** All skills in this repo are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## What Are Skills?

Skills are reusable prompt-driven modules that teach Copilot how to perform specialized tasks. Each skill lives in its own folder and contains a `SKILL.md` file that defines:

- **Trigger phrases** — natural-language prompts that activate the skill
- **Inputs** — what the skill needs from the user (files, folders, parameters)
- **Outputs** — what the skill produces (reports, files, analysis)
- **Detailed instructions** — step-by-step logic Copilot follows to execute the task

## Table of Contents

### Unit Test Log Collection Skills

| Skill | Description |
|-------|-------------|
| [`collect-test-copilot-logs`](#collect-test-copilot-logs) | Collects logs, source files, generated test files from a .NET repo, file, folder, project or solution from a Copilot prompt like: "Write unit tests for X" — captures TRX results, console output, Cobertura code coverage, and Copilot diagnostic logs from `%TEMP%\VSGitHubCopilotLogs` — all stored under a timestamped `test-runs-copilot/` artifacts folder |
| [`collect-test-testingagent-logs`](#collect-test-testingagent-logs) | Collects logs, source files, generated test files from a .NET repo, file, folder, project or solution from a .NET Testing Agent prompt like: "@Test Write unit tests for X" — captures TRX results, console output, Cobertura code coverage, Copilot diagnostic logs, and .NET Testing Agent session logs from `%TEMP%\VSCodeTestingAgentLogs` — all stored under a timestamped `test-runs-testingagent/` artifacts folder |

### Unit Test Review Skills

| Skill | Description |
|-------|-------------|
| [`testing-agent-review`](#testing-agent-review) | Analyze a single .NET Testing Agent run — classifies test quality, scores across five dimensions (Correctness, Coverage Impact, Behavioral Depth, Test Design Quality, Stability & Reliability), and produces a detailed evaluation report |
| [`copilot-test-review`](#copilot-test-review) | Analyze a single GH Copilot Agent Mode test generation run — reviews the log, assesses generated test quality, scores across five dimensions, and produces a detailed evaluation report with ROI-prioritized suggested issues |

### Unit Test Comparison Skills

| Skill | Description |
|-------|-------------|
| [`unit-test-comparison`](#unit-test-comparison) | Run a structured case study review comparing the .NET Testing Agent and GH Copilot Agent Mode for unit test generation — produces individual evaluation reports for each run plus a side-by-side comparison with scoring, test quality assessment, and suggested issues |

## Skills

### [`collect-test-copilot-logs`](./collect-test-copilot-logs/)

**Capture GitHub Copilot and testing logs after a test run.**

This skill automates the collection of test artifacts and Copilot diagnostic logs from a .NET repository. Place the instructions file at the repo root and it will:

- **Run all tests** in the repo (solution-level if a `.sln` exists) with TRX logging and code coverage collection
- **Capture console output** — redirects all test output to a file
- **Collect Copilot logs** — automatically copies the most recent log from `%TEMP%\VSGitHubCopilotLogs`
- **Collect code coverage** — locates and copies the Cobertura XML report
- **Prompt for manual confirmation** — reminds the user to verify the Copilot log matches the current session

All artifacts are stored under `./artifacts/test-runs-copilot/<YYYYMMDD-HHMMSS>/`.

#### Quick Start

Copy `collect-copilot-logs-instructions.md` to the root of your .NET repository and follow the steps inside, or invoke via Copilot:

```
Collect Copilot and test logs for this repo
```

---

### [`collect-test-testingagent-logs`](./collect-test-testingagent-logs/)

**Capture .NET Testing Agent and testing logs after a test run.**

This skill automates the collection of test artifacts, Copilot diagnostic logs, and .NET Testing Agent logs from a .NET repository. Place the instructions file at the repo root and it will:

- **Run all tests** in the repo (solution-level if a `.sln` exists) with TRX logging and code coverage collection
- **Capture console output** — redirects all test output to a file
- **Collect Copilot logs** — automatically copies the most recent log from `%TEMP%\VSGitHubCopilotLogs`
- **Collect Testing Agent logs** — copies the most recently created subfolder from `%TEMP%\VSCodeTestingAgentLogs` into a `testingagent-logs/` folder
- **Collect code coverage** — locates and copies the Cobertura XML report
- **Prompt for manual confirmation** — reminds the user to verify the Copilot log matches the current session

All artifacts are stored under `./artifacts/test-runs-testingagent/<YYYYMMDD-HHMMSS>/`.

#### Quick Start

Copy `collect-testingagent-logs-instructions.md` to the root of your .NET repository and follow the steps inside, or invoke via Copilot:

```
Collect Testing Agent and test logs for this repo
```

---

### [`testing-agent-review`](./testing-agent-review/)

**Analyze and evaluate a .NET Code Testing Agent run.**

This skill reviews the output of a [.NET Testing Agent](https://learn.microsoft.com/en-us/visualstudio/test/unit-testing-with-github-copilot-test-dotnet?view=visualstudio) run and produces a detailed markdown evaluation report. You provide a folder containing the agent's log file and generated test files, along with the original source file or folder of source code that was targeted for test generation. The skill then:

- **Reconstructs the run timeline** — extracts key events, durations, and milestones from the agent log
- **Analyzes compilation errors** — catalogs C# error codes and explains root causes
- **Tracks deleted tests** — identifies tests the agent generated but later removed during fix iterations, and assesses whether they could have been salvaged
- **Classifies test quality** — categorizes every generated test method as Behavioral, Trivial, or Redundant by reading the test code against the original source file
- **Scores the run** — computes a hybrid rubric + LLM-adjusted score (0–100) across five dimensions: Correctness, Coverage Impact, Behavioral Depth, Test Design Quality, and Stability & Reliability
- **Suggests actionable issues** — proposes up to 10 concrete, ROI-prioritized improvements that can be logged as issues against the Testing Agent to help improve its behavior over time

#### Quick Start

```
Review the testing agent run in C:\path\to\TestingAgentFolder, source file is C:\path\to\MyClass.cs
```

or with a folder of source files:

```
Review the testing agent run in C:\path\to\TestingAgentFolder, source is C:\path\to\src\MyProject
```

The report is saved as a timestamped markdown file (e.g., `02062026-TestAgentReview-Run1/02062026-TestingAgent-eShop.md`).

---

### [`copilot-test-review`](./copilot-test-review/)

**Analyze and evaluate a GH Copilot Agent Mode test generation run.**

This skill reviews the output of a GH Copilot Agent Mode test generation run and produces a detailed markdown evaluation report. You provide a folder containing the Copilot log file and generated test files, along with the original source file that was targeted for test generation. The skill then:

- **Reconstructs the run timeline** — extracts key events from the Copilot log (semantic search rounds, file edits, plan updates)
- **Analyzes errors & warnings** — catalogs any errors, auth issues, or LSP failures from the log
- **Classifies test quality** — categorizes every generated test method as Behavioral, Trivial, or Redundant by reading the test code against the original source file
- **Scores the run** — computes a hybrid rubric + LLM-adjusted score (0–100) across five dimensions: Correctness, Coverage Impact, Behavioral Depth, Test Design Quality, and Stability & Reliability
- **Suggests actionable issues** — proposes up to 10 concrete, ROI-prioritized improvements for the Copilot-assisted test generation workflow

#### Quick Start

```
Review the copilot test run in C:\path\to\CopilotFolder, source file is C:\path\to\MyClass.cs
```

The report is saved as a timestamped markdown file (e.g., `02062026-CopilotTestReview-Run1/02062026-CopilotAgent-eShop.md`).

---

### [`unit-test-comparison`](./unit-test-comparison/)

**Compare .NET Testing Agent and GH Copilot Agent Mode test generation runs side-by-side.**

This skill runs a structured case study review comparing the .NET Testing Agent and GH Copilot Agent Mode for unit test generation. You provide artifacts from both tool runs and it produces three reports: one for each tool's run plus a side-by-side comparison with scoring, test quality assessment, and suggested issues.

#### Quick Start

```
Compare testing agent run in C:\path\to\TestingAgentFolder against copilot run in C:\path\to\CopilotFolder
```

## Using These Skills

### GitHub Copilot CLI

The Copilot CLI has built-in skill management via the `/skills` slash command.

1. **Add a skill from a local folder:**

   ```
   /skills add <path-to-skill-folder>
   ```

   For example, to install `testing-agent-review` after cloning this repo:

   ```
   /skills add C:\path\to\skills\testing-agent-review
   ```

   This copies the `SKILL.md` into `~/.copilot/skills/<skill-name>/`.

2. **List installed skills:**

   ```
   /skills list
   ```

3. **Get info about an installed skill:**

   ```
   /skills info testing-agent-review
   ```

4. **Remove a skill:**

   ```
   /skills remove testing-agent-review
   ```

Once installed, simply use one of the skill's trigger phrases in your conversation and Copilot will automatically activate it.

### VS Code (GitHub Copilot Chat)

You can use a skill in VS Code by placing the `SKILL.md` content into one of the custom instruction locations that Copilot Chat reads automatically:

1. **As a prompt file** (recommended for on-demand use):
   - Copy the `SKILL.md` file into your repo at `.github/prompts/<skill-name>.prompt.md`
   - In Copilot Chat, reference it with `#prompt:<skill-name>` or click the **+** button to attach it

2. **As an instructions file** (for automatic activation):
   - Copy the `SKILL.md` file into your repo at `.github/instructions/<skill-name>.instructions.md`
   - Copilot will automatically include the instructions when the context matches

> **Note:** VS Code uses the `.github/copilot-instructions.md` file and the `.github/instructions/` folder for custom instructions. Make sure the **"Enable custom instructions"** setting is turned on in VS Code under **Settings → GitHub Copilot → Chat**.

## Adding a New Skill

1. Create a new folder under the repo root (e.g., `my-new-skill/`)
2. Add a `SKILL.md` file following the format used by existing skills
3. Define trigger phrases, inputs, outputs, and detailed instructions
4. Update this README with a summary of the new skill
