---
name: extract-issues
description: >
  Extract, deduplicate, and prioritize issues from Testing Agent and Copilot review reports
  into two sorted backlog markdowns. Use this skill when asked to extract issues, build a backlog,
  consolidate findings, or pull bugs/improvements from review reports. Trigger phrases include
  "extract issues", "build backlog", "consolidate issues", "pull issues from reports",
  "create issue backlog", "prioritize issues".
---

# Extract Issues Skill

## How to Use

Invoke this skill by asking Copilot to extract issues from review reports. Provide:

1. **One or more folders** containing review report `.md` files (Testing Agent reviews, Copilot reviews, and/or comparison reports)
2. **Output directory** — where to save the backlog files (you will be asked if not provided)

### Example prompts

```
Extract issues from C:\path\to\Evaluations
```

```
Build a backlog from these reports: C:\path\to\folder1 C:\path\to\folder2
```

```
Consolidate issues from all review reports in C:\path\to\reports
```

### What you get

Two markdown backlog files, deduplicated and sorted by ROI priority:

| File | Description |
|------|-------------|
| `backlog-testing-agent.md` | All issues relevant to the .NET Testing Agent, sorted by ROI |
| `backlog-copilot-agent.md` | All issues relevant to GitHub Copilot Agent Mode, sorted by ROI |

### Versioning

If a backlog file already exists in the output directory:
1. The previous version is copied to a `history\` subfolder as `backlog-<tool>-YYYYMMDD-HHMMSS.md`
2. The canonical backlog file is then overwritten with the new consolidated version

This ensures you always have **one clean file to act on** plus **full history** of prior versions.

---

## Purpose

Given one or more review/comparison reports, extract all suggested issues, deduplicate them by theme, and produce two prioritized backlog markdowns — one for the Testing Agent team and one for the Copilot Agent team.

---

## Inputs

- **Report folders** — one or more folders containing `.md` review reports. The skill will recursively search for files matching these patterns:
  - `*-TestingAgent-*.md` (Testing Agent review reports)
  - `*-CopilotAgent-*.md` (Copilot review reports)
  - `*-UnitTestEvaluation-*.md` (Comparison reports)
- **Output directory** — present the default location to the user and ask them to confirm or pick a different folder. Remember their choice for future runs of this skill.
  - **Default:** the same folder as the report input folder(s)
  - **Confirm:** "I'll save the backlog files to `<default path>`. Is that OK, or would you prefer a different location?"
  - **Remember:** If the user picks a custom location, store it and use it as the new default for subsequent runs. If they confirm the default, continue using it.

---

## Process

### Step 1: Discover Reports

Recursively search the provided folders for `.md` files matching the report patterns above. List all discovered reports and confirm with the user before proceeding.

### Step 2: Extract Issues

For each report, find all `### Issue N` sections and extract:

| Field | Source |
|-------|--------|
| Issue number | `### Issue N (ROI: <tier>): <title>` |
| Title | After the colon in the issue header |
| ROI tier | `HIGH`, `MEDIUM`, or `LOW` from the header |
| Problem | Text after `**Problem:**` |
| Suggested fix | Text after `**Suggested fix:**` |
| Fix location | Text after `**Fix location:**` |
| LLM cost | Text after `**LLM cost:**` |
| Why ROI | Text after `**Why ROI:**` |
| Source report | The file name and path of the report |

### Step 2b: Exclude Non-Actionable Issues

Skip any extracted issue whose title or problem text is primarily about missing or not finding Copilot custom instructions files (e.g., `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md`, `.copilot/` instructions). These are environment-specific observations, not tool defects or improvements.

### Step 3: Classify Tool Owner

Assign each issue a `tool_owner` based on the fix location and source report:

| Fix location contains | Tool owner |
|----------------------|------------|
| `Testing Agent orchestrator` (without `Copilot`) | `testing-agent` |
| `Copilot Agent orchestrator` (without `Testing Agent`) | `copilot` |
| Both `Testing Agent` and `Copilot` | `shared` |
| `Roslyn analyzers` only | `shared` |
| `LLM prompt/model` only | `shared` |
| `VS test runner` only | Infer from source report type |
| `NuGet/MSBuild tooling` only | Infer from source report type |

Issues from **comparison reports** that mention both tools are classified as `shared`.

### Step 4: Deduplicate by Theme

Group issues that describe the same root problem across multiple reports. Two issues are considered duplicates if they share:

1. **The same core theme** (e.g., "sealed class mocking" appears in Run 1, Run 2, and Comparison)
2. **The same fix location category**

Assign each group a `dedup_group` label (e.g., `sealed-class-mocking`, `fix-loop-stall`, `cpm-incompatible`).

For each dedup group, select the **best version** of the issue text using this priority:
1. Comparison report version (broadest perspective)
2. Most recent individual report version
3. Earliest individual report version

### Step 5: Assign to Backlogs

- **Testing Agent backlog:** All issues where `tool_owner` is `testing-agent` or `shared`
- **Copilot Agent backlog:** All issues where `tool_owner` is `copilot` or `shared`

Shared issues appear in **both** backlogs.

### Step 6: Sort and Upvote

Within each backlog, sort by:
1. **ROI tier** — HIGH first, then MEDIUM, then LOW
2. **Recurrence count** (descending) — issues seen across more reports rank higher within the same tier

**Upvote count in header:** Include the recurrence count as `(🔺 N)` in each issue header, where N is the number of distinct tool runs that encountered this issue. Each new run that exhibits the same issue ticks up the count by 1. A comparison report echoing a finding from an individual review does not add a new count — only distinct runs do.

**Format:** `### Issue N (ROI: <tier>) (🔺 <count>): <Title>`

### Step 7: Archive Previous Backlogs

Before writing each backlog file:
1. Check if the file already exists in the output directory
2. If it exists, create the `history\` subfolder if needed
3. Copy the existing file to `history\backlog-<tool>-YYYYMMDD-HHMMSS.md` (using current timestamp)
4. Then overwrite the canonical file

### Step 8: Generate Backlog Markdowns

Write each backlog using the format below.

---

## Output Format

```markdown
# <Tool Name> — Issue Backlog

> **Auto-generated** from N reports on MM/DD/YYYY
> **Re-run with:** `extract issues from <folder paths>`

## Summary

- **Total unique issues:** X
- **HIGH ROI:** X | **MEDIUM ROI:** X | **LOW ROI:** X
- **Reports analyzed:**
  - <Report 1 label>
  - <Report 2 label>
  - ...

---

## Pre-Run Checklist

> **Run this before launching <tool>.** Many HIGH-ROI issues in this backlog could be prevented or mitigated by checking constraints upfront.

### 🔍 Analyze Constraints

<Generate a table of 4-6 pre-run checks derived from the HIGH and MEDIUM ROI issues in this backlog. Each check should:>
- Describe what to look for
- Link to the related issue number
- Be actionable (something the user can verify or fix before running)

Common constraint categories:
- **Sealed/static dependencies** — scan constructor parameters for unmockable types
- **Package management** — check for CPM (Directory.Packages.props)
- **Existing test health** — verify existing tests build and pass
- **Scope vs. time budget** — estimate run time based on file count
- **Existing coverage** — identify files that already have high coverage
- **Test discovery state** — ensure VS test runner cache is fresh
- **Mocking framework** — verify the test project has the expected mocking library

### 📋 Define the Test Plan

Before running, decide:
- **Which files to target** — specific files, ordered by priority
- **What must be covered** — critical code paths, error handling, branching logic
- **What patterns to use** — mocking framework, test framework, async patterns
- **What to avoid** — sealed classes, private methods, trivial tests

### 🎯 Define the Quality Bar

<Generate a table of 5-7 quality dimensions derived from the issues in this backlog. Each should have a minimum bar and how to check it. Common dimensions:>
- Build validation (tests compile)
- Test execution (tests pass)
- Behavioral % (≥70%)
- Async coverage (async methods get async tests)
- Negative assertions (at least some DidNotReceive/Times.Never)
- No trivial tests (constructor-not-null, constant assertions)
- All files addressed (every target file has tests or a skip reason)

---

## HIGH ROI

### Issue 1 (ROI: HIGH) (🔺 3): <Title>

- **Problem:** <Merged description citing specific evidence>
- **Fix location:** <Location>
- **Suggested fix:** <Best merged fix suggestion>
- **LLM cost:** <Merged LLM cost from source issues — aggregate or summarize across runs, e.g., "~8 of 15 LLM calls (53%) in Run 1; ~5 of 12 (42%) in Run 2". If all sources report "N/A", use "N/A". If mixed, include available data.>
- **Why ROI:** <Reason for the tier>
- **Observed in:** <List of reports> (N of M reports)
- **Recurrence:** <How many runs exhibited this>

### Issue 2 (ROI: HIGH) (🔺 1): ...

---

## MEDIUM ROI

### Issue N (ROI: MEDIUM): ...

---

## LOW ROI

### Issue N (ROI: LOW): ...
```

### Field Details

| Field | Description |
|-------|-------------|
| **Problem** | Use the best version from deduplication. If multiple reports describe the same issue with different details, merge the most informative parts. |
| **Observed in** | List each report where this issue (or a duplicate) appeared, plus count (e.g., "3 of 4 reports") |
| **Recurrence** | How many actual tool runs exhibited this issue (e.g., "2 of 2 Testing Agent runs"). This is different from report count — a comparison report echoing a finding doesn't add a new run occurrence. |
| **Fix location** | Use the most complete fix location from all duplicates |
| **Suggested fix** | Use the most actionable fix. If multiple reports suggest slightly different fixes, merge the best parts. |
| **LLM cost** | Merge LLM cost data from all duplicate instances. Aggregate across runs (e.g., "~8 of 15 calls in Run 1; ~5 of 12 in Run 2"). If all sources report "N/A", use "N/A". |
| **Why ROI** | Use the most compelling justification from all duplicates |

---

## Style Guidelines

- Use **bullet points** with `- **Field:**` format for all issue fields
- Use **bold** for key metrics (test counts, error counts, durations)
- Use **code formatting** for class names, method names, error codes, and CLI commands
- Keep each field concise — 1-3 sentences max
- Include the `---` horizontal rule between ROI tier sections
- Number issues sequentially across all tiers (don't restart at 1 for each tier)

---

## After Completion

Report to the user:
1. How many reports were analyzed
2. How many total issues were extracted
3. How many unique issues remain after deduplication
4. How many issues in each backlog
5. Whether any previous backlogs were archived to `history\`
6. The full paths to both backlog files
