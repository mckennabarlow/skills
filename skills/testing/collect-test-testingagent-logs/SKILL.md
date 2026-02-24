---
name: collect-test-testingagent-logs
description: >
  Collect test artifacts, GitHub Copilot logs, and .NET Testing Agent logs from a .NET repository
  after a Testing Agent test generation run. Captures TRX results, console output, Cobertura code
  coverage, Copilot diagnostic logs, and Testing Agent session logs into a timestamped artifacts folder.
  Use this skill when asked to collect testing agent logs, gather testing agent artifacts, or capture
  testing agent run output.
  Trigger phrases include "collect testing agent logs", "gather testing agent artifacts",
  "capture testing agent output", "collect testing agent run", "save testing agent logs".
---

# Collect .NET Testing Agent Test Logs Skill

## How to Use

Invoke this skill by asking Copilot to collect test artifacts from a .NET Testing Agent test generation run. The skill should be run from the root of the .NET repository that was used for test generation.

### Example prompts

```
Collect Testing Agent and test logs for this repo
```

```
Gather testing agent artifacts from C:\path\to\repo
```

```
Capture testing agent run output
```

### What you get

A folder at `<run_root>/testingagent/` (when invoked by the pipeline) or `<artifact_root>/testingagent/<YYYYMMDD-HHMMSS>/` (when run standalone) containing:

| File | Description |
|------|-------------|
| `test-results.trx` | TRX test results file |
| `test-console.txt` | Full console output from the test run |
| `coverage.cobertura.xml` | Cobertura XML code coverage report |
| `copilot-output.log` | Most recent Copilot diagnostic log (automatic) |
| `copilot-output.txt` | Manual Copilot output capture (if needed) |
| `testingagent-logs/` | Full .NET Testing Agent session logs folder |
| `run-metadata.md` | Run metadata (tool, prompt, target, date, VS version) |
| `*.Tests.cs` | Generated test file(s) copied from the repo |

---

## Purpose

Automate the collection of test artifacts, GitHub Copilot diagnostic logs, and .NET Testing Agent session logs from a .NET repository after a test generation run. All artifacts are stored in a single timestamped folder for later review using the `testing-agent-review` skill.

---

## Inputs

- **Repository root** — the directory containing the `.sln` file (or the repo root if no `.sln` exists). Treat the directory containing this skill file or the user's current working directory as the repo root.

---

## Run Metadata

Auto-extract as much metadata as possible from the log files before asking the user. Only prompt the user for fields that could not be auto-detected.

| Field | Auto-extract | Fallback |
|-------|-------------|----------|
| **Tool** | .NET Testing Agent | _(hardcoded)_ |
| **Prompt** | Extract from `codetestingagent.log` — search for `Analyzing markdown content and mentions. Content:` and take text after `Content:`. If not found, search `copilot-output.log` for `Request content:` and take the quoted text. | Ask the user only if neither log contains the prompt. |
| **Target** | Extract from the prompt text (the `#file:` mention) or from `codetestingagent.log` `Mapped source file` entries. | Ask the user if not found. |
| **Date/Time** | Extract from the first timestamp in `codetestingagent.log`. | Ask the user if not found. |
| **Visual Studio Version** | _(auto-extract from Copilot log — see Step 4)_ | Ask the user if not found. |
| **Copilot Chat Version** | _(auto-extract from Copilot log — see Step 4)_ | Note "Unknown" if not found. |

---

## Rules

- Do not modify product or test code.
- Do not assume a fixed solution or test project name.
- If one or more `.sln` files exist at repo root, prefer running at the solution level.
- Otherwise, run tests from repo root and allow discovery.
- All artifacts must be copied into the timestamped output folder.

---

## Workflow

Execute the following steps in order using PowerShell from the repo root.

### Step 1: Create output folder

```powershell
$repoRoot = Get-Location
$ts = Get-Date -Format "yyyyMMdd-HHmmss"
# If run_root is provided by the pipeline, write directly to run_root/testingagent/
# Otherwise, fall back to artifact_root/testingagent/<timestamp>/ for standalone use
if ($run_root) {
  $outDir = Join-Path $run_root "testingagent"
} else {
  $artifactRootFile = Join-Path $env:USERPROFILE ".copilot\unittest-artifact-root.txt"
  if (Test-Path $artifactRootFile) {
    $artifactRoot = (Get-Content $artifactRootFile -Raw).Trim()
  } else {
    $artifactRoot = Join-Path $repoRoot "artifacts"
  }
  $outDir = Join-Path $artifactRoot "testingagent/$ts"
}
New-Item -ItemType Directory -Force -Path $outDir | Out-Null
```

### Step 2: Discover solution file

```powershell
$sln = Get-ChildItem -Path $repoRoot -Filter *.sln -File | Select-Object -First 1
```

### Step 3: Run tests

Run with TRX logging, code coverage collection, and console output redirected to a file.

```powershell
if ($sln) {
  dotnet test $sln.FullName `
    --results-directory $outDir `
    --logger "trx;LogFileName=test-results.trx" `
    --collect "XPlat Code Coverage" `
    *> (Join-Path $outDir "test-console.txt")
} else {
  dotnet test `
    --results-directory $outDir `
    --logger "trx;LogFileName=test-results.trx" `
    --collect "XPlat Code Coverage" `
    *> (Join-Path $outDir "test-console.txt")
}
```

### Step 4: Collect Copilot diagnostic logs

Copilot writes diagnostic logs to `%TEMP%\VSGitHubCopilotLogs`. Automatically copy the most recently modified file.

```powershell
$copilotLogDir = Join-Path $env:TEMP "VSGitHubCopilotLogs"
if (Test-Path $copilotLogDir) {
  $latestCopilotLog = Get-ChildItem $copilotLogDir -File |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

  if ($latestCopilotLog) {
    Copy-Item $latestCopilotLog.FullName `
      (Join-Path $outDir "copilot-output.log") `
      -Force
  } else {
    Write-Host "Copilot log directory found, but no log files present."
  }
} else {
  Write-Host "Copilot log directory not found at %TEMP%\VSGitHubCopilotLogs."
}
```

**Auto-validate Copilot log:** After copying the log, verify it corresponds to this session by checking whether the log contains the current repo path (e.g., the repo root or solution path). If found, the log is valid — proceed without asking the user. If the repo path is not found in the log, warn the user that the captured log may not correspond to this session and ask them to confirm or provide a manual capture as `copilot-output.txt`.

**Auto-extract VS, Copilot, and model versions:** After copying the Copilot log, extract the VS version, Copilot Chat version, and LLM model from the log automatically. Search the first 350 lines for:

1. **`Copilot chat version`** — the format is:

```
Copilot chat version <CopilotVersion>. VS: <VSVersion>. Session: <id>
```

Example: `Copilot chat version 18.5.38402+d52add363e (18.5.38402.54570). VS: VisualStudio.18.int.main/18.5.0-insiders+11513.45.main`

Extract and store:
- **VS Version** → the value after `VS:` (e.g., `VisualStudio.18.int.main/18.5.0-insiders+11513.45.main`)
- **Copilot Chat Version** → the value after `Copilot chat version` (e.g., `18.5.38402+d52add363e`)

2. **`PreferredModelFamily=`** — contains the LLM model used (e.g., `claude-opus-4.6`). Extract and store as **Model**.

Use these values in the Run Metadata (Step 8). Do not ask the user for VS version or model — always auto-extract from the Copilot log. If a line is not found, note "Unknown — not found in Copilot log".

### Step 5: Collect .NET Testing Agent logs

The Testing Agent writes session logs to `%TEMP%\VSCodeTestingAgentLogs` in subfolders organized by date/session. Copy the most recently **created** subfolder (use `CreationTime`, not `LastWriteTime`).

```powershell
$testingAgentLogDir = Join-Path $env:TEMP "VSCodeTestingAgentLogs"
if (Test-Path $testingAgentLogDir) {
  $latestAgentFolder = Get-ChildItem $testingAgentLogDir -Directory |
    Sort-Object CreationTime -Descending |
    Select-Object -First 1

  if ($latestAgentFolder) {
    $agentLogDest = Join-Path $outDir "testingagent-logs"
    Copy-Item $latestAgentFolder.FullName $agentLogDest -Recurse -Force
  } else {
    Write-Host "Testing Agent log directory found, but no subfolders present."
  }
} else {
  Write-Host "Testing Agent log directory not found at %TEMP%\VSCodeTestingAgentLogs."
}
```

### Step 6: Collect code coverage

```powershell
$coverage = Get-ChildItem -Path $outDir -Recurse -Filter "coverage.cobertura.xml" | Select-Object -First 1
if ($coverage) {
  Copy-Item $coverage.FullName (Join-Path $outDir "coverage.cobertura.xml") -Force
} else {
  Write-Host "Coverage file not found under results directory."
}
```

### Step 7: Collect generated test files

Auto-detect generated test files instead of asking the user. Use git to find recently modified or added `*Tests*.cs` files:

```powershell
# Find test files modified or added in the working tree (unstaged + staged)
$testFiles = git diff --name-only HEAD -- '*.cs' 2>$null |
  Where-Object { $_ -match 'Tests?\.cs$|Tests?/' }
$testFiles += git diff --cached --name-only -- '*.cs' 2>$null |
  Where-Object { $_ -match 'Tests?\.cs$|Tests?/' }
$testFiles = $testFiles | Select-Object -Unique

if ($testFiles) {
  foreach ($f in $testFiles) {
    $fullPath = Join-Path $repoRoot $f
    if (Test-Path $fullPath) {
      Copy-Item $fullPath (Join-Path $outDir (Split-Path $fullPath -Leaf)) -Force
      Write-Host "Collected: $f"
    }
  }
} else {
  Write-Host "No recently modified test files detected via git."
}
```

If no files are found via git (e.g., changes already committed), ask the user as a fallback.

### Step 8: Save Run Metadata

Write the Run Metadata to a file so the `testing-agent-review` skill can pick it up automatically.

```powershell
$metadata = @"
# Run Metadata

| Field | Value |
|-------|-------|
| **Tool** | .NET Testing Agent |
| **Prompt** | $prompt |
| **Target** | $target |
| **Date/Time** | $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') |
| **Duration** | $duration |
| **Model** | $model |
| **Agent Version** | $agentVersion |
| **Visual Studio Version** | $vsVersion |
| **Copilot Chat Version** | $copilotChatVersion |
| **Custom Instructions** | $customInstructions |
| **Log file** | $logFilePath |
"@
$metadata | Out-File (Join-Path $outDir "run-metadata.md") -Encoding utf8
```

Substitute all variables with values extracted from the logs in Steps 4 and 5:
- `$prompt`, `$target` — from the user or log
- `$vsVersion`, `$copilotChatVersion`, `$model` — auto-extracted from the Copilot log (see Step 4)
- `$agentVersion` — extract from the first line of the Testing Agent log (e.g., `.NET Code Testing Agent v0.4.971-alpha+d3e2de585c`)
- `$duration` — calculate from first and last timestamps in the Testing Agent log, format as `X min Y sec`
- `$customInstructions` — check whether a custom instructions file was detected in the log. Look for references to `.github/copilot-instructions.md` or `.github/instructions/*.instructions.md`. Note the file path(s) if found, or "None detected"
- `$logFilePath` — full path to the Testing Agent log file copied in Step 5

### Step 9: Verify and summarize

```powershell
Write-Host "Artifacts collected at: $outDir"
Get-ChildItem $outDir | Format-Table Name, Length
```

Verify these files exist:
- `test-results.trx`
- `test-console.txt`
- `coverage.cobertura.xml`
- `copilot-output.log` (if available)
- `copilot-output.txt` (if manual capture was needed)
- `testingagent-logs/` (folder, if available)
- `run-metadata.md`
- Generated `.cs` test file(s) (if collected)

---

## Important Notes

- Copilot logs may include background activity beyond this run — the manual confirmation step ensures the correct context is preserved.
- Testing Agent logs use `CreationTime` (not `LastWriteTime`) to identify the most recent session folder. Each subfolder contains `codetestingagent.log` and an `Agents/` directory with sub-agent logs.
- The artifacts folder is designed to be used as input for the `testing-agent-review` skill.
