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

A timestamped folder under `./artifacts/test-runs-testingagent/<YYYYMMDD-HHMMSS>/` containing:

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

Before starting collection, ask the user to confirm the following metadata and record it in the output:

| Field | Value |
|-------|-------|
| **Tool** | .NET Testing Agent |
| **Prompt** | _(record the exact prompt passed to the Testing Agent, e.g., "Generate unit tests for MyClass.cs")_ |
| **Target** | _(file, folder, project, or solution targeted)_ |
| **Date/Time** | _(date and time of the run)_ |
| **Visual Studio Version** | _(auto-extract from Copilot log — see Step 4)_ |
| **Copilot Chat Version** | _(auto-extract from Copilot log — see Step 4)_ |

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
$outDir = Join-Path $repoRoot "artifacts/test-runs-testingagent/$ts"
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

**Manual confirmation (required):** After automatic collection, remind the user to confirm the captured log corresponds to this session. If additional context is needed:
1. Open the Copilot Output or Copilot Chat window in Visual Studio
2. Copy relevant content
3. Save as `copilot-output.txt` in the artifacts folder

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

Ask the user which `.cs` test file(s) were generated during this Testing Agent run. Copy each file into the artifacts folder, preserving the original file name.

```powershell
# Ask the user for paths to generated test files
# For each file provided:
# Copy-Item $testFilePath (Join-Path $outDir (Split-Path $testFilePath -Leaf)) -Force
```

If the user cannot identify the generated files, note "Generated test files not collected — user could not identify them" and continue.

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
| **Visual Studio Version** | $vsVersion |
| **Copilot Chat Version** | $copilotChatVersion |
| **Model** | $model |
"@
$metadata | Out-File (Join-Path $outDir "run-metadata.md") -Encoding utf8
```

Substitute `$prompt`, `$target`, `$vsVersion`, `$copilotChatVersion`, and `$model` with the values extracted from the Copilot log in Step 4. Do not ask the user for VS version or model.

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
