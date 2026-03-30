---
name: playwright-test-review
description: >
  Review Playwright end-to-end test files for quality, maintainability, and reliability. Analyzes
  .NET Playwright tests (NUnit, MSTest, xUnit) for common E2E anti-patterns: brittle selectors,
  missing waits, hardcoded test data, flaky patterns, missing Page Object Models, accessibility
  gaps, and insufficient assertions. Works with any Playwright tests — hand-written or AI-generated.
  When a Copilot or agent log is available, enriches the review with generation context.
  Use this skill when asked to review Playwright tests, analyze E2E test quality, assess end-to-end
  tests, or check Playwright tests for flakiness.
  Trigger phrases include "review playwright tests", "playwright test review",
  "check my playwright tests", "analyze playwright tests", "playwright quality check",
  "are my playwright tests flaky", "review e2e tests", "e2e test review".
---

# Playwright E2E Test Review Skill

## How to Use

Invoke this skill by asking Copilot to review Playwright end-to-end tests. Provide:

1. **A folder or file path** containing Playwright test files (`.cs` for .NET)
2. **The application under test** (optional) — the URL, project path, or description of what's being tested
3. **A Copilot/agent log** (optional) — if the tests were AI-generated, provide the log for enriched analysis

### Example prompts

```
Review the Playwright tests in C:\repos\MyApp\tests\e2e
```

```
E2E test review for C:\repos\MyApp\PlaywrightTests\LoginTests.cs
```

```
Check my Playwright tests for flakiness — tests are in C:\repos\MyApp\tests
```

```
Review Playwright tests in C:\repos\MyApp\tests, log is in C:\path\to\copilot-log.txt
```

```
Playwright quality check for this folder — focus on selector reliability
```

### What you get

A single markdown report saved to `<artifact_root>/reviews/`:

| Report | Description |
|--------|-------------|
| `MMDDYYYY-E2ETestReview-<Project>-Run<N>.md` | Full analysis: selector audit, wait/timing review, POM assessment, flakiness risk, assertion quality, and prioritized issues |

---

## Purpose

Evaluate Playwright end-to-end tests for quality, reliability, and maintainability. E2E tests have a fundamentally different failure profile than unit tests — they fail because of timing issues, brittle selectors, missing waits, environment dependencies, and UI changes, not because of logic errors. This skill applies E2E-specific quality criteria that unit test review skills do not cover.

When an AI-generation log is available, the review also assesses how well the generation process handled E2E-specific concerns (selector strategy, wait patterns, test isolation).

---

## Inputs

You will be provided:

- **Test file(s) or folder** — one or more Playwright test files (`.cs`), or a folder to scan recursively for Playwright test files. Detect Playwright tests by looking for:
  - NuGet references: `Microsoft.Playwright`, `Microsoft.Playwright.NUnit`, `Microsoft.Playwright.MSTest`
  - Using statements: `using Microsoft.Playwright`
  - Base classes: `PageTest`, `ContextTest`, `BrowserTest`
  - Playwright API calls: `Page.GotoAsync`, `Page.ClickAsync`, `Locator`, `Page.WaitForSelectorAsync`, etc.
- **Application context (optional)** — URL, project path, or description of the application under test. Helps assess whether tests cover the right user flows.
- **Generation log (optional)** — a Copilot Agent Mode log, Testing Agent log, or any agent output log. If provided, extract generation context (prompt, model, semantic search scope, errors encountered) and include a Generation Context section in the report.
- **Page/component source files (optional)** — if the user provides Razor/Blazor pages, controller routes, or page model files, use them to assess coverage of the actual UI surface.
- **Output directory** — use the centralized artifact root for all output.
  - **Read artifact root:** Check `~\.copilot\unittest-artifact-root.txt` for the saved preference. If the file doesn't exist, ask the user where to save artifacts and save their choice there.
  - **Output location:** `<artifact_root>/reviews/`
  - Create the `reviews/` directory if it doesn't exist.

---

## Rules

- Do not modify any test or product code.
- Do not assume fixed project, class, or file names — discover dynamically.
- If no Playwright test files are found, report this clearly and stop.
- All artifacts must be written to the output folder, never to the repo itself.
- Always use Unicode emoji (❌, ✅, ⚠️, 🔥) — never use shortcodes like `:x:` or `:boom:`.
- When assessing selectors, always prefer the Playwright best-practice hierarchy: role-based > test ID > CSS/XPath. Never penalize role-based or test-ID selectors.
- When analyzing .NET Playwright tests, recognize all three supported frameworks:
  - **NUnit:** `Microsoft.Playwright.NUnit` base classes (`PageTest`, `ContextTest`, `BrowserTest`)
  - **MSTest:** `Microsoft.Playwright.MSTest` base classes (`PageTest`, `ContextTest`, `BrowserTest`)
  - **xUnit:** Manual `IAsyncLifetime` or fixture-based setup with `Microsoft.Playwright`
- **Playwright version awareness** — check the `Microsoft.Playwright` NuGet version from the `.csproj`. If the version is below 1.27, do not recommend `GetByRole`, `GetByTestId`, `GetByText`, `GetByLabel`, `GetByPlaceholder`, or `GetByAltText` — these APIs were not available. Note the version limitation in the report.

### Scaling Rules

When the test suite is large (>20 test methods or >10 test files), apply these scaling rules to keep the report actionable:

- **Test Inventory table** — show all tests but use a compact format (collapse "What It Tests" to ≤10 words)
- **Selector Audit — Fragile Selectors Detail** — show only the top 20 most fragile selectors, sorted by risk. Add a summary line: "Showing 20 of N fragile selectors — N more omitted."
- **Assertion Depth per Test** — show only tests with 🔴 or ⚠️ ratings. Summarize passing tests as a count.
- **Test Classification** — show the summary table always; show the per-test detail table only for suites with ≤30 tests. For larger suites, list only Trivial tests individually (since those are the actionable findings).
- **Hardcoded Values Found** — show all (these are always actionable regardless of count)
- **All scoring sections and summary tables** — always show in full (these are the primary value of the report)

---

## Workflow

Execute the following steps in order.

### Step 1: Discover and inventory files

Scan the provided path for Playwright test files, Page Object Models, configuration, and the project file.

#### 1a: Find Playwright test files

```powershell
$testPath = "<user-provided-path>"

# If a single file, use it directly
if (Test-Path $testPath -PathType Leaf) {
  $testFiles = @(Get-Item $testPath)
} else {
  # Recursively find .cs files containing Playwright test indicators
  $testFiles = Get-ChildItem -Path $testPath -Recurse -Filter "*.cs" |
    Where-Object {
      $content = Get-Content $_.FullName -Raw
      # Must reference Playwright AND contain test attributes
      ($content -match 'using\s+Microsoft\.Playwright' -or
       $content -match ':\s*(PageTest|ContextTest|BrowserTest)' -or
       $content -match 'IPage\s+\w+.*Page\.GotoAsync') -and
      ($content -match '\[(Test|Fact|Theory|TestMethod|DataRow|TestCase)\]')
    }
}

Write-Host "Found $($testFiles.Count) Playwright test file(s)"
foreach ($f in $testFiles) { Write-Host "  - $($f.FullName)" }
```

If zero files found, report to user and stop.

#### 1b: Find Page Object Model files

Detect POMs using multiple signals — do NOT rely on a single keyword match:

```powershell
$pomFiles = Get-ChildItem -Path $testPath -Recurse -Filter "*.cs" |
  Where-Object {
    $content = Get-Content $_.FullName -Raw
    $path = $_.FullName

    # Must NOT be a test file (no test attributes)
    $isTestFile = $content -match '\[(Test|Fact|Theory|TestMethod)\]'
    if ($isTestFile) { return $false }

    # Signal 1: File is in a Pages/, PageObjects/, Models/, or Components/ folder
    $inPomFolder = $path -match '\\(Pages?|PageObjects?|Models?|Components?)\\' 

    # Signal 2: Has an IPage field/property or constructor parameter
    $hasIPage = $content -match 'IPage\s+\w+' -or $content -match 'private\s+(readonly\s+)?IPage\s+'

    # Signal 3: Calls Playwright APIs (Locator, GetByRole, GotoAsync, etc.)
    $callsPlaywright = $content -match '\.(Locator|GetByRole|GetByTestId|GetByText|GetByLabel|GotoAsync|ClickAsync|FillAsync)\s*\('

    # Signal 4: Class name ends in Page, PageModel, Component, or similar
    $pomClassName = $content -match 'class\s+\w*(Page|PageModel|PageObject|Component)\b'

    # Need at least 2 signals to classify as POM
    $signals = @($inPomFolder, $hasIPage, $callsPlaywright, $pomClassName) | Where-Object { $_ }
    return $signals.Count -ge 2
  }

Write-Host "POM files: $($pomFiles.Count)"
foreach ($f in $pomFiles) { Write-Host "  - $($f.FullName)" }
```

#### 1c: Find configuration and project files

```powershell
# Look for test configuration
$configFiles = Get-ChildItem -Path $testPath -Recurse -Include "*.runsettings", "playwright.config.*", "appsettings*.json" -ErrorAction SilentlyContinue

# Look for project file(s)
$csprojFiles = Get-ChildItem -Path $testPath -Recurse -Filter "*.csproj"

Write-Host "Config files: $($configFiles.Count)"
Write-Host "Project files: $($csprojFiles.Count)"
foreach ($f in $csprojFiles) { Write-Host "  - $($f.FullName)" }
```

#### 1d: Analyze project file(s)

Read each `.csproj` to extract Playwright-relevant metadata. This informs version-specific guidance, framework detection, and accessibility tool availability.

```powershell
foreach ($proj in $csprojFiles) {
  [xml]$projXml = Get-Content $proj.FullName
  $targetFramework = $projXml.Project.PropertyGroup.TargetFramework |
    Where-Object { $_ } | Select-Object -First 1

  # Extract Playwright NuGet package references and versions
  $packageRefs = $projXml.Project.ItemGroup.PackageReference |
    Where-Object { $_.Include -match 'Playwright|AxeCore|Verify|Shouldly' }

  foreach ($pkg in $packageRefs) {
    Write-Host "  Package: $($pkg.Include) v$($pkg.Version)"
  }

  Write-Host "  TargetFramework: $targetFramework"
}
```

Key metadata to extract:
- **Playwright version** — the `Microsoft.Playwright` NuGet version determines which APIs are available. `GetByRole`, `GetByTestId`, `GetByText`, `GetByLabel`, `GetByPlaceholder` were added in Playwright 1.27+. If the project uses an older version, do not recommend APIs that don't exist.
- **Playwright integration package** — `Microsoft.Playwright.NUnit`, `Microsoft.Playwright.MSTest`, or neither (xUnit / manual setup)
- **Accessibility tooling** — `Deque.AxeCore.Playwright` indicates axe-core is available (for the Accessibility section)
- **Assertion libraries** — `Shouldly`, `FluentAssertions`, `Verify` (affects assertion pattern detection)
- **Target framework** — .NET version

### Step 2: Read and analyze all test files

Read every discovered test file. For each file, extract:

- **Test class name and namespace**
- **Base class** (PageTest, ContextTest, BrowserTest, or custom)
- **Test framework** (NUnit, MSTest, xUnit)
- **Individual test methods** with their attributes
- **Test method names** — assess naming conventions (see quality patterns below)
- **All Playwright API calls** — build a complete inventory
- **All selectors used** — see Selector Extraction Reference below
- **All navigation calls** — `GotoAsync()` URLs
- **All wait/timing calls** — `WaitForSelectorAsync()`, `WaitForLoadStateAsync()`, `WaitForURLAsync()`, `WaitForResponseAsync()`, etc.
- **All assertion calls** — `Expect()` assertions, NUnit/xUnit assertions, custom assertions
- **Setup/teardown patterns** — `[SetUp]`, `[TearDown]`, `[OneTimeSetUp]`, constructor/dispose, `IAsyncLifetime`
- **Hardcoded values** — URLs, credentials, test data, magic strings
- **Screenshot/trace usage** — `ScreenshotAsync()`, `Tracing.StartAsync()`, `Tracing.StopAsync()`
- **Authentication patterns** — see Authentication Detection Reference below

Also read any POM files and config files discovered in Step 1.

#### Selector Extraction Reference

When extracting selectors from C# Playwright code, search for these exact method patterns and classify each by strategy:

| Strategy | C# Method Signatures to Search For | Reliability |
|----------|-------------------------------------|-------------|
| **Role-based** | `GetByRole(AriaRole.XXX)`, `GetByRole(AriaRole.XXX, new() { Name = "..." })` | ✅ Best |
| **Test ID** | `GetByTestId("...")` | ✅ Excellent |
| **Label** | `GetByLabel("...")` | ✅ Good |
| **Placeholder** | `GetByPlaceholder("...")` | 🟡 Good |
| **Text** | `GetByText("...")`, `GetByText("...", new() { Exact = true })` | 🟡 Good — fragile if text changes |
| **Alt text** | `GetByAltText("...")` | 🟡 Good |
| **Title** | `GetByTitle("...")` | 🟡 Good |
| **CSS** | `Locator("css=...")`, `Locator(".className")`, `Locator("#id")`, `Locator("div.card")`, `Locator("[data-attr]")` | 🟡 Moderate |
| **XPath** | `Locator("xpath=...")`, `Locator("//div/span")` | 🔴 Fragile |
| **Positional** | `.First`, `.Last`, `.Nth(N)`, `.Locator(":nth-child(N)")`, `Locator("... >> nth=N")` | 🔴 Very fragile |
| **Chained/filtered** | `.Filter(new() { HasText = "..." })`, `.Filter(new() { Has = ... })`, `.Locator("...")` (chained) | Inherits parent's rating |
| **Deprecated** | `QuerySelectorAsync("...")`, `QuerySelectorAllAsync("...")` | 🔴 Deprecated — should use Locator API |

Also detect **selector construction patterns**:
- **String interpolation in selectors** — `Locator($"#item-{id}")` — 🟡 dynamic but fragile if structure changes
- **Selectors stored in constants/fields** — good practice, note positively
- **Selectors built from POM properties** — good practice, note positively

#### Authentication Detection Reference

Detect these authentication patterns in test code:

| Pattern | What to Look For | Assessment |
|---------|------------------|------------|
| **Storage state reuse** | `StorageStateAsync()`, `StorageStatePath`, `NewContextAsync(new() { StorageStatePath = "..." })`, `BrowserNewContextOptions { StorageStatePath }` | ✅ Best — authenticate once, reuse |
| **Global setup auth** | `[OneTimeSetUp]` or `[AssemblyInitialize]` containing login flow + `StorageStateAsync` | ✅ Good — centralizes auth |
| **Per-test login via UI** | `GotoAsync("...login")` + `FillAsync("...password...")` inside `[SetUp]` or each test method | 🟡 Slow — OK for few tests, scales poorly |
| **Per-test login via API** | `APIRequestContext` + `PostAsync("...auth...")` in setup | 🟡 Better than UI login but still per-test |
| **Hardcoded credentials** | String literals matching password patterns (`"admin123"`, `"Password1!"`, `"P@ssw0rd"`) in `FillAsync` calls | 🔴 Security risk + breaks across environments |
| **Environment-based credentials** | `Environment.GetEnvironmentVariable("...")` for auth values | ✅ Good — configurable |

#### .runsettings / Configuration Analysis

If `.runsettings` files were found, read them and extract Playwright-relevant settings:

```xml
<!-- Common Playwright settings in .runsettings -->
<Playwright>
  <BrowserName>chromium|firefox|webkit</BrowserName>
  <LaunchOptions>
    <Headless>true|false</Headless>
  </LaunchOptions>
  <ExpectTimeout>5000</ExpectTimeout>
</Playwright>
```

Also check `appsettings.json` or environment variable setup for:
- Base URLs (`PLAYWRIGHT_BASE_URL`, `BASE_URL`, or configured via `BrowserNewContextOptions.BaseURL`)
- Browser selection
- Timeout configuration
- Screenshot/trace/video configuration

This data feeds into the Cross-Browser & Environment Readiness and Diagnostic Instrumentation sections.

### Step 3: Analyze generation context (if log available)

If a generation log was provided, extract:

- **Generation tool** — Copilot Agent Mode, Testing Agent, or other
- **Prompt used** — the original request
- **Model** — LLM model used
- **Semantic search scope** — what files the agent examined
- **Errors during generation** — build failures, selector issues, timeouts
- **Iteration count** — how many edit cycles occurred
- **Whether the agent used the Playwright MCP server** — look for MCP tool calls (navigate, click, snapshot, etc.)

### Step 4: Generate the review report

Using all data from Steps 1–3, generate the full review report following the Output Format below. This is the core LLM analysis step.

**Analysis approach:**
1. Classify every selector by strategy and assess fragility
2. Evaluate wait patterns — explicit waits vs. implicit/hardcoded delays
3. Check for POM usage and separation of concerns
4. Assess assertion depth and coverage
5. Identify flakiness risk factors
6. Evaluate test isolation and data management
7. Check for accessibility testing patterns
8. Assess cross-browser readiness
9. If generation log available, correlate generation decisions with test quality

Write the completed report to `<artifact_root>/reviews/MMDDYYYY-E2ETestReview-<ProjectName>-Run<N>.md`.

### Step 5: Print summary

```powershell
Write-Host ""
Write-Host "=== E2E Test Review Complete ==="
Write-Host "Report: <full path to report>"
Write-Host "Tests reviewed: <N> methods across <M> files"
Write-Host "Reliability score: <score>/100"
Write-Host ""
```

---

## Output Format

### File Name

- **Format:** `MMDDYYYY-E2ETestReview-<ProjectName>-Run<N>.md`
- `MMDDYYYY` — date of the review
- `<ProjectName>` — extracted from the project file, solution, or folder name
- `<N>` — run number starting at 1, incremented if a file for the same date already exists
- **Check for existing files** in the reviews directory matching `MMDDYYYY-E2ETestReview-*-Run*`.

**Example:** `reviews/03302026-E2ETestReview-eShop-Run1.md`

### Report Structure

```markdown
# Playwright E2E Test Review — <ProjectName> (<MM/DD/YYYY>)
```

### Review Metadata

| Field | Value |
|-------|-------|
| **Test Files** | `<count>` files, `<total lines>` LOC |
| **Test Framework** | `<NUnit / MSTest / xUnit>` with `Microsoft.Playwright.<framework>` |
| **Test Methods** | `<count>` |
| **Date** | `<YYYY-MM-DD HH:mm:ss>` |
| **Application Under Test** | `<URL or description — if provided>` |
| **Page Object Models** | `<count found or "None detected">` |
| **Generation Tool** | `<Copilot Agent Mode / Testing Agent / Hand-written / Unknown>` |
| **Generation Model** | `<model — if log available>` |

---

### Test Inventory

List every test method discovered across all files:

| # | File | Test Method | What It Tests | Selectors Used | Assertions |
|---|------|-------------|---------------|----------------|------------|
| 1 | `<file>` | `<method>` | `<brief description>` | `<count>` | `<count>` |

---

### 🎯 Selector Audit

This is the highest-impact section for E2E test reliability. Selectors are the #1 cause of flaky E2E tests.

#### Selector Strategy Distribution

| Strategy | Count | % | Reliability |
|----------|-------|---|-------------|
| **Role-based** (`GetByRole`) | `<N>` | `<X%>` | ✅ Best |
| **Test ID** (`GetByTestId`) | `<N>` | `<X%>` | ✅ Excellent |
| **Text-based** (`GetByText`, `GetByLabel`, `GetByPlaceholder`) | `<N>` | `<X%>` | 🟡 Good — fragile if text changes frequently |
| **CSS selector** (`Locator("css=...")`, `Locator(".class")`, `Locator("#id")`) | `<N>` | `<X%>` | 🟡 Moderate — breaks on styling/structure changes |
| **XPath** (`Locator("xpath=...")`) | `<N>` | `<X%>` | 🔴 Fragile — breaks on DOM structure changes |
| **Positional/index** (`:nth-child`, `.First`, `.Nth()`) | `<N>` | `<X%>` | 🔴 Very fragile — breaks on reordering |

#### Fragile Selectors (Detail)

For each selector rated 🟡 or 🔴, list:

| # | File:Line | Selector | Risk | Recommended Strategy |
|---|-----------|----------|------|---------------------|
| 1 | `<file>:<line>` | `<selector code>` | `<why it's fragile>` | `<recommended approach — see guidance below>` |

**Important:** Since this skill reviews code without access to the running application's DOM, do NOT suggest specific replacement selectors (e.g., "use `GetByRole(AriaRole.Button, new() { Name = "Submit" })`") — the actual ARIA roles, names, and test IDs cannot be determined from code alone. Instead, recommend the **strategy**:

- For CSS/XPath selectors → "Upgrade to `GetByRole` or `GetByTestId`. Use `playwright codegen <url>` or the Playwright Inspector (`playwright open <url>`) to discover the correct role-based selector."
- For positional selectors (`.First`, `.Nth()`) → "Add a `data-testid` attribute to the target element and use `GetByTestId`."
- For text-based selectors with frequently changing text → "Consider `GetByTestId` or `GetByRole` for stability."
- For deprecated `QuerySelectorAsync` → "Replace with the Locator API (`Page.Locator()` or `GetBy*` methods) for auto-waiting and retry support."

#### Selector Score

- **Score:** `<0–100>` — percentage of selectors using role-based or test-ID strategies
- Rating: 🔴 <40% | 🟡 40–74% | ✅ ≥75%

---

### ⏱️ Wait & Timing Analysis

E2E tests that rely on hardcoded delays or missing waits are inherently flaky.

#### Wait Pattern Inventory

| Pattern | Count | Assessment |
|---------|-------|------------|
| **Auto-waiting** (Playwright built-in — `ClickAsync`, `FillAsync`, etc.) | `<N>` | ✅ Correct — Playwright auto-waits for actionability |
| **Explicit waits** (`WaitForSelectorAsync`, `WaitForURLAsync`, `WaitForLoadStateAsync`, `WaitForResponseAsync`) | `<N>` | ✅ Good — explicit state synchronization |
| **`Expect` with polling** (`Expect(locator).ToBeVisibleAsync()`, `ToHaveTextAsync()`) | `<N>` | ✅ Best — assertion-based waiting |
| **Hardcoded delays** (`Task.Delay`, `Thread.Sleep`, `Page.WaitForTimeoutAsync`) | `<N>` | 🔴 Anti-pattern — source of flakiness |
| **Framework assertion after async action (no sync point)** — e.g., `await Page.ClickAsync(...)` → `Assert.AreEqual(...)` with no Playwright `Expect` or explicit wait in between | `<N>` | ⚠️ Risk — framework assertions (`Assert.That`, `Assert.Equal`, `Assert.AreEqual`) are snapshot-in-time checks with no retry. They may pass locally but fail in CI when the page hasn't updated yet. Replace with Playwright `Expect(locator).To*Async()` which polls until the condition is met. **Note:** Playwright `Expect` assertions ARE proper sync points — do NOT flag `Expect(locator).ToBeVisibleAsync()` or similar as missing a wait. |

#### Hardcoded Delays (Detail)

For each hardcoded delay found:

| # | File:Line | Code | Duration | Suggested Replacement |
|---|-----------|------|----------|----------------------|
| 1 | `<file>:<line>` | `<code>` | `<ms>` | `<what to wait for instead>` |

#### Timing Score

- **Score:** `<0–100>` — percentage of synchronization points using proper wait strategies (not hardcoded delays)
- Rating: 🔴 <60% | 🟡 60–84% | ✅ ≥85%

---

### 🏗️ Page Object Model Assessment

Evaluate whether tests use the POM pattern for maintainability.

| Criteria | Assessment |
|----------|------------|
| **POM classes exist** | `<Yes (N classes) / No>` |
| **Tests use POMs vs. inline selectors** | `<X% of interactions go through POMs>` |
| **POM encapsulation** | `<Do POMs expose raw selectors or semantic methods?>` |
| **POM reuse** | `<Are POMs shared across test files?>` |
| **Navigation in POMs** | `<Do POMs handle page navigation or is it inline?>` |

#### POM Recommendations

- If no POMs exist: recommend which page interactions should be extracted into POMs, based on repetition patterns across test files
- If POMs exist: assess quality — are they thin wrappers or do they provide meaningful abstraction?

#### POM Score

- **Score:** `<0–100>`
- 🔴 No POMs and >3 test files or >10 unique selectors repeated
- 🟡 POMs exist but incomplete — significant inline selector usage remains
- ✅ POMs cover ≥75% of page interactions with semantic method names

---

### ✅ Assertion Quality

E2E tests with weak or missing assertions provide false confidence.

#### Assertion Inventory

| Pattern | Count | Assessment |
|---------|-------|------------|
| **Playwright `Expect` assertions** (`Expect(locator).ToBeVisibleAsync()`, `.ToHaveTextAsync()`, `.ToHaveURLAsync()`, etc.) | `<N>` | ✅ Best — auto-retrying, E2E-aware |
| **Framework assertions** (NUnit `Assert.That`, xUnit `Assert.Equal`, MSTest `Assert.AreEqual`) | `<N>` | 🟡 OK — but no auto-retry; snapshot-in-time |
| **No assertion** (test navigates/clicks but never asserts outcome) | `<N>` | 🔴 Anti-pattern — test can't detect failures |
| **URL-only assertions** (only checks URL changed) | `<N>` | ⚠️ Weak — doesn't verify page content |

#### Assertion Depth per Test

| Test Method | Actions | Assertions | Ratio | Assessment |
|-------------|---------|------------|-------|------------|
| `<method>` | `<N>` | `<N>` | `<N:1>` | `<emoji>` |

- 🔴 0 assertions = test cannot detect failures
- ⚠️ 1 assertion for >5 actions = likely under-asserted
- ✅ Meaningful assertions covering the test's intent

#### Assertion Score

- **Score:** `<0–100>` — weighted by assertion quality (Playwright Expect > framework assert > none)
- Rating: 🔴 <50% | 🟡 50–79% | ✅ ≥80%

---

### 🧪 Test Classification

Categorize each test method:

| Test Method | Category | Notes |
|-------------|----------|-------|
| `<method>` | `<category>` | `<what it verifies>` |

Categories:
- **User Flow** — tests a complete user journey (e.g., login → navigate → action → verify)
- **Component Interaction** — tests a specific UI component's behavior (e.g., form validation, dropdown selection)
- **Navigation** — tests page routing and URL behavior
- **Visual/Layout** — tests visual appearance or responsive behavior
- **Error Handling** — tests error states, validation messages, error pages
- **Smoke** — minimal test verifying a page loads and key elements exist
- **Trivial** — tests something with near-zero chance of catching real bugs (e.g., page title exists)

#### Classification Summary

| Category | Count | % |
|----------|-------|---|
| User Flow | `<N>` | `<X%>` |
| Component Interaction | `<N>` | `<X%>` |
| Navigation | `<N>` | `<X%>` |
| Error Handling | `<N>` | `<X%>` |
| Smoke | `<N>` | `<X%>` |
| Trivial | `<N>` | `<X%>` |

---

### 👍 What's Well Done

Acknowledge good testing practices observed in the test suite. This section is important — especially for hand-written tests, recognizing what the team does right encourages adoption and builds trust in the review.

Look for and note:
- Role-based or test-ID selectors used consistently
- Proper use of Playwright `Expect` assertions with auto-retry
- Page Object Model pattern implemented and used across tests
- Storage state reuse for authentication
- Trace/screenshot collection configured for CI debugging
- Good test naming that describes user intent (e.g., `Should_ShowError_When_LoginFails`)
- Proper setup/teardown with test isolation
- Environment-configurable URLs and settings
- Good test categorization (mix of smoke, user flow, error handling)
- Cross-browser configuration
- Keyboard/accessibility testing patterns
- API-based test data setup instead of UI-based

Only include items that are **directly evidenced** in the test code. If nothing positive is observed, omit this section rather than fabricating praise.

---

### 📛 Test Naming Assessment

Evaluate test method naming conventions:

| Pattern | Example | Assessment |
|---------|---------|------------|
| **Descriptive user intent** | `Should_RedirectToDashboard_When_LoginSucceeds`, `LoginFlow_InvalidCredentials_ShowsError` | ✅ Best — communicates what the test verifies |
| **Feature-based** | `LoginPage_SubmitForm`, `Checkout_AddToCart` | 🟡 OK — describes scope but not expected outcome |
| **Generic/sequential** | `Test1`, `LoginTest`, `TestMethod3` | 🔴 Poor — no information about test intent |
| **Framework-generated** | `Theory_0`, `TestCase_1` | 🔴 Poor — auto-generated names provide no context in failure reports |

#### Naming Summary
- **Tests with descriptive names:** `<N>` (`<X%>`)
- **Tests with generic/poor names:** `<N>` (`<X%>`)

---

### 🔄 Test Isolation & Data Management

| Criteria | Assessment |
|----------|------------|
| **Test independence** | `<Do tests depend on execution order or shared state?>` |
| **Data setup** | `<How is test data created? API seeding, UI setup, hardcoded, fixtures?>` |
| **Data cleanup** | `<Is test data cleaned up? TearDown, API cleanup, database reset?>` |
| **Authentication** | `<How do tests authenticate? Storage state reuse, per-test login, hardcoded credentials?>` |
| **Parallel safety** | `<Can these tests run in parallel without conflicts?>` |
| **Environment coupling** | `<Are URLs, ports, connection strings hardcoded or configurable?>` |

#### Hardcoded Values Found

| # | File:Line | Type | Value | Risk |
|---|-----------|------|-------|------|
| 1 | `<file>:<line>` | `<URL/credential/data/port>` | `<value (redact secrets)>` | `<why this is a problem>` |

---

### 📸 Diagnostic Instrumentation

| Criteria | Present | Assessment |
|----------|---------|------------|
| **Screenshots on failure** | `<Yes/No>` | `<✅ or ❌ — critical for CI debugging>` |
| **Trace collection** | `<Yes/No>` | `<✅ or ❌ — Playwright traces are the gold standard for E2E debugging>` |
| **Video recording** | `<Yes/No>` | `<🟡 Nice to have for complex flows>` |
| **Console log capture** | `<Yes/No>` | `<⚠️ Helps catch JS errors that cause UI issues>` |
| **Custom logging** | `<Yes/No>` | `<Notes>` |

#### Recommendation

If traces or screenshots-on-failure are missing, provide the exact code to add them, adapted to the detected framework:

**NUnit (`Microsoft.Playwright.NUnit`):**
```csharp
// In your test class inheriting PageTest — override TearDown
[TearDown]
public async Task TearDown()
{
    if (TestContext.CurrentContext.Result.Outcome.Status == NUnit.Framework.Interfaces.TestStatus.Failed)
    {
        var screenshotPath = Path.Combine(TestContext.CurrentContext.WorkDirectory, 
            $"{TestContext.CurrentContext.Test.Name}.png");
        await Page.ScreenshotAsync(new() { Path = screenshotPath, FullPage = true });

        // Or use traces (more comprehensive):
        // Start in [SetUp]: await Context.Tracing.StartAsync(new() { Screenshots = true, Snapshots = true, Sources = true });
        // Stop here:
        var tracePath = Path.Combine(TestContext.CurrentContext.WorkDirectory,
            $"{TestContext.CurrentContext.Test.Name}.zip");
        await Context.Tracing.StopAsync(new() { Path = tracePath });
    }
}
```

**MSTest (`Microsoft.Playwright.MSTest`):**
```csharp
// In your test class inheriting PageTest
[TestCleanup]
public async Task TestCleanup()
{
    if (TestContext.CurrentTestOutcome != UnitTestOutcome.Passed)
    {
        await Page.ScreenshotAsync(new() { Path = $"{TestContext.TestName}.png", FullPage = true });
    }
}
```

**xUnit (manual setup):**
```csharp
// In your IAsyncLifetime.DisposeAsync or Dispose
public async Task DisposeAsync()
{
    // Traces need to be started in InitializeAsync
    await _context.Tracing.StopAsync(new() { Path = "trace.zip" });
    // Note: xUnit doesn't expose pass/fail status in cleanup — capture traces unconditionally
    // and delete on success, or use a custom test wrapper
}
```

Also recommend adding trace/screenshot configuration in `.runsettings` if the project uses one:
```xml
<Playwright>
  <LaunchOptions>
    <Headless>true</Headless>
  </LaunchOptions>
</Playwright>
```

---

### ♿ Accessibility Considerations

| Criteria | Assessment |
|----------|------------|
| **Role-based selectors** | `<Already captured in Selector Audit — summarize % here>` |
| **ARIA attribute usage in selectors** | `<Tests that rely on aria-label, aria-describedby, role>` |
| **axe-core integration** | `<Is Deque.AxeCore.Playwright or similar referenced?>` |
| **Keyboard navigation testing** | `<Any tests using Page.Keyboard for tab/enter/escape flows?>` |
| **Screen reader flow testing** | `<Any tests verifying ARIA live regions, focus management?>` |

This section is informational — it highlights accessibility testing opportunities, not failures.

---

### 🌐 Cross-Browser & Environment Readiness

| Criteria | Assessment |
|----------|------------|
| **Browser targets configured** | `<Chromium only / Chromium + Firefox + WebKit / Configurable>` |
| **Browser-specific workarounds** | `<Any browser-conditional logic in tests?>` |
| **Viewport/responsive testing** | `<Fixed viewport or multiple viewports tested?>` |
| **CI configuration** | `<Is there a CI pipeline config for running these tests?>` |
| **Headed vs. headless** | `<Configurable or hardcoded?>` |

---

### 🛡️ E2E Reliability Score

A composite score (0–100) estimating how reliable these tests will be in CI.

#### Scoring Rubric

**Selector Reliability (0–20):**
| Score | Criteria |
|-------|----------|
| 18–20 | ≥80% of selectors use role-based or test-ID strategies; zero XPath |
| 14–17 | 60–79% role/test-ID selectors; ≤2 XPath selectors |
| 8–13 | 40–59% role/test-ID selectors; some XPath/positional |
| 1–7 | <40% role/test-ID; significant XPath/positional usage |
| 0 | All selectors are CSS/XPath/positional with no role-based or test-ID selectors |

**Wait Strategy (0–20):**
| Score | Criteria |
|-------|----------|
| 18–20 | Zero hardcoded delays; all sync points use Playwright Expect or explicit waits |
| 14–17 | ≤1 hardcoded delay; ≥90% proper wait patterns |
| 8–13 | 2–5 hardcoded delays; ≥70% proper wait patterns |
| 1–7 | >5 hardcoded delays or <70% proper wait patterns |
| 0 | Pervasive `Thread.Sleep`/`Task.Delay` with no proper synchronization |

**Assertion Quality (0–20):**
| Score | Criteria |
|-------|----------|
| 18–20 | ≥80% of tests use Playwright `Expect` assertions; zero tests with no assertions |
| 14–17 | ≥60% Playwright Expect; all tests have at least one assertion |
| 8–13 | Mix of Playwright Expect and framework assertions; ≤2 tests with no assertions |
| 1–7 | Mostly framework assertions; multiple tests with no or URL-only assertions |
| 0 | No meaningful assertions in the test suite |

**Test Isolation (0–20):**
| Score | Criteria |
|-------|----------|
| 18–20 | Tests are independent; storage state or API-based auth; environment-configurable; parallel-safe |
| 14–17 | Mostly independent; minor shared state; configurable URLs; auth mostly centralized |
| 8–13 | Some order dependencies or shared state; hardcoded URLs; per-test UI login |
| 1–7 | Significant shared state; hardcoded credentials; tests fail if reordered |
| 0 | Tests are tightly coupled, share mutable state, and use hardcoded secrets |

**Diagnostic Coverage (0–20):**
| Score | Criteria |
|-------|----------|
| 18–20 | Traces + screenshots on failure + console capture all configured |
| 14–17 | Screenshots on failure + either traces or console capture |
| 8–13 | Screenshots on failure only, or traces only |
| 1–7 | Some logging but no screenshots or traces on failure |
| 0 | No diagnostic instrumentation — failures in CI produce no artifacts |

| Dimension | Score (0–20) | Rationale |
|-----------|-------------|-----------|
| **Selector Reliability** | `<0–20>` | `<X%>` of selectors use role-based or test-ID strategies |
| **Wait Strategy** | `<0–20>` | `<X%>` proper waits, `<N>` hardcoded delays |
| **Assertion Quality** | `<0–20>` | `<X%>` of tests have meaningful Playwright Expect assertions |
| **Test Isolation** | `<0–20>` | `<assessment of independence, data management, parallel safety>` |
| **Diagnostic Coverage** | `<0–20>` | `<screenshots on failure, traces, logging>` |

### 🛡️ E2E Reliability Score: `<total>`/100 `<emoji>`

Rating: 🔴 0–39 (High flakiness risk) | 🟡 40–69 (Moderate — needs improvement) | ✅ 70–100 (Production-ready)

---

### Generation Context

*Include this section ONLY if a generation log was provided.*

| Field | Value |
|-------|-------|
| **Generation Tool** | `<Copilot Agent Mode / Testing Agent / Other>` |
| **Prompt** | `<original prompt>` |
| **Model** | `<model name>` |
| **Playwright MCP Used** | `<Yes — N tool calls / No>` |
| **Semantic Search Scope** | `<files examined>` |
| **Edit Iterations** | `<N>` |
| **Build Errors** | `<N — describe types>` |

#### Generation Quality Assessment

- **Selector strategy chosen by agent** — did the agent default to CSS/XPath or use role-based selectors?
- **Wait pattern quality** — did the agent add proper synchronization or rely on implicit timing?
- **Test structure** — did the agent follow POM patterns or generate monolithic tests?
- **Error handling** — did the agent consider negative paths and error states?

---

### 📈 Review Trend

*Include this section ONLY if a previous E2E test review report exists in the output directory (matching `*-E2ETestReview-*` in the reviews folder).*

Compare with the most recent previous review:

| Metric | Previous | Current | Delta |
|--------|----------|---------|-------|
| **E2E Reliability Score** | `<N/100>` | `<N/100>` | `<+/-N>` |
| **Selector Score** | `<N/100>` | `<N/100>` | `<+/-N>` |
| **Timing Score** | `<N/100>` | `<N/100>` | `<+/-N>` |
| **Assertion Score** | `<N/100>` | `<N/100>` | `<+/-N>` |
| **Test Count** | `<N>` | `<N>` | `<+/-N>` |
| **Fragile Selectors** | `<N>` | `<N>` | `<+/-N>` |
| **Hardcoded Delays** | `<N>` | `<N>` | `<+/-N>` |

#### What Improved
- `<bullet points — specific improvements observed since last review>`

#### What Regressed
- `<bullet points — areas that got worse or new issues introduced>`

---

## Suggested Issues

After completing the analysis, suggest concrete improvements based on what was observed. These are actionable items the user or team can implement to improve their E2E test suite.

Generate up to **10 issues**, each specific to patterns observed in this review. Do not use pre-written or standing issues — every issue must be derived from evidence in the reviewed test code, configuration, or generation log.

### HIGH-ROI ISSUE PRIORITIZATION (Required)

When producing this section:

1) Assign an ROI tier to every issue:
   - ROI: HIGH
   - ROI: MEDIUM
   - ROI: LOW

2) Compute ROI using these criteria (use best judgment, no need to show the math):
   - Breadth (how many tests/files affected): 0–3
   - Impact (flakiness reduction, maintenance savings, CI reliability): 0–3
   - Fixability (clear pattern + straightforward refactor): 0–2
   - Urgency (currently causing CI failures or false positives): 0–2
   - Total: 0–10

   Map score to tier:
   - 8–10 => ROI: HIGH
   - 4–7  => ROI: MEDIUM
   - 0–3  => ROI: LOW

3) Sort the issues by ROI tier first (HIGH, then MEDIUM, then LOW).
   Within the same tier, sort by score descending.

4) Include the ROI tier in each issue header:
   Example: `### Issue 1 (ROI: HIGH): Hardcoded delays causing CI flakiness`

5) For each issue, include a short "Why ROI" line (1 sentence) stating the main reason.

### Issue Format

Each issue must follow this format:

```
### Issue N (ROI: <tier>): <Short title>

- **Problem:** <What was observed — cite specific test methods, selectors, file:line references>

- **Impact:** <How this affects test reliability, maintainability, or CI stability>

- **Suggested fix:** <Concrete code change or refactoring approach — include before/after code snippets where helpful>

- **Fix scope:** <How many files/tests are affected>

- **Why ROI:** <1 sentence — the main reason this issue merits its tier>
```

**Common E2E patterns to look for (use only if evidenced):**
- Hardcoded `Thread.Sleep` or `Task.Delay` instead of proper waits
- XPath or fragile CSS selectors when role-based alternatives exist — recommend `playwright codegen <url>` or the Playwright Inspector (`playwright open <url>`) to discover stable selectors
- Tests with zero or weak assertions (navigate and click, but never verify)
- Duplicated selectors across tests that should be in a Page Object Model
- Hardcoded URLs, ports, or credentials
- Missing screenshot/trace collection on failure
- Tests that depend on execution order or shared mutable state
- Login flow repeated per test instead of using stored authentication state — recommend `StorageStateAsync` pattern
- No error/negative path coverage (only happy paths tested)
- Missing `WaitForLoadStateAsync` or `WaitForURLAsync` after navigation
- Tests that work headed but fail headless (often viewport or timing issues)
- Missing cross-browser configuration
- Deprecated `QuerySelectorAsync` usage instead of Locator API
- Playwright version is outdated — missing access to newer `GetBy*` APIs

Only include issues that are **directly evidenced** by the reviewed tests. If fewer than 10 issues are supported by the data, include only those that are.

---

## Future: TypeScript Playwright Support

This skill currently targets .NET Playwright tests (C# with NUnit, MSTest, or xUnit). TypeScript Playwright test support is planned. When adding TypeScript support:

- Detect `.ts`/`.js` files with `import { test, expect } from '@playwright/test'`
- Adapt test method detection for `test('name', async ({ page }) => { ... })` pattern
- Adapt selector extraction for the same Playwright API (identical methods, different language)
- Adapt POM detection for TypeScript class patterns
- Configuration detection: `playwright.config.ts` instead of `.runsettings`
- Authentication: `storageState` in `playwright.config.ts` global setup
- The quality criteria, selector audit, wait analysis, and assertion assessment are framework-agnostic and apply identically

### Playwright Version Reference

For version-specific guidance, key API milestones:
- **1.27+** — `GetByRole`, `GetByTestId`, `GetByText`, `GetByLabel`, `GetByPlaceholder`, `GetByAltText`, `GetByTitle` added
- **1.29+** — `Filter` with `Has`/`HasNot`/`HasText`/`HasNotText` on locators
- **1.30+** — `Expect(locator).ToBeAttachedAsync()`, `Expect(locator).ToBeHiddenAsync()`
- **1.33+** — `Locator.Or()`, `Locator.And()` combinators
- **1.37+** — `Expect(page).ToHaveTitleAsync()` regex overloads
- **1.40+** — Clock API (`Page.Clock`) for time manipulation in tests

When reviewing tests on an older Playwright version, flag opportunities to upgrade for access to newer APIs — but do not recommend APIs that don't exist in the project's installed version.
