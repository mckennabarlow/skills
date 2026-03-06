---
name: code-coverage-analysis
description: >
  Generate a comprehensive code coverage report with actionable insights for a .NET project or
  solution. Runs tests with coverage collection (or analyzes existing Cobertura XML), produces an
  HTML visual report via ReportGenerator, and generates a markdown insights report with metrics,
  risk analysis, and prioritized recommendations for where to improve test coverage.
  Use this skill when asked to analyze code coverage, generate a coverage report, find coverage
  gaps, or get coverage insights for a .NET project.
  Trigger phrases include "code coverage analysis", "coverage report", "analyze coverage",
  "coverage insights", "generate coverage report", "where do I need more tests",
  "coverage gaps", "test coverage analysis".
---

# Code Coverage Analysis Skill

## How to Use

Invoke this skill by asking Copilot to analyze code coverage for a .NET project or solution. The skill supports two modes:

1. **Fresh collection** — run tests, collect coverage, and analyze (default)
2. **Existing coverage** — analyze a pre-existing `coverage.cobertura.xml` file

### Example prompts

```
Analyze code coverage for C:\repos\MyApp
```

```
Generate a coverage report for this solution
```

```
Coverage insights for C:\repos\MyApp\coverage.cobertura.xml
```

```
Where do I need more tests in C:\repos\MyApp?
```

```
Code coverage analysis — skip test run, use existing coverage at C:\path\to\coverage.cobertura.xml
```

### What you get

Two artifacts saved to `<artifact_root>/coverage/`:

| Artifact | Description |
|----------|-------------|
| `html/index.html` | Interactive HTML report via ReportGenerator — visual drill-down with line-by-line highlighting |
| `MMDDYYYY-CoverageAnalysis-<ProjectName>.md` | Markdown insights report with metrics, risk analysis, and prioritized recommendations |

---

## Purpose

Provide a one-shot code coverage workflow that goes beyond raw metrics. The HTML report gives developers the visual drill-down they already know from ReportGenerator. The markdown insights report adds an intelligence layer that ReportGenerator cannot provide: risk prioritization, flaky test signals, dead code detection, production risk heatmaps, and concrete "where to write tests next" recommendations.

---

## Inputs

- **Repository root or solution path** — the directory containing the `.sln` file, or a direct path to a `.csproj`. Treat the user's current working directory as the repo root if no path is provided.
- **Existing coverage file (optional)** — a path to a pre-existing `coverage.cobertura.xml`. If provided, skip test execution and analyze this file directly.
- **Output directory** — use the centralized artifact root for all output.
  - **Read artifact root:** Check `~\.copilot\unittest-artifact-root.txt` for the saved preference. If the file doesn't exist, ask the user where to save artifacts and save their choice there.
  - **Output location:** `<artifact_root>/coverage/`
  - Create the `coverage/` directory if it doesn't exist.

---

## Rules

- Do not modify product or test code.
- Do not assume fixed solution, project, or test project names — discover dynamically.
- If one or more `.sln` files exist at repo root, prefer running at the solution level.
- If `dotnet-reportgenerator-globaltool` is not installed, install it automatically via `dotnet tool install -g dotnet-reportgenerator-globaltool`.
- If `dotnet-coverage` is not installed, install it automatically via `dotnet tool install -g dotnet-coverage`. This tool is needed to merge and convert `.coverage` files to Cobertura XML.
- **Prefer `coverlet.collector` for coverage collection** — it produces Cobertura XML directly with full branch-level data. The built-in Microsoft Code Coverage collector (`--collect "Code Coverage"`) produces `.coverage` files that must be converted and **does not emit branch data** in the Cobertura export (all branch rates show 100% vacuously). See Step 1 for the detection and fallback logic.
- If tests fail during collection, still proceed with coverage analysis — failing tests produce valid coverage data.
- All artifacts must be written to the output folder, never to the repo itself.
- Always use Unicode emoji (❌, ✅, ⚠️, 🔥) — never use shortcodes like `:x:` or `:boom:`.

---

## Workflow

Execute the following steps in order using PowerShell from the repo root.

### Step 0: Validate prerequisites

```powershell
$repoRoot = Get-Location

# Read artifact root from saved preference
$artifactRootFile = Join-Path $env:USERPROFILE ".copilot\unittest-artifact-root.txt"
if (Test-Path $artifactRootFile) {
  $artifactRoot = (Get-Content $artifactRootFile -Raw).Trim()
} else {
  $artifactRoot = Join-Path $repoRoot "artifacts"
}
$outDir = Join-Path $artifactRoot "coverage"
New-Item -ItemType Directory -Force -Path $outDir | Out-Null

# Check for ReportGenerator
$rgInstalled = dotnet tool list -g 2>$null | Select-String "dotnet-reportgenerator-globaltool"
if (-not $rgInstalled) {
  Write-Host "Installing dotnet-reportgenerator-globaltool..."
  dotnet tool install -g dotnet-reportgenerator-globaltool
}

# Check for dotnet-coverage (needed for .coverage → Cobertura conversion)
$dcInstalled = dotnet tool list -g 2>$null | Select-String "dotnet-coverage"
if (-not $dcInstalled) {
  Write-Host "Installing dotnet-coverage..."
  dotnet tool install -g dotnet-coverage
}

# Discover solution or project
$sln = Get-ChildItem -Path $repoRoot -Filter *.sln -File | Select-Object -First 1
if ($sln) {
  Write-Host "Found solution: $($sln.FullName)"
} else {
  $csproj = Get-ChildItem -Path $repoRoot -Recurse -Filter *.csproj -File | Select-Object -First 1
  if ($csproj) { Write-Host "Found project: $($csproj.FullName)" }
}

# Detect coverlet.collector availability
# Check test projects for coverlet.collector package reference
$testProjects = Get-ChildItem -Path $repoRoot -Recurse -Filter *.csproj |
  Where-Object { $_.FullName -match 'Tests?' }
$hasCoverlet = $false
foreach ($tp in $testProjects) {
  $content = Get-Content $tp.FullName -Raw
  if ($content -match 'coverlet\.collector') {
    $hasCoverlet = $true
    break
  }
}
# Also check Directory.Packages.props if using Central Package Management
$dpp = Join-Path $repoRoot "Directory.Packages.props"
if ((Test-Path $dpp) -and (Get-Content $dpp -Raw) -match 'coverlet\.collector') {
  $hasCoverlet = $true
}

if ($hasCoverlet) {
  Write-Host "coverlet.collector: FOUND — will use XPlat Code Coverage (preferred)"
  $coverageCollector = "XPlat"
} else {
  Write-Host "coverlet.collector: NOT FOUND — will use Microsoft Code Coverage with conversion"
  Write-Host "  NOTE: Microsoft Code Coverage does not emit branch data in Cobertura exports."
  Write-Host "  For full branch coverage, add coverlet.collector to test projects."
  $coverageCollector = "Microsoft"
}
```

If the user provided an existing `coverage.cobertura.xml` path, skip to **Step 2**.

### Step 1: Collect coverage

Run `dotnet test` with coverage collection and TRX logging. The collector used depends on what's available in the project:

- **If `coverlet.collector` is present:** Use `--collect "XPlat Code Coverage"` — produces Cobertura XML directly with full branch data.
- **If `coverlet.collector` is NOT present:** Use `--collect "Code Coverage"` — produces `.coverage` files (Microsoft binary format), then convert to Cobertura XML using `dotnet-coverage merge`. ⚠️ This path **does not produce branch-level data** — note this limitation in the report.

**Important:** When testing multiple projects without a `.sln` file, run each test project individually. `dotnet test` does not accept multiple `.csproj` paths in a single invocation.

#### Path A: coverlet.collector available (preferred)

```powershell
$testResultsDir = Join-Path $outDir "test-results"
New-Item -ItemType Directory -Force -Path $testResultsDir | Out-Null

if ($sln) {
  dotnet test $sln.FullName `
    --results-directory $testResultsDir `
    --logger "trx;LogFileName=test-results.trx" `
    --collect "XPlat Code Coverage" `
    *> (Join-Path $outDir "test-console.txt")
} else {
  # Run each test project individually
  foreach ($tp in $testProjects) {
    $projName = [System.IO.Path]::GetFileNameWithoutExtension($tp.Name)
    dotnet test $tp.FullName `
      --results-directory (Join-Path $testResultsDir $projName) `
      --logger "trx;LogFileName=test-results.trx" `
      --collect "XPlat Code Coverage" `
      2>&1 | Tee-Object -Append -FilePath (Join-Path $outDir "test-console.txt")
  }
}

# Locate the generated Cobertura file(s)
$coverageFiles = Get-ChildItem -Path $testResultsDir -Recurse -Filter "coverage.cobertura.xml"
if ($coverageFiles) {
  Write-Host "Found $($coverageFiles.Count) coverage file(s)"
  Copy-Item $coverageFiles[0].FullName (Join-Path $outDir "coverage.cobertura.xml") -Force
} else {
  Write-Host "WARNING: No coverage file generated. Check test-console.txt for errors."
}
```

#### Path B: Microsoft Code Coverage fallback

```powershell
$testResultsDir = Join-Path $outDir "test-results"
New-Item -ItemType Directory -Force -Path $testResultsDir | Out-Null

if ($sln) {
  dotnet test $sln.FullName `
    --results-directory $testResultsDir `
    --logger "trx;LogFileName=test-results.trx" `
    --collect "Code Coverage" `
    *> (Join-Path $outDir "test-console.txt")
} else {
  # Run each test project individually
  foreach ($tp in $testProjects) {
    $projName = [System.IO.Path]::GetFileNameWithoutExtension($tp.Name)
    dotnet test $tp.FullName `
      --results-directory (Join-Path $testResultsDir $projName) `
      --logger "trx;LogFileName=test-results.trx" `
      --collect "Code Coverage" `
      2>&1 | Tee-Object -Append -FilePath (Join-Path $outDir "test-console.txt")
  }
}

# Find .coverage files (exclude duplicates in In\ subfolders)
$coverageFiles = Get-ChildItem -Path $testResultsDir -Recurse -Filter "*.coverage" |
  Where-Object { $_.DirectoryName -notmatch '\\In\\' }

if ($coverageFiles) {
  Write-Host "Found $($coverageFiles.Count) .coverage file(s) — merging to Cobertura XML..."
  $coberturaOutput = Join-Path $outDir "coverage.cobertura.xml"
  dotnet-coverage merge $coverageFiles.FullName --output $coberturaOutput --output-format cobertura
  Write-Host "Merged to: $coberturaOutput"
} else {
  Write-Host "WARNING: No .coverage files generated. Check test-console.txt for errors."
}
```

**Multiple test projects:** If the solution contains multiple test projects, `dotnet test` produces one `coverage.cobertura.xml` per test project. Collect ALL of them for ReportGenerator (it merges automatically). Build a semicolon-separated list of all coverage file paths for the `-reports` parameter in Step 2.

```powershell
$coveragePathList = ($coverageFiles | ForEach-Object { $_.FullName }) -join ";"
```

### Step 2: Generate HTML report via ReportGenerator

```powershell
$htmlDir = Join-Path $outDir "html"

# Use the user-provided file, or the collected file(s)
if ($existingCoverageFile) {
  $reportInput = $existingCoverageFile
} else {
  $reportInput = $coveragePathList
}

reportgenerator `
  "-reports:$reportInput" `
  "-targetdir:$htmlDir" `
  "-reporttypes:Html;Badges;TextSummary" `
  "-classfilters:-AutoGeneratedCode;-*.Migrations.*;-*.Grpc.Basket;-*.Grpc.Basket.*;-*SerializationContext;-*Generated*;-System.Runtime.CompilerServices*" `
  "-verbosity:Warning"

Write-Host "HTML report generated at: $htmlDir\index.html"
```

Also generate a `Summary.txt` for easy metric extraction:

```powershell
$summaryFile = Join-Path $outDir "Summary.txt"
if (Test-Path (Join-Path $htmlDir "Summary.txt")) {
  Copy-Item (Join-Path $htmlDir "Summary.txt") $summaryFile -Force
}
```

### Step 3: Parse Cobertura XML

Read the Cobertura XML and extract all metrics. Use PowerShell XML parsing:

```powershell
[xml]$coberturaXml = Get-Content (Join-Path $outDir "coverage.cobertura.xml")
$coverage = $coberturaXml.coverage

# Top-level metrics
$lineRate = [math]::Round([double]$coverage.'line-rate' * 100, 2)
$branchRate = [math]::Round([double]$coverage.'branch-rate' * 100, 2)
$linesValid = $coverage.'lines-valid'
$linesCovered = $coverage.'lines-covered'
$branchesValid = $coverage.'branches-valid'
$branchesCovered = $coverage.'branches-covered'
$complexity = $coverage.complexity

Write-Host "Line Coverage: $lineRate%"
Write-Host "Branch Coverage: $branchRate%"
Write-Host "Lines: $linesCovered / $linesValid covered"
Write-Host "Branches: $branchesCovered / $branchesValid covered"
```

Extract per-package (assembly) metrics:

```powershell
foreach ($pkg in $coverage.packages.package) {
  $pkgName = $pkg.name
  $pkgLineRate = [math]::Round([double]$pkg.'line-rate' * 100, 2)
  $pkgBranchRate = [math]::Round([double]$pkg.'branch-rate' * 100, 2)
  $pkgComplexity = $pkg.complexity
  Write-Host "$pkgName — Line: $pkgLineRate%, Branch: $pkgBranchRate%, Complexity: $pkgComplexity"
}
```

Extract per-class and per-method metrics for deep analysis:

```powershell
$classData = @()
foreach ($pkg in $coverage.packages.package) {
  foreach ($cls in $pkg.classes.class) {
    $methods = @()
    foreach ($m in $cls.methods.method) {
      $methods += @{
        Name = $m.name
        LineRate = [math]::Round([double]$m.'line-rate' * 100, 2)
        BranchRate = [math]::Round([double]$m.'branch-rate' * 100, 2)
        Complexity = [int]$m.complexity
      }
    }
    $classData += @{
      Package = $pkg.name
      ClassName = $cls.name
      Filename = $cls.filename
      LineRate = [math]::Round([double]$cls.'line-rate' * 100, 2)
      BranchRate = [math]::Round([double]$cls.'branch-rate' * 100, 2)
      Complexity = [int]$cls.complexity
      Methods = $methods
    }
  }
}
```

### Step 4: Enrich with git history (optional)

If the repo root is a git repository, gather change frequency and recency data to power the production risk heatmap. This step is optional — skip gracefully if git is unavailable or the directory is not a repo.

```powershell
$isGitRepo = Test-Path (Join-Path $repoRoot ".git")
$changeFrequency = @{}
$lastModified = @{}

if ($isGitRepo) {
  # Get change frequency per source file (last 6 months)
  $sixMonthsAgo = (Get-Date).AddMonths(-6).ToString("yyyy-MM-dd")
  $gitLogOutput = git -C $repoRoot log --since="$sixMonthsAgo" --name-only --pretty=format: -- '*.cs' 2>$null |
    Where-Object { $_ -ne '' -and $_ -notmatch 'Tests?\.cs$|Tests?/' }

  foreach ($file in $gitLogOutput) {
    if ($changeFrequency.ContainsKey($file)) {
      $changeFrequency[$file]++
    } else {
      $changeFrequency[$file] = 1
    }
  }

  # Get last commit date per uncovered file
  foreach ($cls in $classData | Where-Object { $_.LineRate -lt 50 }) {
    $relPath = $cls.Filename
    if ($relPath) {
      $lastCommitDate = git -C $repoRoot log -1 --format="%ai" -- $relPath 2>$null
      if ($lastCommitDate) {
        $lastModified[$relPath] = $lastCommitDate
      }
    }
  }

  Write-Host "Git enrichment complete: $($changeFrequency.Count) files with change history"
}
```

### Step 5: Analyze and generate markdown report

This is the core LLM analysis step. Using all the data collected in Steps 3-4, produce the full markdown insights report. Read the Cobertura XML data, the ReportGenerator summary, relevant source files for context on critical gaps, and the git history data.

**Analysis approach:**
1. Read the parsed coverage data (top-level metrics, per-assembly, per-class, per-method)
2. Read the ReportGenerator `Summary.txt` for formatted metrics
3. For the top uncovered files, read the actual source code to understand what's untested
4. Cross-reference with git change frequency for risk scoring
5. Generate all report sections (see Output Format below)

**When reading source files for context:**
- Only read files flagged as high-risk or critical gaps (top 10-15 files max)
- Use `view` with `view_range` for large files — focus on uncovered method signatures and their surrounding context
- Look for: public API surface, complexity indicators, error handling patterns, dependency injection, async patterns

Write the completed report to `<artifact_root>/coverage/MMDDYYYY-CoverageAnalysis-<ProjectName>.md`.

### Step 6: Print summary and offer to open HTML

```powershell
Write-Host ""
Write-Host "=== Coverage Analysis Complete ==="
Write-Host "HTML Report: $htmlDir\index.html"
Write-Host "Insights Report: $outDir\<report-filename>.md"
Write-Host ""
```

Ask the user if they would like to open the HTML report in their browser:

```powershell
Start-Process "$htmlDir\index.html"
```

---

## Output Format

### File Name

- **Format:** `MMDDYYYY-CoverageAnalysis-<ProjectName>.md`
- `MMDDYYYY` — date of the analysis
- `<ProjectName>` — extracted from the solution name, repo folder name, or project name
- **Check for existing files** in the coverage directory. If `Run1` exists, use `Run2`, etc. Append `-Run<N>` only when a file for the same date already exists.

**Example:** `coverage/03062026-CoverageAnalysis-eShop.md`

### Report Structure

```markdown
# Code Coverage Analysis — <ProjectName> (<MM/DD/YYYY>)

## Run Metadata

| Field | Value |
|-------|-------|
| **Solution/Project** | <path> |
| **Date** | <YYYY-MM-DD HH:mm:ss> |
| **Test Framework** | <MSTest / xUnit / NUnit / Mixed> |
| **Test Projects** | <count> |
| **Total Tests** | <N passed, N failed, N skipped> |
| **Coverage Tool** | <XPlat Code Coverage (coverlet) / Microsoft Code Coverage (converted)> |
| **Branch Data Available** | <Yes / No — Microsoft Code Coverage does not emit branch data> |
| **ReportGenerator Version** | <version> |

---

## Metrics Glossary

This report uses several metrics. Here is what each one means:

| Metric | Definition |
|--------|------------|
| **Line Coverage** | Percentage of executable source code lines that were executed at least once during the test run. A line with `hits > 0` is "covered." Lines that are not executable (comments, braces, blank lines) are excluded from the total. |
| **Branch Coverage** | Percentage of decision branches (if/else, switch cases, ternary operators, null-coalescing, short-circuit `&&`/`||`) where both the true and false paths were exercised. A method can have 100% line coverage but <100% branch coverage if only one side of a conditional is tested. *Requires `coverlet.collector` — Microsoft Code Coverage does not emit branch data.* |
| **Method Coverage** | Percentage of methods where at least one line was executed. A method is "covered" even if only its first line ran. **Full method coverage** means every line in the method was executed. |
| **Cyclomatic Complexity** | The number of linearly independent paths through a method's source code. Each `if`, `else`, `case`, `catch`, `&&`, `||`, `??`, and ternary `?:` adds 1 to the count. A method with no branches has complexity 1. Higher complexity = more paths to test = higher risk when untested. Typical thresholds: **1–5** (simple), **6–10** (moderate), **11–20** (complex), **>20** (very high risk). |
| **Risk Score** | A computed value combining complexity and coverage gaps: `complexity × (1 - line_coverage)`. Higher values indicate code that is both complex and poorly tested — the most likely source of production bugs. When git history is available, an enhanced formula is used: `complexity × (1 - line_coverage) × (1 + log₂(change_frequency + 1))`. |
| **Safety Net Score** | A composite 0–100 score estimating how well the test suite protects against regressions. Unlike raw coverage percentages, it weighs five dimensions: overall branch coverage, critical path coverage, error handling coverage, complexity coverage, and change coverage. See the Scoring section for the full rubric. |
| **Change Frequency** | The number of git commits that touched a file in the last 6 months. Files changed frequently are more likely to introduce regressions and benefit more from test coverage. |
| **Test-to-Code Ratio** | Lines of code in test projects divided by lines of code in source projects. Industry benchmark for well-tested projects is 1:1 to 2:1 (test LOC : source LOC). |

---

## Coverage Summary

| Metric | Value | Rating |
|--------|-------|--------|
| **Line Coverage** | <XX.X%> (<covered>/<total>) | <emoji> |
| **Branch Coverage** | <XX.X%> (<covered>/<total>) | <emoji> |
| **Method Coverage** | <XX.X%> (<covered>/<total>) | <emoji> |
| **Class Coverage** | <XX.X%> (<covered>/<total>) | <emoji> |
| **Total Complexity** | <N> | — |

Rating: 🔴 <50% | 🟡 50–79% | ✅ ≥80%

---

## Per-Assembly Breakdown

| # | Assembly | Line % | Branch % | Complexity | Uncovered Lines | Rating |
|---|----------|--------|----------|------------|-----------------|--------|
| 1 | <name> | <XX.X%> | <XX.X%> | <N> | <N> | <emoji> |

Sorted by line coverage ascending (worst first).

---

## 🔴 Critical Coverage Gaps

Top uncovered areas in business-logic assemblies that pose the highest risk.

### Gap N: <ClassName.MethodName> — 0% coverage

- **File:** `<path>`
- **Complexity:** <N>
- **Why it matters:** <1-2 sentences: what this code does, why it's risky untested>
- **What to test:** <specific scenarios — happy path, error cases, edge cases>

Focus on:
- Public methods/classes with 0% coverage in non-test, non-migration assemblies
- Methods with cyclomatic complexity ≥ 5 and 0% branch coverage
- Entry points: controllers, API endpoints, command/query handlers, middleware

---

## 🟡 Branch Coverage Gaps

Methods with reasonable line coverage but missing branch coverage — tests execute the code but skip conditional paths.

| # | Class.Method | Line % | Branch % | Missing Branches |
|---|-------------|--------|----------|------------------|
| 1 | <name> | <XX%> | <XX%> | <description of untested branches> |

Common patterns:
- Null checks where only the non-null path is tested
- Try/catch where only the happy path is tested
- Switch/pattern matching with missing cases
- Conditional logic (if/else) where only one branch is exercised
- Short-circuit evaluation (`&&`, `||`) where only one operand is evaluated

---

## 🔥 Complexity vs. Coverage Risk Matrix

Cross-referencing cyclomatic complexity with coverage to identify risk quadrants.

| Quadrant | Count | Description |
|----------|-------|-------------|
| 🔥 **Critical Risk** | <N> | High complexity (≥10) + Low coverage (<50%) |
| ⚠️ **Fragile** | <N> | High complexity (≥10) + High coverage (≥50%) |
| 💤 **Easy Win** | <N> | Low complexity (<10) + Low coverage (<50%) |
| ✅ **Safe** | <N> | Low complexity (<10) + High coverage (≥50%) |

### 🔥 Critical Risk Methods (Top 10)

| # | Class.Method | Complexity | Line % | Branch % | Risk Score |
|---|-------------|-----------|--------|----------|------------|
| 1 | <name> | <N> | <XX%> | <XX%> | <score> |

Risk score = `complexity × (1 - line_rate) × (1 - branch_rate)`

---

## 🧩 Flaky Test Indicators

Signals that suggest tests may be intermittently passing or exercising inconsistent paths.

- **Partial branch coverage patterns** — methods where exactly 50% of branches are covered often indicate tests that only run one side of a conditional
- **Happy-path-only coverage** — methods where line coverage is high but error-handling branches (catch, finally, guard clauses) show 0%
- **Non-deterministic path indicators** — methods involving `DateTime.Now`, `Random`, `Task.WhenAny`, or environment-dependent logic where coverage may vary between runs

| # | Class.Method | Branch % | Pattern | Concern |
|---|-------------|----------|---------|---------|
| 1 | <name> | <XX%> | <pattern> | <why this is a flaky signal> |

---

## 🪫 Dead Code Signals

Code that may be entirely unused — candidates for removal or documentation.

| # | Class/Method | Coverage | Last Modified | Visibility | Signal |
|---|-------------|----------|---------------|------------|--------|
| 1 | <name> | 0% | <date or "Unknown"> | <public/internal> | <why it looks dead> |

Detection criteria:
- 0% line coverage AND 0% branch coverage
- Public or internal visibility (private methods may be dead if their callers are also uncovered)
- Cross-referenced with git history: if last modified >12 months ago + 0% coverage = strong dead code signal
- Not in a test project, migration, or auto-generated file

---

## ♻️ Test Overlap Analysis

Areas where multiple test assemblies exercise the same production code, indicating potential redundancy or integration test duplication.

- **Assemblies covered by multiple test projects:** list any production assemblies that appear in coverage from >1 test project
- **Over-tested methods:** methods with high line+branch coverage AND low complexity — may indicate trivial code receiving disproportionate test investment

---

## 🔥 Production Risk Heatmap

Top 10 highest-risk files ranked by combined risk score.

| Rank | File | Line % | Branch % | Complexity | Changes (6mo) | Risk Score |
|------|------|--------|----------|------------|----------------|------------|
| 1 | <path> | <XX%> | <XX%> | <N> | <N> | <score> |

**Risk score formula:** `complexity × (1 - line_coverage) × (1 + log2(change_frequency + 1))`

Higher scores = more complex code, less tested, frequently changed. These are the files most likely to harbor production bugs.

If git history is not available, omit the change frequency column and use: `complexity × (1 - line_coverage)`.

---

## 🎯 Where to Write Tests Next

Prioritized recommendations for where new tests will have the highest impact.

### Priority 1: <ClassName> — <MethodName>

- **File:** `<path>`
- **Current coverage:** Line <XX%>, Branch <XX%>
- **Complexity:** <N>
- **Change frequency:** <N changes in 6 months>
- **What to test:**
  - <Specific scenario 1 — e.g., "null input parameter">
  - <Specific scenario 2 — e.g., "database exception during save">
  - <Specific scenario 3 — e.g., "concurrent access edge case">
- **Effort:** <⚡ Quick win (< 30 min) | 🔧 Medium effort (1-2 hours) | 🏗️ Significant (needs refactoring for testability)>
- **Expected impact:** <estimated coverage increase, e.g., "+5pp line coverage">

List up to 10 recommendations, sorted by expected impact descending.

**Effort classification criteria:**
- **⚡ Quick win** — method has injectable dependencies, simple inputs/outputs, low complexity
- **🔧 Medium effort** — needs mock setup, multiple scenarios, moderate complexity
- **🏗️ Significant** — tightly coupled code, sealed dependencies, static calls, needs interface extraction or refactoring before tests can be written

---

## 🛡️ Safety Net Score

An overall confidence metric (0–100) estimating how well the test suite protects against regressions.

**If line coverage is 0%, use the floor rule — skip the dimension table and output:**

> ### 🛡️ Safety Net Score: 0/100 🔴
>
> No production code was executed during the test run. The test suite provides zero regression protection.

**Otherwise, evaluate each dimension:**

| Dimension | Score (0–20) | Rationale |
|-----------|-------------|-----------|
| **Overall Branch Coverage** | <0–20> | <XX%> branch coverage — <rationale> |
| **Critical Path Coverage** | <0–20> | <N of M> entry points (controllers, handlers) covered |
| **Error Handling Coverage** | <0–20> | <XX%> of catch/finally blocks covered |
| **Complexity Coverage** | <0–20> | <XX%> of high-complexity methods (≥10) have ≥50% branch coverage |
| **Change Coverage** | <0–20> | <XX%> of frequently-changed files (top 20) have ≥50% coverage |

### 🛡️ Safety Net Score: <total>/100 <emoji>

Rating: 🔴 0–39 (Low confidence) | 🟡 40–69 (Moderate) | ✅ 70–100 (High confidence)

---

## 📈 Coverage Trend

*(Include this section only if a previous coverage analysis report exists in the output directory.)*

| Metric | Previous | Current | Delta |
|--------|----------|---------|-------|
| **Line Coverage** | <XX.X%> | <XX.X%> | <+/-X.Xpp> |
| **Branch Coverage** | <XX.X%> | <XX.X%> | <+/-X.Xpp> |
| **Safety Net Score** | <N/100> | <N/100> | <+/-N> |

### Files Improved
| File | Previous | Current | Delta |
|------|----------|---------|-------|
| <path> | <XX%> | <XX%> | <+Xpp> |

### Files Regressed
| File | Previous | Current | Delta |
|------|----------|---------|-------|
| <path> | <XX%> | <XX%> | <-Xpp> |

---

## 🩺 Additional Insights

### Exception Handler Coverage
- Catch blocks with 0% coverage = untested error handling paths
- List top N uncovered catch/finally/using-dispose blocks

### Async Path Coverage
- Async methods where only the successful `Task` completion path is tested
- Missing: `Task.FromException`, `Task.FromCanceled`, `OperationCanceledException`, timeout scenarios

### Guard Clause Coverage
- Parameter validation, null checks, range checks at method entry with 0% branch coverage
- These are often the cheapest tests to write and catch the most common production errors

### Error Handling Patterns
- Methods with try/catch where only the try block is covered
- Finally blocks with cleanup logic that is never exercised

### Constructor & DI Coverage
- Constructor logic (validation, initialization) with 0% coverage
- Dependency injection paths not exercised — services registered but never tested in isolation

### Test-to-Code Ratio
| Metric | Value |
|--------|-------|
| **Source LOC** | <N> |
| **Test LOC** | <N> |
| **Ratio** | <X:1> |

Industry benchmark: 1:1 to 2:1 (test LOC : source LOC) for well-tested projects.

---

## ✅ Actionable Summary

Top 5 actions to take, ordered by expected impact:

### Action 1: <title>
- **What:** <specific action — e.g., "Add tests for OrderService.ProcessPayment error paths">
- **Where:** `<file path>`
- **Effort:** <⚡ Quick win | 🔧 Medium | 🏗️ Significant>
- **Expected impact:** <e.g., "+8pp branch coverage, covers 3 critical error paths">
- **Suggested prompt:** `<a prompt you could give to Copilot or Testing Agent to generate these tests>`

### Action 2–5: ...

---

## HTML Report

The interactive HTML report is available at:

📂 `<artifact_root>/coverage/html/index.html`

Open it in a browser for line-by-line coverage visualization, file-level drill-down, and branch highlighting.
```

---

## How to Extract Data from Cobertura XML

### Top-Level Attributes

```xml
<coverage line-rate="0.85" branch-rate="0.72"
          lines-covered="1200" lines-valid="1412"
          branches-covered="340" branches-valid="472"
          complexity="285"
          version="..." timestamp="...">
```

- `line-rate` and `branch-rate` are decimals (0.0–1.0) — multiply by 100 for percentages
- `complexity` is the total cyclomatic complexity across all methods

### Per-Package (Assembly)

```xml
<package name="MyApp.Core" line-rate="0.90" branch-rate="0.78" complexity="120">
```

### Per-Class

```xml
<class name="MyApp.Core.OrderService" filename="src/Core/OrderService.cs"
       line-rate="0.75" branch-rate="0.60" complexity="25">
```

### Per-Method

```xml
<method name="ProcessPayment" signature="..." line-rate="0.50" branch-rate="0.33" complexity="8">
  <lines>
    <line number="42" hits="5" branch="True" condition-coverage="50% (2/4)" />
    <line number="43" hits="0" branch="False" />
  </lines>
</method>
```

Key fields:
- `hits="0"` — line never executed (uncovered)
- `branch="True"` + `condition-coverage="50% (2/4)"` — 2 of 4 branch conditions covered
- `complexity` — cyclomatic complexity of the method

### Identifying Uncovered Code

Lines with `hits="0"` are uncovered. For branch analysis, lines with `branch="True"` and `condition-coverage` less than `100%` have untested branches. The `condition-coverage` format is always `"XX% (covered/total)"`.

### Filtering Auto-Generated Code

Exclude from analysis:
- Classes in `*.Migrations.*` namespaces
- Classes with `[GeneratedCode]` or `[CompilerGenerated]` attributes (often reflected in class names containing `<>` or `__`)
- Files matching `*.g.cs`, `*.designer.cs`, `*.AssemblyInfo.cs`

---

## Style Guidelines

- Use **tables** for all structured data — metrics, breakdowns, recommendations
- Use **emoji** for visual scanning: ✅ ≥80%, 🟡 50–79%, 🔴 <50%, 🔥 critical risk, 🎯 recommendation, 🛡️ safety
- Use **bold** for key metrics and percentages
- Keep insight sections concise — prefer bullet points over prose
- Include full file paths for all file references
- Sort tables by severity/impact (worst first) unless otherwise specified
- Use **code blocks** for file paths, class names, and method names
- Always provide the "Suggested prompt" in the Actionable Summary — this creates a direct bridge to test generation skills

---

## Scoring: Safety Net Score (0–100)

The Safety Net Score provides a single metric for how well the test suite protects against regressions. It is **NOT** the same as code coverage percentage — it weighs coverage quality, critical path protection, and error handling.

### Floor Rule

**If overall line coverage is 0% (no production code was executed during the test run), the Safety Net Score is automatically 0/100.** Do not evaluate individual dimensions — a test suite that exercises zero production code provides zero regression protection, regardless of other factors like change frequency. Skip the per-dimension table and state:

```
### 🛡️ Safety Net Score: 0/100 🔴

No production code was executed during the test run. The test suite provides zero regression protection.
```

### Dimensions (0–20 each)

When line coverage is > 0%, evaluate each dimension:

| Dimension | 0–5 | 6–10 | 11–15 | 16–20 |
|-----------|-----|------|-------|-------|
| **Overall Branch Coverage** | <20% branches | 20–40% | 40–70% | >70% branches covered |
| **Critical Path Coverage** | <20% entry points tested | 20–50% | 50–80% | >80% controllers/handlers covered |
| **Error Handling Coverage** | 0% catch blocks covered | <20% | 20–50% | >50% catch/finally blocks exercised |
| **Complexity Coverage** | 0% high-complexity covered | <30% methods ≥10 complexity have tests | 30–60% | >60% have ≥50% branch coverage |
| **Change Coverage** | <20% of hot files tested | 20–40% | 40–70% | >70% of frequently-changed files covered |

**Dimension scoring rules:**
- Each dimension scores 0–20 based on the rubric above
- **Change Coverage special case:** If there are 0 file changes in the lookback period, score this dimension based on overall line coverage instead (the absence of changes does not mean the code is safe — it means there is no change signal to evaluate). Use: 0% coverage → 0, 1–20% → 5, 21–40% → 10, 41–70% → 15, >70% → 20.
- A dimension cannot score above 0 if the underlying metric is literally 0% (e.g., 0% branch coverage → 0/20 for Overall Branch Coverage, regardless of other factors)

### Rating

- ✅ **70–100** — High confidence: test suite likely catches most regressions
- 🟡 **40–69** — Moderate: significant gaps exist, especially in error handling or critical paths
- 🔴 **0–39** — Low confidence: test suite provides minimal regression protection

---

## Important Notes

- **coverlet.collector vs. Microsoft Code Coverage:** Always prefer `coverlet.collector` — it produces Cobertura XML directly with **full branch-level data** (`branch="True"` attributes, `condition-coverage` percentages on individual lines). The built-in Microsoft Code Coverage collector (`--collect "Code Coverage"`) produces binary `.coverage` files that can be converted to Cobertura XML via `dotnet-coverage merge`, but the resulting Cobertura **does not contain branch data** (all `branch-rate` values are vacuously 100% with 0 total branches). Without branch data, the Branch Coverage Gaps, Complexity Risk Matrix (branch dimension), and Flaky Test Indicators sections are significantly limited. When coverlet is unavailable, note this limitation prominently in the report header and use line coverage as the primary metric.
- **Adding coverlet.collector to a project:** If the project uses Central Package Management (Directory.Packages.props), add `<PackageVersion Include="coverlet.collector" Version="6.*" />` to Directory.Packages.props and `<PackageReference Include="coverlet.collector" />` to each test `.csproj`. If not using CPM, add `<PackageReference Include="coverlet.collector" Version="6.*" />` directly to each test project. Always include this as an actionable recommendation when coverlet is missing.
- **Multiple test projects without a .sln:** `dotnet test` does not accept multiple `.csproj` paths in a single invocation. Run each test project separately and merge coverage files afterward using `dotnet-coverage merge` or pass all paths to ReportGenerator via semicolon-separated `-reports`.
- **Auto-generated code noise:** gRPC protobuf stubs, JSON source-generated serialization contexts, OpenAPI generated code, and `System.Runtime.CompilerServices` emit classes that inflate the "uncovered" count but are not meaningful coverage targets. Use ReportGenerator `-classfilters` to exclude them, and do not flag them as dead code or critical gaps in the insights report.
- **Multiple test projects:** When a solution has multiple test projects, `dotnet test` produces separate Cobertura files. ReportGenerator merges them automatically. Ensure all coverage files are passed via the `-reports` parameter.
- **Coverage without assertions:** The Cobertura format only measures execution, not assertion quality. A line with `hits > 0` was *executed* but may not have been *verified*. The "Flaky Test Indicators" and "Additional Insights" sections address this limitation qualitatively.
- **Git history enrichment:** The production risk heatmap and change coverage dimension are most valuable when git history is available. The skill degrades gracefully without git — it simply omits change-frequency data and notes "Git history not available" in affected sections.
- **Large solutions:** For solutions with >50 assemblies, focus the detailed analysis (Critical Gaps, Where to Write Tests Next) on the top 15–20 assemblies by risk. Note the narrowed scope in the report header.
- **Pre-existing coverage files:** When analyzing an existing `coverage.cobertura.xml`, the skill cannot determine test pass/fail counts — note "Test results not available — analyzing pre-existing coverage file" in the metadata section.
