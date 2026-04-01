---
name: playwright-accessibility-audit
description: >
  Run an automated accessibility audit on a web application using Playwright and axe-core.
  Navigates to application pages (via URL or by starting the app from a project path), runs
  axe-core WCAG 2.1 AA analysis on each page, and produces a prioritized compliance report
  with remediation guidance. Supports both crawl-based page discovery and explicit URL lists.
  Uses the Playwright MCP server for page discovery and navigation, and Deque.AxeCore.Playwright
  for standards-based automated analysis.
  Use this skill when asked to audit accessibility, check WCAG compliance, run an a11y audit,
  scan for accessibility issues, or check a web app for accessibility problems.
  Trigger phrases include "accessibility audit", "a11y audit", "WCAG compliance check",
  "check accessibility", "scan for accessibility", "run accessibility tests",
  "accessibility scan", "is my app accessible", "check WCAG", "a11y check",
  "accessibility review", "audit accessibility".
---

# Playwright Accessibility Audit Skill

> **Note:** This skill currently supports **.NET web applications**. The target app can be built with any web framework (Razor Pages, Blazor, MVC, etc.), but the audit tooling runs as a .NET console app using `Deque.AxeCore.Playwright`. Auditing a non-.NET app by URL works fine — but the "project path" input mode requires a `.csproj` and `dotnet run`.

## How to Use

Invoke this skill by asking Copilot to run an accessibility audit. Provide one of:

1. **A URL** — if the application is already running (any web framework)
2. **A .NET project path** — if the app needs to be started (the skill will run it with `dotnet run`)
3. **Both** — a project path plus a URL override if the app runs on a non-default port

Optionally provide:

- **A list of pages/routes** — specific URLs or relative paths to audit (otherwise the skill crawls from the root)
- **WCAG level override** — defaults to WCAG 2.1 AA; can be overridden to A or AAA

### Example prompts

```
Run an accessibility audit on https://localhost:5001
```

```
Check WCAG compliance for C:\repos\MyApp — it's a Blazor app
```

```
A11y audit for https://localhost:5001 — check these pages: /, /login, /dashboard, /settings
```

```
Accessibility scan on C:\repos\MyApp — focus on WCAG 2.1 AAA
```

```
Is my app accessible? Project is at C:\repos\MyApp
```

### What you get

A single markdown report saved to `<project_path>/a11y-reports/` (or `<artifact_root>/reviews/` if only a URL was provided):

| Report | Description |
|--------|-------------|
| `MMDDYYYY-A11yAudit-<Project>-Run<N>.md` | Full audit: per-page violations, impact breakdown, WCAG criteria mapping, remediation guidance, compliance score, and prioritized issues |

---

## Purpose

Run an automated WCAG 2.1 AA accessibility audit on a live web application. Accessibility issues are among the most common and impactful defects in web applications — they affect real users, carry legal risk, and are often invisible to developers without specialized tooling.

This skill bridges the gap between "we should test accessibility" and actually doing it. It:

- **Discovers pages** automatically by crawling the app or accepting an explicit page list
- **Runs axe-core** — the industry-standard accessibility engine by Deque — against each page
- **Prioritizes violations** by impact level (critical → serious → moderate → minor)
- **Maps findings to WCAG success criteria** so teams know exactly which standards are affected
- **Provides remediation guidance** — not just "what's wrong" but "how to fix it"
- **Detects patterns** — identifies systematic issues that appear across multiple pages (e.g., all forms missing labels, global navigation lacking ARIA landmarks)
- **Produces a compliance score** — a single number stakeholders can track over time

### What axe-core can and cannot test

**axe-core reliably detects (~57% of WCAG criteria automatically):**
- Missing alt text, form labels, ARIA attributes
- Color contrast failures
- Missing landmarks and heading structure
- Keyboard trap risks, focus management issues
- Language attributes, link purposes, table structure

**axe-core cannot detect (requires manual testing):**
- Whether alt text is *meaningful* (it only checks presence)
- Logical reading order and content sequence
- Whether custom widgets are *usable* with assistive technology
- Cognitive load and plain language
- Video captions and audio descriptions quality
- Complex keyboard interaction patterns

The report clearly marks this boundary so teams know where automated coverage ends and manual review should begin.

---

## Inputs

You will be provided:

- **URL and/or project path** — at least one is required:
  - **URL**: The base URL of a running application (e.g., `https://localhost:5001`). The skill verifies reachability before proceeding.
  - **Project path**: The path to a .NET project (`.csproj` or directory containing one). The skill starts the app with `dotnet run` and discovers the URL from console output.
  - **Both**: If both are provided, the skill starts the app from the project path but uses the URL override for navigation.
- **Page list (optional)** — a list of specific URLs or relative paths to audit. If not provided, the skill crawls from the base URL.
- **WCAG level (optional)** — the conformance level to test against. Defaults to **WCAG 2.1 AA**. Accepted values:
  - `A` — WCAG 2.1 Level A only
  - `AA` — WCAG 2.1 Level A + AA (default)
  - `AAA` — WCAG 2.1 Level A + AA + AAA
- **Output directory** — determine the output location using this priority:
  1. **Project-local (preferred):** If a project path was provided, write to `<project_path>/a11y-reports/`. This keeps reports co-located with the project they describe. **Add `a11y-reports/` to the project's `.gitignore`** if it's not already there to avoid committing reports to source control.
  2. **Centralized artifact root (fallback):** If only a URL was provided (no project path), check `~\.copilot\unittest-artifact-root.txt` for a saved preference. If the file exists, write to `<artifact_root>/reviews/`. If it doesn't exist, ask the user where to save the report.
  - Create the output directory if it doesn't exist.

---

## Rules

- Do not modify any application source code.
- Do not assume fixed project, class, or file names — discover dynamically.
- All artifacts must be written to the output folder, never to the repo itself.
- Always use Unicode emoji (❌, ✅, ⚠️, 🔥, ♿) — never use shortcodes like `:x:` or `:boom:`.
- Default to **WCAG 2.1 AA** unless the user explicitly requests a different level.
- When the skill starts an application with `dotnet run`, it **must** stop the process when the audit completes. Use an async shell session to keep the app alive across tool calls, then terminate the session in the cleanup step.
- **Impact severity hierarchy** (from axe-core): `critical` > `serious` > `moderate` > `minor`. Always sort and prioritize by this order.
- **Do not report "passes" in detail** — the report focuses on violations and incomplete checks. Mention total pass count as a single line for context.
- The temporary audit project created during execution must be deleted after the audit completes. Do not leave temp files behind.
- If the application is behind authentication, ask the user for credentials or a pre-authenticated URL before proceeding. Do not attempt to bypass authentication.
- **Always generate a markdown report file**, even when zero violations are found. The report is the primary deliverable of this skill — never skip it.

### axe-core WCAG Tag Reference

Use these tags when configuring the axe-core run based on the requested conformance level:

| WCAG Level | axe-core Tags to Include |
|------------|--------------------------|
| A          | `wcag2a`, `wcag21a` |
| AA         | `wcag2a`, `wcag2aa`, `wcag21a`, `wcag21aa` |
| AAA        | `wcag2a`, `wcag2aa`, `wcag2aaa`, `wcag21a`, `wcag21aa`, `wcag21aaa` |

Always include `best-practice` tag findings in a separate "Best Practices" section (not counted toward the compliance score but included for awareness).

### Impact Level Reference

| Impact | Meaning | Emoji |
|--------|---------|-------|
| critical | Blocks access entirely for some users | 🔴 |
| serious | Creates significant barriers | 🟠 |
| moderate | Creates inconvenience but workarounds exist | 🟡 |
| minor | Minor annoyance, low user impact | 🔵 |

### Page Discovery Limits

- **Maximum pages to audit**: 50. If crawling discovers more than 50 pages, audit the first 50 (sorted by depth from root, then alphabetically) and note how many were skipped.
- **Maximum crawl depth**: 3 levels from the base URL.
- **Same-origin only**: Do not follow links to external domains.
- **Skip non-HTML resources**: Ignore links to PDFs, images, API endpoints, file downloads, etc.
- **Deduplicate**: Normalize URLs (remove trailing slashes, fragments, sort query parameters) before deduplication.

### Scaling Rules

When the report is large, apply these rules to keep it actionable:

- **Per-Page Results:**
  - If ≤10 pages: Show full violation tables for each page
  - If 11–25 pages: Show full tables for pages with critical/serious violations; show summary lines only for pages with only moderate/minor violations
  - If >25 pages: Show full tables for the 10 worst pages (by violation count weighted by impact); summarize the rest in a single table
- **Remediation Guide:**
  - Always show all critical and serious violations in full detail
  - For moderate violations: show full detail if ≤10, otherwise show top 10 and summarize the rest
  - For minor violations: always show as a summary table (rule, count, fix hint) — no full detail
- **WCAG Criteria Coverage, Impact Breakdown, Systematic Issues, Compliance Score** — always show in full (these are the primary value of the report)

---

## Workflow

Execute the following steps in order.

### Step 1: Determine target and start the application (if needed)

#### 1a: Parse user input

Determine what was provided:
- If a **URL** was provided, verify it's reachable:

```powershell
try { $response = Invoke-WebRequest -Uri "<URL>" -Method Head -TimeoutSec 10 -UseBasicParsing; Write-Host "✅ URL reachable: $($response.StatusCode)" } catch { Write-Host "❌ URL not reachable: $_" }
```

- If a **project path** was provided, find the `.csproj`:

```powershell
Get-ChildItem -Path "<PROJECT_PATH>" -Filter "*.csproj" -Recurse -Depth 2 | Select-Object -First 5 FullName, Name
```

#### 1b: Start the application (if project path was provided and no URL)

**Important:** The app must stay running across multiple tool calls. Use an **async shell session** (not `Start-Process`, which may terminate between shell invocations). Start the app in a dedicated background shell:

```powershell
# Run in an async/background shell session so the app stays alive
dotnet run --project "<CSPROJ_PATH>" --urls "https://localhost:5001"
```

> The LLM should start this command in async mode (e.g., a persistent background shell session). This keeps the `dotnet run` process alive while subsequent steps execute in separate shells. **Do not use `Start-Process`** — the spawned process may die between shell invocations depending on the execution environment.

Wait for the app to become available (in a separate shell):

```powershell
$maxRetries = 30; $retryCount = 0
while ($retryCount -lt $maxRetries) {
    try { Invoke-WebRequest -Uri "https://localhost:5001" -Method Head -TimeoutSec 2 -UseBasicParsing | Out-Null; Write-Host "✅ App started"; break } catch { $retryCount++; Start-Sleep -Seconds 2 }
}
if ($retryCount -eq $maxRetries) { Write-Host "❌ App did not start within 60 seconds" }
```

**Remember the shell session ID** — you will need it in Step 6 (Cleanup) to stop the app by terminating the shell session.

#### 1c: Verify the base URL is accessible

Navigate to the base URL using the Playwright MCP server and confirm the page loads:

Use `browser_navigate` (Playwright MCP) to navigate to the base URL. If the page loads successfully, proceed. If it fails (e.g., SSL error, auth redirect, connection refused), report the issue and stop.

### Step 2: Discover pages to audit

#### 2a: If the user provided a page list

Validate each URL or relative path. Convert relative paths to absolute URLs using the base URL. Verify each is reachable (HEAD request).

Build the page inventory table:

| # | URL | Source | Status |
|---|-----|--------|--------|
| 1 | https://localhost:5001/ | User-provided | ✅ Reachable |
| 2 | https://localhost:5001/login | User-provided | ✅ Reachable |

#### 2b: If no page list was provided — crawl the application

**Option A (preferred): Use the Playwright MCP server** if available in the current environment:

1. Navigate to the base URL with `browser_navigate`
2. Use `browser_snapshot` to get the accessibility tree (which includes all links)
3. Extract all same-origin `<a href>` links
4. For each link (up to depth 3), navigate and repeat
5. Build the page inventory from all discovered URLs

**Option B (fallback): Use PowerShell link extraction** if the Playwright MCP server is not available:

```powershell
# Fetch page HTML and extract same-origin links
$baseUri = [System.Uri]"<BASE_URL>"
$html = (Invoke-WebRequest -Uri $baseUri -UseBasicParsing).Content
$links = [regex]::Matches($html, "href=['""]([^'""]*)['""]") | ForEach-Object {
    $href = $_.Groups[1].Value
    try {
        $resolved = [System.Uri]::new($baseUri, $href)
        if ($resolved.Host -eq $baseUri.Host) { $resolved.AbsoluteUri }
    } catch {}
} | Sort-Object -Unique
$links | ForEach-Object { Write-Host $_ }
```

Repeat for each discovered link up to depth 3. This is less capable than MCP (no JavaScript rendering, won't find SPA routes) but works everywhere.

> ⚠️ **SPA limitation:** For single-page application frameworks (Blazor WASM, Angular, React), the PowerShell fallback will not discover client-side routes — the server-rendered HTML contains no `<a href>` links. For SPAs, either provide an explicit page list or ensure the Playwright MCP server is available.

Apply the Page Discovery Limits from the Rules section.

Output a discovery summary:

```
Page Discovery Summary:
  Starting URL: https://localhost:5001/
  Pages found: 12
  Max depth reached: 2
  External links skipped: 4
  Duplicate URLs removed: 3
```

### Step 3: Create and execute the axe-core audit

#### 3a: Create a temporary .NET console project

Create a minimal .NET console application that uses `Deque.AxeCore.Playwright` to audit each page:

```powershell
$tempDir = Join-Path $env:TEMP "a11y-audit-$(Get-Date -Format 'yyyyMMddHHmmss')"
New-Item -ItemType Directory -Path $tempDir -Force
Push-Location $tempDir
# Use the installed .NET SDK version (do not hardcode framework version)
$tfm = "net$(dotnet --version | ForEach-Object { $_.Split('.')[0] }).0"
dotnet new console -n "A11yAudit" --framework $tfm
Set-Location "A11yAudit"
dotnet add package Microsoft.Playwright
dotnet add package Deque.AxeCore.Playwright
```

**Note:** If `$env:TEMP` is restricted in the current environment, create the temp project under the artifact root directory instead: `$tempDir = Join-Path "<artifact_root>" "temp-a11y-audit"`

#### 3b: Write the audit program

Create a `Program.cs` with the following complete template. Replace `<URLS_PLACEHOLDER>` with the comma-separated URL list from page discovery, and adjust the WCAG tags based on the requested conformance level:

```csharp
using Deque.AxeCore.Commons;
using Deque.AxeCore.Playwright;
using Microsoft.Playwright;
using Newtonsoft.Json;

var urls = args.Length > 0
    ? args[0].Split(',', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
    : new[] { "<URLS_PLACEHOLDER>" };
var outputDir = args.Length > 1 ? args[1] : Path.Combine(Directory.GetCurrentDirectory(), "results");
Directory.CreateDirectory(outputDir);

using var playwright = await Playwright.CreateAsync();
await using var browser = await playwright.Chromium.LaunchAsync(new BrowserTypeLaunchOptions { Headless = true });
var context = await browser.NewContextAsync();
var page = await context.NewPageAsync();

// WCAG 2.1 AA + best-practice tags in a single run (avoids 2x audit time)
var allOptions = new AxeRunOptions
{
    RunOnly = RunOnlyOptions.Tags("wcag2a", "wcag2aa", "wcag21a", "wcag21aa", "best-practice")
};

for (int i = 0; i < urls.Length; i++)
{
    var url = urls[i];
    Console.WriteLine($"[{i + 1}/{urls.Length}] Auditing: {url}");
    try
    {
        // Verify app is still reachable before navigating
        var response = await page.GotoAsync(url, new PageGotoOptions
        {
            WaitUntil = WaitUntilState.NetworkIdle,
            Timeout = 30000
        });
        if (response == null || (int)response.Status >= 400)
        {
            Console.WriteLine($"  ⚠️ Page returned status {response?.Status ?? 0}, skipping");
            continue;
        }

        var results = await page.RunAxe(allOptions);

        // Separate WCAG vs best-practice violations by checking tags
        var wcagViolations = results.Violations?
            .Where(v => v.Tags.Any(t => t.StartsWith("wcag")))
            .ToArray() ?? Array.Empty<AxeResultItem>();
        var bpViolations = results.Violations?
            .Where(v => v.Tags.Contains("best-practice") && !v.Tags.Any(t => t.StartsWith("wcag")))
            .ToArray() ?? Array.Empty<AxeResultItem>();

        var resultsJson = JsonConvert.SerializeObject(results, Formatting.Indented);

        // Derive a readable file name from the URL path
        var safeName = new Uri(url).AbsolutePath.Trim('/').Replace("/", "_");
        if (string.IsNullOrEmpty(safeName)) safeName = "root";

        File.WriteAllText(Path.Combine(outputDir, $"results-{i:D3}-{safeName}.json"), resultsJson);

        Console.WriteLine($"  ✅ {wcagViolations.Length} WCAG violations, {bpViolations.Length} best-practice, {results.Passes?.Length ?? 0} passes, {results.Incomplete?.Length ?? 0} incomplete");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"  ❌ Error: {ex.Message}");
    }
}

Console.WriteLine($"\nAudit complete. Results saved to: {outputDir}");
```

**Key implementation notes:**
- `using Deque.AxeCore.Playwright;` is required — `RunAxe()` is an extension method in `PageExtensions`
- Use `Newtonsoft.Json.JsonConvert.SerializeObject()` for result serialization (axe-core NuGet depends on Newtonsoft internally; `System.Text.Json` may fail on the result objects)
- The `RunOnlyOptions.Tags()` static helper sets the correct `Type = "tags"` (plural) internally
- `WaitUntilState.NetworkIdle` ensures the page is fully loaded before running axe-core
- The health check before each navigation catches cases where the target app crashes mid-audit
- Adjust the WCAG tags in the template based on the user's requested conformance level — see the WCAG Tag Reference table in the Rules section
- **Single `RunAxe()` call per page** — WCAG and best-practice tags are combined in one run, then separated by tag during analysis. This halves audit time compared to two separate runs.

**Important**: The program must handle pages that fail to load (timeout, redirect loop, auth wall) gracefully — log the error and continue to the next page. Before each page navigation, verify the base URL is still reachable. If it becomes unreachable, attempt to restart the app (if the skill started it). If restart fails, report results for pages completed so far and note which pages were skipped.

#### 3c: Install Playwright browsers and execute

```powershell
# Build and install browsers (find playwright.ps1 dynamically — do not hardcode TFM or configuration)
dotnet build
$playwrightScript = Get-ChildItem -Path . -Recurse -Filter "playwright.ps1" | Select-Object -First 1
pwsh -File $playwrightScript.FullName install chromium

# Run the audit
dotnet run -- "<comma-separated-urls>" "<tempDir>/results"
```

#### 3d: Collect results

Read the JSON output files. Each file contains the axe-core results for one page, including:
- `violations`: Array of rule violations with impact, description, nodes, helpUrl, tags
- `passes`: Array of rules that passed
- `incomplete`: Array of rules that need manual review
- `inapplicable`: Array of rules not applicable to the page

### Step 4: Analyze results

#### 4a: Aggregate violations across pages

Build a consolidated view:

1. **By violation rule** — count how many pages each violation appears on (pattern detection)
2. **By page** — count total violations per page (identify worst pages)
3. **By impact** — count violations at each severity level
4. **By WCAG criterion** — map each violation to its WCAG success criterion using the `tags` array

#### 4b: Detect systematic patterns

A violation is **systematic** if it appears on ≥50% of audited pages. Flag these separately — they likely stem from a shared component or layout and fixing them once will resolve them everywhere.

Examples of systematic patterns:
- Missing landmark regions (same layout on every page)
- Missing skip-navigation link
- Color contrast issues in shared header/footer
- Form labels missing in a shared component

#### 4c: Calculate compliance score

Compute an **Accessibility Compliance Score (0–100)** using this formula:

```
Score = max(0, 100 - CriticalPenalty - SeriousPenalty - ModeratePenalty - MinorPenalty)

Where:
  CriticalPenalty = (critical_violations * 10)   — capped at 50
  SeriousPenalty  = (serious_violations * 5)     — capped at 30
  ModeratePenalty = (moderate_violations * 2)    — capped at 15
  MinorPenalty    = (minor_violations * 0.5)     — capped at 5
```

**Unique violations only** — if the same rule violation appears on 5 pages, count it once for scoring (since one fix resolves all instances).

> **Scoring caveat:** The score reflects unique violation *types*, not total instances. A single widespread critical violation (e.g., "no images have alt text" affecting 50 images) still scores relatively high. Always review the Impact Breakdown and Systematic Issues sections for the full picture — the score alone does not capture severity of widespread issues.

| Score Range | Rating | Emoji |
|-------------|--------|-------|
| 90–100 | Excellent | ✅ |
| 70–89 | Good | 🟢 |
| 50–69 | Needs Work | 🟡 |
| 30–49 | Poor | 🟠 |
| 0–29 | Critical | 🔴 |

### Step 5: Generate the report

Write the report to `<output_dir>/MMDDYYYY-A11yAudit-<Project>-Run<N>.md`.

Determine the output directory using the priority from the Inputs section:
- If a project path was provided: `<project_path>/a11y-reports/`
- If only a URL was provided: `<artifact_root>/reviews/` (from `~\.copilot\unittest-artifact-root.txt`)

Determine `<Project>`:
- From the `.csproj` file name (without extension) if a project path was provided
- From the URL hostname if only a URL was provided
- Fallback: `UnknownApp`

Determine `Run<N>`: Scan the output directory for existing `A11yAudit` reports for the same project and date, and increment the run number.

#### Clean audit shortcut

If the audit found **zero violations** across all pages, **still generate a full markdown report file** — but use this simplified structure:

```markdown
# ♿ Accessibility Audit Report

| Field | Value |
|-------|-------|
| **Application** | <Project name or URL> |
| **Base URL** | <base URL> |
| **Date** | <YYYY-MM-DD HH:mm> |
| **WCAG Level** | 2.1 AA |
| **Pages Audited** | <count> |
| **axe-core Version** | <version from NuGet> |
| **Compliance Score** | 100/100 ✅ Excellent |

## Executive Summary

No WCAG 2.1 AA violations were detected across <count> audited pages — excellent! The application passes all automated accessibility checks at the requested conformance level.

**Pages audited:** <list of page URLs>

> ⚠️ Automated testing covers approximately 57% of WCAG criteria. A clean automated audit does not guarantee full accessibility — see Limitations & Next Steps below.

## WCAG Criteria Coverage

| WCAG SC | Name | Status |
|---------|------|--------|
| 1.1.1 | Non-text Content | ✅ Pass |
| 1.3.1 | Info and Relationships | ✅ Pass |
| ... | (all tested criteria) | ✅ Pass |

## 💡 Best Practices

<include if any best-practice violations were found, otherwise write "No best-practice issues detected.">

## Limitations & Next Steps

<use the standard Limitations & Next Steps section>
```

Skip these sections (they would all be empty): Impact Breakdown, Systematic Issues, Per-Page Results, Remediation Guide, Suggested Issues.

### Step 6: Cleanup

1. **Delete the temporary audit project** created in Step 3:
   ```powershell
   Pop-Location
   Remove-Item -Path $tempDir -Recurse -Force
   ```
2. **Stop the application** if it was started in Step 1b — terminate the async shell session that is running `dotnet run`. This stops the app process cleanly.
3. Verify the report file was written successfully.

---

## Output Format

The report must follow this exact structure.

### Report Header

```markdown
# ♿ Accessibility Audit Report

| Field | Value |
|-------|-------|
| **Application** | <Project name or URL> |
| **Base URL** | <base URL> |
| **Date** | <YYYY-MM-DD HH:mm> |
| **WCAG Level** | 2.1 AA |
| **Pages Audited** | <count> |
| **axe-core Version** | <version from NuGet> |
| **Compliance Score** | <score>/100 <emoji> <rating> |
```

### Executive Summary

A 3–5 sentence summary covering:
- Overall compliance posture
- Number of unique violations by impact level
- Most critical finding
- Whether systematic patterns were detected
- Recommendation (pass/conditional pass/fail)

### Impact Breakdown

```markdown
## Impact Breakdown

| Impact | Unique Violations | Total Instances | Pages Affected |
|--------|-------------------|-----------------|----------------|
| 🔴 Critical | <count> | <instances across all pages> | <count> |
| 🟠 Serious | <count> | <instances> | <count> |
| 🟡 Moderate | <count> | <instances> | <count> |
| 🔵 Minor | <count> | <instances> | <count> |
| **Total** | **<count>** | **<instances>** | **<count>/<total pages>** |
```

### Systematic Issues

Only include this section if systematic patterns (≥50% of pages) were detected.

```markdown
## 🔥 Systematic Issues

These violations appear across most of the application and likely stem from shared components or layouts. Fixing these will resolve multiple instances at once.

| Rule | Impact | Pages Affected | Component Likely Source | WCAG Criterion |
|------|--------|----------------|------------------------|----------------|
| <rule-id> | <impact emoji> | <N>/<total> | <inferred component> | <SC number> |
```

### Per-Page Results

```markdown
## Per-Page Results

### Page: <URL>

| # | Rule | Impact | WCAG | Element | Description |
|---|------|--------|------|---------|-------------|
| 1 | color-contrast | 🟠 Serious | 1.4.3 | `<p class="subtitle">` | Element has insufficient color contrast ratio of 2.5:1 (required 4.5:1) |
| 2 | image-alt | 🟠 Serious | 1.1.1 | `<img src="hero.jpg">` | Image element missing alt attribute |

**Violations:** <count> | **Incomplete (needs manual review):** <count> | **Passes:** <count>
```

Apply the Scaling Rules from the Rules section for per-page detail.

### WCAG Criteria Coverage

```markdown
## WCAG Criteria Coverage

| WCAG SC | Name | Status | Violations | Pages |
|---------|------|--------|------------|-------|
| 1.1.1 | Non-text Content | ❌ Fail | 3 | 2 |
| 1.3.1 | Info and Relationships | ✅ Pass | 0 | — |
| 1.4.3 | Contrast (Minimum) | ❌ Fail | 7 | 5 |
| 2.1.1 | Keyboard | ⚠️ Incomplete | 1 | 1 |
```

Include all WCAG criteria that axe-core tested (from passes + violations + incomplete). Mark each as:
- ✅ **Pass** — no violations found
- ❌ **Fail** — one or more violations found
- ⚠️ **Incomplete** — axe-core flagged for manual review

### Remediation Guide

For each unique violation (deduplicated across pages), provide:

```markdown
## Remediation Guide

### 🟠 `image-alt` — Images must have alternate text (WCAG 1.1.1)

**Impact:** Serious | **Instances:** 5 across 3 pages

**What's wrong:** Image elements are missing `alt` attributes, making them invisible to screen reader users.

**How to fix:**
- Add descriptive `alt` text to each `<img>` element: `<img src="hero.jpg" alt="Team collaboration illustration">`
- For decorative images, use an empty alt: `<img src="divider.png" alt="" role="presentation">`
- For complex images (charts, diagrams), provide a longer description via `aria-describedby`

**Affected elements:**
| Page | Element |
|------|---------|
| /home | `<img src="hero.jpg">` |
| /about | `<img src="team.jpg">` |
| /about | `<img src="office.jpg">` |

**Learn more:** [Deque University — image-alt](https://dequeuniversity.com/rules/axe/4.7/image-alt)
```

**Ordering:** Sort remediation items by impact (critical first), then by instance count (most instances first).

Apply the Scaling Rules from the Rules section for remediation detail.

### Best Practices (Non-WCAG)

```markdown
## 💡 Best Practices

These findings are not WCAG violations but represent accessibility best practices that improve the user experience.

| Rule | Description | Instances | Pages |
|------|-------------|-----------|-------|
| <rule-id> | <description> | <count> | <count> |
```

### Incomplete Checks (Manual Review Needed)

```markdown
## ⚠️ Manual Review Needed

axe-core flagged these elements for manual review — automated testing cannot determine pass/fail.

| Rule | WCAG | Description | Pages | What to Check |
|------|------|-------------|-------|---------------|
| <rule-id> | <SC> | <description> | <count> | <manual check instruction> |
```

### Limitations & Next Steps

```markdown
## Limitations & Next Steps

### What this audit covers
- Automated WCAG 2.1 AA checks via axe-core (approximately 57% of WCAG criteria)
- Static page analysis at load time (not after user interactions)

### What this audit does NOT cover
- Keyboard navigation testing (tab order, focus traps, keyboard-only operation)
- Screen reader compatibility testing (NVDA, JAWS, VoiceOver)
- Cognitive accessibility (plain language, consistent navigation, error prevention)
- Dynamic content accessibility (modals, tooltips, live regions after interactions)
- Mobile/responsive accessibility
- Video/audio captions and transcripts

### Recommended next steps
1. Fix all critical and serious violations identified in this report — use the suggested prompts below
2. Manually review all items flagged in the "Manual Review Needed" section
3. Conduct keyboard-only navigation testing
4. Test with at least one screen reader (NVDA on Windows or VoiceOver on macOS)
5. Re-run this audit after fixes to verify remediation

### Fix it with Copilot

For each suggested issue in this report, a ready-to-use Copilot prompt is included. Copy and paste the prompt into Copilot to fix the issue. Prompts reference the specific files, elements, and WCAG criteria from the audit findings.
```

### Suggested Issues

For each violation that warrants a tracked work item, produce an issue in this format. **Every issue must include a `Copilot prompt` field** — a ready-to-use prompt the user can copy-paste into Copilot to fix the issue. The prompt should be specific: reference exact file paths, element types, WCAG criteria, and the fix strategy.

```markdown
## Suggested Issues

### Issue 1: Fix missing alt text on images (WCAG 1.1.1)
- **ROI:** 🔥 HIGH — Serious impact, affects 3 pages, single root cause
- **Scope:** src/Shared/ImageComponent.razor (likely)
- **What:** 5 images across 3 pages are missing alt attributes, making them invisible to screen readers
- **Fix:** Add descriptive alt text to all `<img>` elements; use `alt=""` for decorative images
- **WCAG:** 1.1.1 Non-text Content (Level A)
- **Copilot prompt:**
  ```
  Fix WCAG 1.1.1 violations in my project. The following <img> elements are missing alt attributes:
  - Pages/Index.cshtml: hero-banner.jpg, team-photo.jpg, office.jpg
  - Pages/Dashboard.cshtml: chart1.png, chart2.png, reports-icon.png
  Add descriptive alt text to each image. For decorative images use alt="". For the chart images,
  add alt text that summarizes what the chart shows.
  ```
```

**Copilot prompt guidelines:**
- Reference the specific files and elements from the audit findings — do not write generic prompts
- Include the WCAG criterion number so Copilot understands the requirement
- For systematic issues (e.g., shared layout), mention the root file to fix (e.g., `_Layout.cshtml`)
- Keep prompts self-contained — the user should be able to paste them without additional context
- For color contrast issues, include the failing color values and the required ratio

**ROI classification:**
| ROI | Criteria |
|-----|----------|
| 🔥 HIGH | Critical/serious impact AND (systematic OR ≥3 instances) AND clear fix path |
| ⚡ MEDIUM | Moderate impact OR (serious with only 1-2 instances) OR fix requires investigation |
| 📋 LOW | Minor impact OR best-practice-only OR complex fix with uncertain scope |

---

## axe-core Violation Quick Reference

The top 10 most common violations. For the full rule list, see [axe-core rule descriptions](https://github.com/dequelabs/axe-core/blob/develop/doc/rule-descriptions.md). Always use the actual impact level from the axe-core results — impact can vary by element context.

| Rule ID | WCAG SC | Typical Impact | Common Cause |
|---------|---------|----------------|--------------|
| `color-contrast` | 1.4.3 | Serious | Text doesn't meet 4.5:1 contrast ratio |
| `image-alt` | 1.1.1 | Serious–Critical | `<img>` missing alt attribute |
| `label` | 1.3.1 | Serious–Critical | Form element has no associated label |
| `link-name` | 2.4.4 | Serious | Anchor has no discernible text |
| `button-name` | 4.1.2 | Serious–Critical | Button has no discernible text |
| `html-has-lang` | 3.1.1 | Serious | `<html>` missing `lang` attribute |
| `select-name` | 1.3.1 | Serious–Critical | Select element missing accessible name |
| `document-title` | 2.4.2 | Serious | Page missing `<title>` element |
| `duplicate-id-active` | 4.1.1 | Critical | Active elements share duplicate IDs |
| `heading-order` | 1.3.1 | Moderate | Heading levels skip (e.g., h1 → h3) |

For WCAG success criteria mapping, use the `tags` array from axe-core results (e.g., `wcag111` → SC 1.1.1). For detailed criteria descriptions, see the [WCAG 2.1 Quick Reference](https://www.w3.org/WAI/WCAG21/quickref/).

---

## Troubleshooting

### Common issues during audit execution

| Problem | Cause | Solution |
|---------|-------|----------|
| `dotnet run` fails to start | Missing dependencies or build errors | Run `dotnet build` first and check for errors |
| App starts but URL not reachable | Wrong port or HTTPS cert issues | Try `--urls "http://localhost:5000"` (HTTP) or add `--environment Development` |
| axe-core returns empty results | Page hasn't fully loaded | Add `await page.WaitForLoadStateAsync(LoadState.NetworkIdle)` before `RunAxe()` |
| "Browser not installed" error | Playwright browsers not installed | Run `pwsh -File playwright.ps1 install chromium` in the build output directory |
| Auth redirect intercepts navigation | App requires login | Ask user for credentials or pre-auth cookie/token |
| Too many violations to be useful | Very early-stage app | Suggest running on key pages only, not full crawl |
| `Deque.AxeCore.Playwright` version conflict | NuGet version mismatch with Playwright | Ensure both packages are compatible versions |
| `Newtonsoft.Json` missing or conflict | `Deque.AxeCore.Commons` depends on Newtonsoft.Json transitively | Run `dotnet add package Newtonsoft.Json` explicitly if restore fails |
| Result serialization fails with `System.Text.Json` | `AxeResult` uses Newtonsoft.Json internally | Use `Newtonsoft.Json.JsonConvert.SerializeObject()` instead of `System.Text.Json.JsonSerializer.Serialize()` |
