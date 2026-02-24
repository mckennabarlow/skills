---
name: run-diagnosis
description: >
  Quickly diagnose a .NET test generation run — either from the .NET Testing Agent or GH Copilot
  Agent Mode. Locates log files, reads key events, and produces a short summary explaining what
  happened, why the run succeeded or failed, and any blocking issues found.
  Use this skill when asked to diagnose a test run, explain what happened, find out why a run
  failed, or get a quick summary of a test generation run.
  Trigger phrases include "diagnose run", "what happened", "why did it fail", "run summary",
  "quick look at logs", "explain this run", "what went wrong".
---

# Test Run Diagnosis Skill

## Speed Contract

This skill is optimized for speed. The goal is **answer in ≤ 3 tool-call rounds** for a typical run:
1. **Round 1** — locate artifacts + read log head/tail + read run-metadata.md (all in parallel)
2. **Round 2** — one targeted grep IF the head/tail didn't reveal the root cause
3. **Round 3** — write the diagnosis file and print summary

Do NOT do sequential per-field grep searches. Do NOT ask the user for metadata. Extract everything from the log.

---

## How to Use

```
What happened to this test run? C:\path\to\repo
```
```
Why did this run fail? C:\path\to\repo
```
```
Diagnose the testing agent run in C:\path\to\artifacts
```

---

## Rules

- **Read-only** — do not modify files, run `dotnet test`, or build anything.
- **No quality scoring** — that is for the review skills.
- **No advisory findings** — only report issues that directly caused or contributed to the run's outcome. Do not suggest improvements for future runs (e.g., missing custom instructions, project structure changes). That is the review skill's job.
- **No user prompts** — do not ask for prompt, VS version, or model. Extract from logs.
- **Under 30 lines** for the outcome + explanation section.
- **≤ 5 issues** — this is triage, not a full review.

---

## Workflow

### Step 1: Locate log file (single tool call)

Given the user's path, run ONE PowerShell command to find the log and detect tool type:

```powershell
$path = "<user-provided-path>"

# Read artifact root from saved preference
$artifactRootFile = Join-Path $env:USERPROFILE ".copilot\unittest-artifact-root.txt"
if (Test-Path $artifactRootFile) {
  $artifactRoot = (Get-Content $artifactRootFile -Raw).Trim()
} else {
  $artifactRoot = Join-Path $path "artifacts"
}

# If it's a repo root, find latest artifacts folder
$artDir = $null
foreach ($sub in @("testingagent", "copilot")) {
    $d = Get-ChildItem -Path (Join-Path $artifactRoot $sub) -Directory -ErrorAction SilentlyContinue |
        Sort-Object Name -Descending | Select-Object -First 1
    if ($d -and (!$artDir -or $d.Name -gt $artDir.Name)) { $artDir = $d }
}
if (!$artDir) { $artDir = Get-Item $path }  # assume direct artifacts folder

# Detect tool type
$taLog = Join-Path $artDir.FullName "testingagent-logs\codetestingagent.log"
$copilotLog = Join-Path $artDir.FullName "copilot-output.log"
if (Test-Path $taLog) { Write-Host "TOOL:TestingAgent"; Write-Host "LOG:$taLog" }
elseif (Test-Path $copilotLog) { Write-Host "TOOL:Copilot"; Write-Host "LOG:$copilotLog" }
else { Write-Host "TOOL:NotFound" }
Write-Host "ARTIFACTS:$($artDir.FullName)"

# Also check for run-metadata.md
$meta = Join-Path $artDir.FullName "run-metadata.md"
if (Test-Path $meta) { Write-Host "METADATA:$meta" }
```

If no log is found in artifacts, fall back to `%TEMP%\VSCodeTestingAgentLogs` (latest subfolder by CreationTime) and `%TEMP%\VSGitHubCopilotLogs` (latest file by LastWriteTime).

### Step 2: Read log + metadata in parallel

Make these tool calls **simultaneously in a single response**:

1. **Read first 80 lines** of the log file (contains version, prompt, target, model, scope, solution)
2. **Read last 80 lines** of the log file (contains how it ended — success, crash, cancel, errors)
3. **Read `run-metadata.md`** if it exists — this is the **primary source** for all identity metadata (Tool, Prompt, Target, Date/Time, Duration, Model, Agent Version, VS Version, Copilot Chat Version, Custom Instructions, Log file). If it contains all needed fields, skip step 4.
4. **Only if `run-metadata.md` is missing or incomplete:** Read first 350 lines of `copilot-output.log` if it exists — search for `Copilot chat version` to extract VS Version and Copilot Chat Version, and `PreferredModelFamily=` for the model. This is the fallback path only.

This gives you ~90% of what you need. The head has startup metadata; the tail has the outcome.

### Step 3: One targeted grep (only if needed)

If the head + tail didn't reveal the root cause, do ONE grep for the most likely missing piece:

| If you need... | Grep for |
|---------------|----------|
| Error details | `Exception:\|Error \|\|ServiceActivation\|Build: ` |
| Test results | `Test execution completed\|passed tests\|failed tests` |
| Cancellation | `Cancellation triggered\|cancel` |
| LLM call count | `Starting LLM call` (count matches) |
| Stall evidence | `get_type_info\|Duration:` |

Use a single grep with `\|` (OR) to find multiple patterns at once. **Do not do more than one grep.**

### Step 4: Write diagnosis and print immediately

Compose the diagnosis, save it to `run-diagnosis-output.md` in the artifacts folder, and print the full summary to the console. Do this in a single step.

---

## How to Extract Key Fields

### Testing Agent (`codetestingagent.log`)

From the **first 80 lines** (head):
- **Agent version** — first line: `.NET Code Testing Agent v...`
- **Start time** — first timestamp
- **Prompt** — after `Analyzing markdown content and mentions. Content:`
- **Target scope** — `scenarioType` in JSON response
- **Model** — `model:` after `Starting test generation with scope`. If not found, extract `PreferredModelFamily=` from `copilot-output.log` (within first 350 lines).
- **Custom instructions** — `Discovered \d+ custom instruction file` or `No custom instruction files found`
- **Solution** — `Found solution:`

From the **last 80 lines** (tail):
- **End time** — last timestamp
- **How it ended** — completion message, cancellation, or exception
- **Build result** — `Build: N succeeded, N failed`
- **Test results** — `Test execution completed` with Success/test count
- **Errors** — `Exception:`, `| Error |`, `ServiceActivationFailedException`

**Duration** = End time − Start time.

### Copilot Agent Mode (`copilot-output.log`)

From the **head**: session start, prompt (`Request content:`), early search terms, model (`PreferredModelFamily=` in config section).
From the **tail**: last file edits (`FileEditingState`), token totals (`TotalTokenCount`), how it ended.

---

## Run Outcome Classification

| Outcome | Condition |
|---------|-----------|
| ✅ **Success** | Tests generated, built, and passing |
| ⚠️ **Partial success** | Some tests generated but issues (build errors, some failing, tests deleted) |
| ❌ **Crashed** | Exception (e.g., `ServiceActivationFailedException`) terminated the run |
| ❌ **Cancelled** | `Cancellation triggered` found |
| ❌ **Stalled** | Duration > 15 min, < 5 tests produced |
| ❌ **No tests generated** | Run completed, 0 test files produced |
| ⚠️ **Unvalidated** | Tests generated but never built/executed (Copilot-specific) |

---

## Output Format

```markdown
# Run Diagnosis — <ProjectName> (<MM/DD/YYYY>)

## Run Metadata

| Field | Value |
|-------|-------|
| **Tool** | <.NET Testing Agent / GH Copilot Agent Mode> |
| **Prompt** | <extracted from log> |
| **Target** | <project/solution/file path> |
| **Start** | <timestamp> |
| **Duration** | <X min Y sec> |
| **Model** | <model name or "Unknown"> |
| **Agent Version** | <version string or "N/A"> |
| **VS Version** | <from run-metadata.md, or auto-extract from Copilot log> |
| **Copilot Chat Version** | <from run-metadata.md, or auto-extract from Copilot log, or "Unknown"> |
| **Custom Instructions** | <from run-metadata.md, or extract from log, or "None detected"> |
| **Log file** | <full path to the primary log file> |

## Outcome: <emoji> <outcome label>

<2-5 sentence plain-English explanation. State what the agent did, how far it got, and what stopped it.>

## Key Events

| Time | Event |
|------|-------|
| <HH:MM:SS> | <event> |

(5-15 most important events only.)

## Issues Found

(If clean run: "No issues detected." If problems found, use issue format below. Max 5 issues.)

### Issue N (ROI: <HIGH|MEDIUM|LOW>): <title>

- **Problem:** <what happened — 2-3 sentences describing the failure>

- **Evidence:**
  - **Exception/error:** <exact exception type and message from the log, quoted verbatim>
  - **Log location:** <timestamp and log line number(s) where the error appears>
  - **Stack trace path:** <the key frames showing the call chain, e.g., `CallerMethod` → `MiddleMethod` → `FailingMethod (file:line)`. Include the source file and line number if present in the stack trace.>
  - **Owning component:** <which component/assembly owns the failing code — e.g., `Microsoft.VisualStudio.CoverageServices (18.5.0.0)` or `Microsoft.Copilot.Testing.Core`. Identify whether it is a VS-internal component, an agent component, or user code.>
  - **Trigger condition:** <what the agent was doing when the error occurred — e.g., "during initial coverage collection after running 3 existing tests", "while searching for types across 24 projects">
  - **Transient or persistent:** <will retrying hit the same error? Explain why — e.g., "Persistent — hard assembly reference mismatch" or "Likely transient — service timeout under load">

- **Suggested fix:** <numbered list of concrete resolution paths, from most likely to workaround. For each, identify who would need to act (VS team, agent team, user).>

- **Fix location:** <`Testing Agent orchestrator` | `Copilot Agent orchestrator` | `VS ServiceHub` | `User workflow` | etc.>

- **Why ROI:** <1 sentence>
```

---

## Failure Pattern Quick-Reference

Use this to identify root causes — do NOT include this table in the output.

| Pattern | Log signal | Cause |
|---------|-----------|-------|
| VS service crash | `ServiceActivationFailedException` | VS internal service failure |
| Agent stall | Long timestamp gaps, `get_type_info` across many projects | Code-gen agent stuck in tool-calling loop |
| User cancelled | `Cancellation triggered for request` | Manual cancellation |
| Build failure | `Build: 0 succeeded, N failed` | Generated code doesn't compile |
| No testable members | `Found 0 testable members` | Nothing for agent to test |
| Missing packages | `NU1001`, `CS0246` | Test project missing NuGet references |
| DB connection | `Npgsql` exceptions | Functional tests needing PostgreSQL |
| Rate limit | `429` or `rate limit` | LLM API throttled |
| Context overflow | `token limit` or `context length` | Source too large for LLM |

---

## Style

- **Direct and concise** — this is triage, not a report
- **Lead with outcome** — user wants "what happened?" first
- **Plain English** — avoid jargon
- **Emoji** for visual scanning: ✅ ⚠️ ❌ 💥
- If it's simple (e.g., "VS crashed"), keep it short
- **Always use Unicode emoji** (❌, ✅, ⚠️, 💥) — never use shortcodes like `:x:` or `:boom:` as they don't render in all viewers
