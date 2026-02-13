# Instructions: Capture .NET Testing Agent and Testing Logs
Also includes: TRX + Console Output + Code Coverage (Repo Root) output

## Run Metadata

Auto-extract as much metadata as possible from the log files before asking the user. Only prompt the user for fields that could not be auto-detected.

| Field | Auto-extract | Fallback |
|-------|-------------|----------|
| **Tool** | .NET Testing Agent | _(hardcoded)_ |
| **Prompt** | Extract from `codetestingagent.log` — search for `Analyzing markdown content and mentions. Content:` and take text after `Content:`. If not found, search `copilot-output.log` for `Request content:` and take the quoted text. | Ask the user only if neither log contains the prompt. |
| **Target** | Extract from the prompt text (the `#file:` mention) or from log `Mapped source file` entries. | Ask the user if not found. |
| **Date/Time** | Extract from the first timestamp in `codetestingagent.log`. | Ask the user if not found. |
| **Visual Studio Version** | _(e.g., 17.14 Preview 3)_ | Ask the user if not found. |

Context:
- This file lives at the root of the repository.
- All commands must be executed relative to this directory.
- Only use files and folders located alongside this .md file or below it.

Goal:
- Run all tests in the repo
- Produce a TRX test results file
- Capture full test console output
- Produce a code coverage report (Cobertura XML)
- Store all artifacts under: ./artifacts/test-runs-testingagent/<timestamp>/

Rules:
- Do not modify product or test code.
- Do not assume a fixed solution or test project name.
- If one or more .sln files exist at repo root, prefer running at the solution level.
- Otherwise, run tests from repo root and allow discovery.
- All artifacts must be copied into the timestamped output folder.

Steps (PowerShell, run from repo root):

1. Treat the directory containing this file as the repo root.

2. Create a timestamped artifacts folder under:
  ./artifacts/test-runs-testingagent/<YYYYMMDD-HHMMSS>/

3. Run tests with:
   - TRX logging enabled
   - Code coverage collection enabled
   - All console output redirected to a file

4. Collect Copilot output logs (automatic + manual confirmation):

  Purpose:
  - Preserve GitHub Copilot interaction context for this run.
  - Keep Copilot logs alongside test, build, and coverage artifacts.

  Automatic Copilot log collection:
  - Copilot writes diagnostic logs to: %TEMP%\VSGitHubCopilotLogs
  - After test execution completes:
    1. Locate the Copilot logs directory: %TEMP%\VSGitHubCopilotLogs
    2. Identify the most recently modified file in that directory.
    3. Copy that file into the timestamped artifacts folder as:
      ./artifacts/test-runs-testingagent/<timestamp>/copilot-output.log
    4. If the directory does not exist or contains no files:
      - Print a message indicating no Copilot logs were found.
      - Continue without failing the workflow.

  Manual Copilot log confirmation (still required):
  - Before finishing, Copilot must remind the user to confirm whether the captured
    Copilot log corresponds to this test generation or execution session.
  - If additional Copilot chat or output context is relevant and not reflected in
    the log file:
    1. Open the Copilot Output or Copilot Chat window in Visual Studio.
    2. Copy any relevant content.
    3. Save it as:
      ./artifacts/test-runs-testingagent/<timestamp>/copilot-output.txt
    4. Confirm the file exists before completing the workflow.

  Important:
  - Copilot logs may include additional background activity beyond this run.
  - The manual confirmation step ensures the correct context is preserved.

5. Collect .NET Testing Agent logs (automatic):

  Purpose:
  - Preserve .NET Testing Agent diagnostic context for this run.
  - Keep Testing Agent logs alongside test, build, and coverage artifacts.

  Automatic Testing Agent log collection:
  - The .NET Testing Agent writes diagnostic logs to: %TEMP%\VSCodeTestingAgentLogs
  - Logs are organized into subfolders by date/session.
  - After test execution completes:
    1. Locate the Testing Agent logs directory: %TEMP%\VSCodeTestingAgentLogs
    2. Identify the most recently created subfolder in that directory.
    3. Copy the entire contents of that subfolder into the timestamped artifacts folder under:
      ./artifacts/test-runs-testingagent/<timestamp>/testingagent-logs/
    4. If the directory does not exist or contains no subfolders:
      - Print a message indicating no Testing Agent logs were found.
      - Continue without failing the workflow.

6. Locate the generated coverage.cobertura.xml file under the results directory
   (it may be nested under TestResults).

7. Copy coverage.cobertura.xml into the root of the timestamped folder.

8. Verify the following files exist:
   - test-results.trx
   - test-console.txt
   - coverage.cobertura.xml
  - copilot-output.log (if available)
  - copilot-output.txt (if manual capture was needed)
  - testingagent-logs/ (folder, if available)

Commands:

Create output folder:
$repoRoot = Get-Location
$ts = Get-Date -Format "yyyyMMdd-HHmmss"
$outDir = Join-Path $repoRoot "artifacts/test-runs-testingagent/$ts"
New-Item -ItemType Directory -Force -Path $outDir | Out-Null

Discover solution file (if any):
$sln = Get-ChildItem -Path $repoRoot -Filter *.sln -File | Select-Object -First 1

Run tests:
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

Collect Copilot output logs (automatic):
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

Collect .NET Testing Agent logs (automatic):
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

Locate and copy Cobertura coverage file:
$coverage = Get-ChildItem -Path $outDir -Recurse -Filter "coverage.cobertura.xml" | Select-Object -First 1
if ($coverage) {
  Copy-Item $coverage.FullName (Join-Path $outDir "coverage.cobertura.xml") -Force
} else {
  Write-Host "Coverage file not found under results directory."
}

Print summary:
Write-Host "Artifacts collected at: $outDir"
Get-ChildItem $outDir | Format-Table Name, Length
