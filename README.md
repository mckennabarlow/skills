# .NET Copilot Skills

A collection of prototype [Copilot Skills](https://docs.github.com/en/copilot/copilot-extensions/copilot-skills) for .NET development workflows. These skills extend GitHub Copilot CLI with domain-specific capabilities tailored to common .NET tasks and capabilities.

> **⚠️ Prototype Notice:** All skills in this repo are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## What Are Skills?

Skills are reusable prompt-driven modules that teach Copilot how to perform specialized tasks. Each skill lives in its own folder and contains a `SKILL.md` file that defines:

- **Trigger phrases** — natural-language prompts that activate the skill
- **Inputs** — what the skill needs from the user (files, folders, parameters)
- **Outputs** — what the skill produces (reports, files, analysis)
- **Detailed instructions** — step-by-step logic Copilot follows to execute the task

## Skills

### [`test-agent-review`](./test-agent-review/)

**Analyze and evaluate a .NET Code Testing Agent run.**

This skill reviews the output of a [.NET Code Testing Agent](https://devblogs.microsoft.com/dotnet/introducing-code-testing-agent/) run and produces a detailed markdown evaluation report. Given a folder containing the agent's log file and generated test files, it:

- **Reconstructs the run timeline** — extracts key events, durations, and milestones from the agent log
- **Analyzes compilation errors** — catalogs C# error codes and explains root causes
- **Tracks deleted tests** — identifies tests the agent generated but later removed during fix iterations, and assesses whether they could have been salvaged
- **Classifies test quality** — categorizes every generated test method as Behavioral, Trivial, or Redundant by reading the test code against the original source file
- **Scores the run** — computes a hybrid rubric + LLM-adjusted score (0–100) across five dimensions: Correctness, Coverage Impact, Behavioral Depth, Test Design Quality, and Stability & Reliability
- **Suggests actionable issues** — proposes up to 10 concrete, ROI-prioritized improvements for the Testing Agent based on evidence from the run

#### Quick Start

```
Review the testing agent run in C:\path\to\TestingAgentFolder, source file is C:\path\to\MyClass.cs
```

The report is saved as a timestamped markdown file (e.g., `02062026-TestAgentReview-Run1/02062026-TestingAgent-eShop.md`).

## Adding a New Skill

1. Create a new folder under the repo root (e.g., `my-new-skill/`)
2. Add a `SKILL.md` file following the format used by existing skills
3. Define trigger phrases, inputs, outputs, and detailed instructions
4. Update this README with a summary of the new skill
