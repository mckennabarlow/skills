---
name: playwright-test-generator
description: >
  Generate and execute Playwright end-to-end tests for .NET web applications. Analyzes application
  source code (routes, controllers, Razor views, page models, API endpoints) to understand the
  testable surface, then generates C# Playwright tests with Page Object Models. Supports NUnit,
  MSTest, and xUnit — auto-detects the project's framework or defaults to NUnit. Creates a real
  test project in the target repo that's committable and CI-runnable. Optionally enriches page
  discovery via Playwright MCP snapshots when available.
  Use this skill when asked to generate Playwright tests, scaffold E2E tests, create end-to-end
  tests, or bootstrap a Playwright test suite for a .NET web app.
  Trigger phrases include "generate playwright tests", "scaffold e2e tests", "create playwright
  tests", "bootstrap playwright test suite", "generate e2e tests for my app",
  "playwright test generator", "create end-to-end tests".
---

# Playwright Test Generator Skill

## How to Use

Invoke this skill by asking Copilot to generate Playwright E2E tests for a .NET web application. Provide:

1. **A project or solution path** (required) — the path to the web application's `.csproj`, `.sln`, or root folder
2. **A running URL** (optional) — if the app is already running, provide the URL to skip auto-start
3. **A page list** (optional) — specific pages/routes to test; otherwise the skill discovers them from code
4. **Focus areas** (optional) — e.g., "focus on the checkout flow" or "only test authenticated pages"

### Example prompts

```
Generate Playwright tests for C:\repos\MyApp\MyApp.csproj
```

```
Scaffold E2E tests for C:\repos\eShop — the app is running at https://localhost:5001
```

```
Create Playwright tests for C:\repos\MyApp — focus on the user registration and login flows
```

```
Generate end-to-end tests for C:\repos\MyApp.sln — only test the admin pages
```

```
Bootstrap a Playwright test suite for this project — include auth setup
```

### What you get

| Output | Location |
|--------|----------|
| Test project (`<AppName>.E2ETests/`) | In the target repo, alongside the app |
| Page Object Models | `<AppName>.E2ETests/PageObjects/` |
| Test files | `<AppName>.E2ETests/Tests/` |
| Auth fixture (if auth detected) | `<AppName>.E2ETests/Infrastructure/` |
| Generation report | `<artifact_root>/reviews/MMDDYYYY-E2ETestGen-<Project>-Run<N>.md` |

---

## Purpose

Generate meaningful Playwright E2E tests that exercise real user flows — not just visibility checks. This skill reads your application's source code to understand routes, page structure, forms, navigation, and auth patterns, then generates tests that verify the app actually *works*, not just that it *renders*.

The generated tests are designed to score well against the `playwright-test-review` skill's quality criteria:
- Role-based selectors (never raw CSS/XPath)
- Page Object Models for shared elements
- Proper auth fixtures using `StorageStateAsync`
- No `Thread.Sleep` or `Task.Delay` — only Playwright auto-waiting
- Parallel-safe test isolation
- Descriptive assertions with clear failure messages

### Where this skill fits in the pipeline

```
playwright-test-generator  →  playwright-test-review  →  playwright-accessibility-audit
    (generate & run)            (assess quality)            (check compliance)
```

This skill generates and runs. Review and accessibility auditing are separate skill invocations.

---

## Inputs

You will be provided:

- **Project path (required)** — a `.csproj` file, `.sln` file, or folder containing a .NET web application. The skill scans for:
  - ASP.NET Core MVC (controllers + views)
  - Razor Pages (`.cshtml` with `@page` directive or PageModel classes)
  - Blazor Server / Blazor WebAssembly (`@page` directives in `.razor` files)
  - Minimal API (`MapGet`, `MapPost`, etc. in `Program.cs` or endpoint files)

- **Running URL (optional)** — if the app is already running, provide the base URL (e.g., `https://localhost:5001`). Skips auto-start and goes directly to page discovery.

- **Page list (optional)** — specific routes to test. If not provided, the skill discovers routes from code.

- **Focus areas (optional)** — natural language description of what to focus on. The skill uses this to prioritize which routes get user flow tests vs. smoke tests.

- **Output directory** — use the centralized artifact root for the generation report.
  - **Read artifact root:** Check `~\.copilot\unittest-artifact-root.txt` for the saved preference. If the file doesn't exist, ask the user where to save artifacts and save their choice there.
  - **Report location:** `<artifact_root>/reviews/`
  - **Test files location:** `<project_root>/<AppName>.E2ETests/` (in the repo, not in artifact root)

---

## Rules

### General

- All artifacts (reports) must be written to the artifact root, never to the repo itself.
- Generated test code goes into the repo at `<AppName>.E2ETests/` — this is intended to be committed.
- Always use Unicode emoji (❌, ✅, ⚠️, 🔥) — never shortcodes like `:x:`.
- Do not modify any existing application code. Only create new test project files.
- If the project already has a Playwright test project, detect it and generate into it (don't create a duplicate).

### Framework Detection

Detect the test framework by scanning the solution/project for existing test projects:

```
Scan solution for *.csproj containing Playwright references
    │
    ├── Found: Microsoft.Playwright.NUnit    → Use NUnit
    ├── Found: Microsoft.Playwright.MSTest   → Use MSTest
    ├── Found: Microsoft.Playwright (only)   → Check for xUnit/NUnit/MSTest refs → match
    └── Not found: No existing Playwright tests
          │
          ├── Other test projects exist? → Match their framework
          └── No test projects at all   → Default to NUnit
```

### Selector Strategy

Always generate selectors in this priority order (matching the playwright-test-review quality rubric):

1. **Role-based** (best): `Page.GetByRole(AriaRole.Button, new() { Name = "Submit" })`
2. **Test ID**: `Page.GetByTestId("submit-button")`
3. **Label**: `Page.GetByLabel("Email address")`
4. **Placeholder**: `Page.GetByPlaceholder("Search...")`
5. **Text**: `Page.GetByText("Welcome back")`

**Never generate:**
- Raw CSS selectors (`Page.Locator(".btn-primary")`)
- XPath selectors (`Page.Locator("//div/button")`)
- Positional selectors (`.First`, `.Nth(N)`)
- Deprecated APIs (`QuerySelectorAsync`)

When generating from code analysis alone (no MCP snapshot), use the most semantically appropriate selector based on the HTML element type discovered in Razor/Blazor views. When an MCP aria snapshot is available, use the exact roles and names from the snapshot.

### Naming Conventions

- **Test project:** `<AppName>.E2ETests` (e.g., `eShop.E2ETests`)
- **POM classes:** `<PageName>Page.cs` in `PageObjects/` (e.g., `LoginPage.cs`, `HomePage.cs`)
- **Test classes:** `<PageName>Tests.cs` in `Tests/` (e.g., `LoginTests.cs`, `HomeTests.cs`)
- **Auth fixture:** `AuthSetup.cs` in `Infrastructure/`
- **Test methods:** `<Action>_Should_<ExpectedResult>` (e.g., `Login_Should_RedirectToDashboard`)
- **Namespace:** `<AppName>.E2ETests.Tests`, `<AppName>.E2ETests.PageObjects`, `<AppName>.E2ETests.Infrastructure`

### Scaling Rules

| App size | Behavior |
|----------|----------|
| ≤5 routes | Generate user flow + smoke tests for all routes |
| 6–15 routes | Generate user flow tests for focus areas or top routes; smoke tests for the rest |
| 16–30 routes | User flows for top 5–8 routes; smoke tests for top 15; list remaining as "not covered" |
| >30 routes | User flows for top 5; smoke for top 10; report the full route inventory with coverage plan |

Always generate at least one smoke test per route discovered, up to the scaling cap.

### Playwright Version Awareness

Check the `Microsoft.Playwright` NuGet version from the `.csproj`. If the version is below 1.27, do not generate `GetByRole`, `GetByTestId`, `GetByText`, `GetByLabel`, `GetByPlaceholder`, or `GetByAltText` — these APIs were not available. Fall back to `Page.Locator()` with accessible CSS selectors and note the version limitation in the report.

---

## Workflow

### Step 1: Discover the application surface

#### 1a: Find the project and determine app type

```powershell
# If given a .sln, find all web app projects
Get-ChildItem -Path "<solution_dir>" -Recurse -Filter "*.csproj" |
  ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    if ($content -match 'Microsoft\.NET\.Sdk\.Web' -or
        $content -match 'Microsoft\.AspNetCore') {
      $_.FullName
    }
  }
```

For each web project, classify the app type by scanning for indicators:

| Indicator | App Type |
|-----------|----------|
| `Pages/` directory with `@page` in `.cshtml` files | Razor Pages |
| `Controllers/` directory with classes inheriting `Controller` / `ControllerBase` | MVC |
| `.razor` files with `@page` directives | Blazor |
| `MapGet` / `MapPost` / `MapPut` / `MapDelete` in `Program.cs` | Minimal API |

An app can be a hybrid (e.g., MVC + Razor Pages). Detect all types present.

**Multiple web projects:** If the solution contains more than one web project, ask the user which one to test. If only one project has controllers or pages (non-API), prefer it automatically. Do not attempt to generate tests for multiple web apps in a single run.

#### 1b: Extract routes

**Razor Pages:**
```powershell
# Find all Razor Pages with @page directive
Get-ChildItem -Path "<project>/Pages" -Recurse -Filter "*.cshtml" |
  ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    if ($content -match '@page\s*"?([^"\r\n]*)"?') {
      $routeTemplate = $Matches[1].Trim()
      if ([string]::IsNullOrEmpty($routeTemplate)) {
        # Bare @page — derive route from file path convention:
        # Pages/Products/Index.cshtml → /Products
        # Pages/Products/Details.cshtml → /Products/Details
        # Pages/Index.cshtml → /
        $relativePath = $_.FullName.Replace("<project>\Pages\", "").Replace("\", "/")
        $routeTemplate = "/" + ($relativePath -replace '\.cshtml$', '' -replace '/Index$', '')
        if ($routeTemplate -eq "/") { $routeTemplate = "/" }
      }
      @{ File = $_.FullName; Route = $routeTemplate }
    }
  }
```

**MVC Controllers:**
- Read each controller class. Extract `[Route]`, `[HttpGet]`, `[HttpPost]` attributes.
- For conventional routing, map controller name + action name to `/Controller/Action`.
- Note `[Authorize]` attributes on controllers or actions.

**Blazor:**
- Scan `.razor` files for `@page "/route"` directives.
- Note `@attribute [Authorize]` for auth-protected pages.
- **Blazor WASM vs Server:** Check the project SDK and package references to distinguish:
  - `Microsoft.NET.Sdk.BlazorWebAssembly` or `Microsoft.AspNetCore.Components.WebAssembly` → **Blazor WASM**
  - `Microsoft.NET.Sdk.Web` with `Microsoft.AspNetCore.Components.Server` → **Blazor Server**
  - For Blazor WASM: the app is served as static files — there are no server-side routes. All navigation is client-side. Tests must wait for WebAssembly to load before interacting: use `Page.WaitForLoadStateAsync(LoadState.NetworkIdle)` instead of `DOMContentLoaded`. Page snapshots via MCP are especially valuable here since the HTML is fully client-rendered.
  - For Blazor Server: pages render server-side initially. Standard `DOMContentLoaded` waits work. SignalR connection must be established before interactive elements work — wait for the `blazor-connected` class or similar indicator.

**Minimal API:**
- Read `Program.cs` (and any files it references) for `app.MapGet("/path", ...)` patterns.
- These are typically API-only — generate API test patterns, not page navigation tests.

#### 1c: Read page structure for testable elements

For each discovered route, read the corresponding view/page file to identify:

| Element | What to generate |
|---------|-----------------|
| `<form>` with inputs and submit button | Form submission user flow test |
| `<a asp-page>` / `<a asp-action>` navigation links | Navigation test |
| `<table>` or data-bound lists (`@foreach`) | Data display smoke test |
| `<input>`, `<select>`, `<textarea>` with validation attributes | Form validation test |
| `@if (User.Identity.IsAuthenticated)` conditionals | Auth-state-dependent tests |
| `<partial name="">` or `<vc:component>` | Shared component → POM candidate |
| Layout file (`_Layout.cshtml` / `_Host.cshtml`) | Shared nav, header, footer → POM candidate |

**Do not attempt to generate exact DOM selectors from Razor/Blazor markup.** Server-rendered HTML may differ from the template. Instead, identify the *type* of interaction (form submit, link click, text display) and generate role-based selectors that match semantically. If MCP snapshots are available (Step 1f), use those for precise selectors.

#### 1d: Detect auth patterns

```powershell
# Check for ASP.NET Identity
$hasIdentity = Get-ChildItem -Path "<project>" -Recurse -Filter "*.cs" |
  ForEach-Object { Get-Content $_.FullName -Raw } |
  Where-Object { $_ -match 'AddDefaultIdentity|AddIdentity|UseAuthentication' }

# Check for [Authorize] attributes
$authRoutes = Get-ChildItem -Path "<project>" -Recurse -Filter "*.cs" |
  Select-String '\[Authorize' | ForEach-Object { $_.Filename } | Sort-Object -Unique

# Check for cookie auth
$hasCookieAuth = (Get-Content "<project>/Program.cs" -Raw) -match 'AddCookie|AddAuthentication.*Cookie'

# Check for JWT
$hasJwt = (Get-Content "<project>/Program.cs" -Raw) -match 'AddJwtBearer|JwtBearerDefaults'
```

If auth is detected, the skill must generate an auth fixture (see Step 3d). Classify routes as:
- **Public** — no `[Authorize]`, no auth check in page model
- **Authenticated** — requires login, any role
- **Role-restricted** — requires specific role(s)

#### 1e: Detect existing Playwright tests

```powershell
# Find existing Playwright test projects
Get-ChildItem -Path "<solution_dir>" -Recurse -Filter "*.csproj" |
  ForEach-Object {
    $content = Get-Content $_.FullName -Raw
    if ($content -match 'Microsoft\.Playwright') {
      @{ Project = $_.FullName; Content = $content }
    }
  }
```

If found:
- Read existing test files to understand what routes are already covered
- Identify the test framework (NUnit/MSTest/xUnit) and match it
- Generate only tests for *uncovered* routes (gap-fill mode)
- Place new tests in the existing project structure

If not found:
- Scaffold a new test project (Step 3)

#### 1f: Optional — MCP page discovery (if Playwright MCP is available)

If the Playwright MCP server is available AND the app is running (either at a provided URL or after auto-start in Step 4b):

```
For each discovered route (up to 10):
    1. browser_navigate → <base_url>/<route>
    2. browser_snapshot → capture aria tree
    3. Extract roles, names, and interactive elements
    4. Map to precise selectors for test generation
```

**This step is optional.** If MCP is not available, the skill generates tests from code analysis alone (Step 1c) using semantic selector strategies. Tests generated with MCP enrichment will have more precise selectors; tests without will use broader role-based selectors that are still correct but may be less specific.

**Rules for MCP discovery:**
- Cap at 10 pages to avoid excessive browser interaction
- Prioritize routes identified as user flow candidates over smoke-only routes
- Do not use MCP for test *execution* — tests run via `dotnet test`
- If MCP navigation fails for a page (auth wall, error), skip it and note in the report

---

### Step 2: Plan test generation

#### 2a: Classify routes into test categories

For each discovered route, assign a test category:

| Category | Criteria | Test depth |
|----------|----------|------------|
| **User Flow** | Has forms, multi-step interactions, state changes, or is in the user's focus area | Full interaction chain: navigate → fill → submit → verify result |
| **Navigation** | Links to other pages, breadcrumbs, menu items | Click → verify destination URL and key element |
| **Form Validation** | Has inputs with validation attributes (`[Required]`, `[StringLength]`, etc.) | Submit empty/invalid → verify error messages |
| **Smoke** | All other routes — verify the page loads and key elements are present | Navigate → verify heading, key elements visible |
| **Error Handling** | Error pages, 404 handlers, exception middleware | Navigate to invalid route → verify error page |

A route can have multiple test categories (e.g., a page with a form gets both a Smoke test and a User Flow test).

#### 2b: Identify POM candidates

Shared elements that appear across multiple pages should become Page Object Models:

| Pattern | POM class |
|---------|-----------|
| Layout nav bar / header | `NavBarComponent.cs` |
| Layout footer | `FooterComponent.cs` |
| Login/register forms | `LoginPage.cs` |
| Shared search bar | `SearchComponent.cs` |
| Data tables with paging | `DataTableComponent.cs` |

**POM generation rules:**
- One POM per page, one POM per shared component
- Each POM encapsulates locators as properties and interactions as methods
- POMs never contain assertions — assertions belong in tests
- POMs accept `IPage` in the constructor

#### 2c: Gap analysis (if existing tests found)

If Step 1e found existing tests, compare the route inventory against covered routes:

```
Route inventory:         [/Home, /Products, /Products/{id}, /Cart, /Checkout, /Login, /Admin]
Already tested routes:   [/Home, /Login]
Gap:                     [/Products, /Products/{id}, /Cart, /Checkout, /Admin]
```

Report the gap and generate tests only for uncovered routes. If the user specified focus areas, prioritize those within the gap.

---

### Step 3: Scaffold the test project

#### 3a: Create or detect existing test project

**If no existing Playwright test project (from Step 1e):**

```powershell
# Determine project name from app project
$appName = [System.IO.Path]::GetFileNameWithoutExtension("<app_csproj>")
$testProjectDir = Join-Path "<solution_dir>" "$appName.E2ETests"
$testProjectPath = Join-Path $testProjectDir "$appName.E2ETests.csproj"

# Detect TFM from the app project
$appCsproj = Get-Content "<app_csproj>" -Raw
if ($appCsproj -match '<TargetFramework>(net\d+\.\d+)</TargetFramework>') {
    $tfm = $Matches[1]
} else {
    $tfm = "net$(dotnet --version | ForEach-Object { $_.Split('.')[0] }).0"
}

# Create project
New-Item -ItemType Directory -Path $testProjectDir -Force
```

**If existing test project found:** use its directory and framework. Skip to Step 3c.

#### 3b: Generate the project file

Generate the `.csproj` file based on the detected framework.

**NUnit (default):**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{tfm}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
</Project>
```

**MSTest:**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{tfm}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
</Project>
```

**xUnit:**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{tfm}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
</Project>
```

After generating the `.csproj`, add NuGet packages using `dotnet add package` (which resolves the latest compatible version automatically) and add the project to the solution:

**NUnit:**
```powershell
dotnet add "$testProjectPath" package Microsoft.Playwright.NUnit
dotnet add "$testProjectPath" package Microsoft.NET.Test.Sdk
dotnet add "$testProjectPath" package NUnit
dotnet add "$testProjectPath" package NUnit3TestAdapter
```

**MSTest:**
```powershell
dotnet add "$testProjectPath" package Microsoft.Playwright.MSTest
dotnet add "$testProjectPath" package Microsoft.NET.Test.Sdk
dotnet add "$testProjectPath" package MSTest.TestAdapter
dotnet add "$testProjectPath" package MSTest.TestFramework
```

**xUnit:**
```powershell
dotnet add "$testProjectPath" package Microsoft.Playwright
dotnet add "$testProjectPath" package Microsoft.NET.Test.Sdk
dotnet add "$testProjectPath" package xunit
dotnet add "$testProjectPath" package xunit.runner.visualstudio
```

Then add the test project to the solution file and restore:

```powershell
# Add to solution so IDE and CI can discover it
$slnFile = Get-ChildItem -Path "<solution_dir>" -Filter "*.sln" | Select-Object -First 1
if ($slnFile) {
    dotnet sln $slnFile.FullName add "$testProjectPath"
}

dotnet restore "$testProjectPath"
```

#### 3c: Generate base infrastructure

**`.runsettings` file** (at test project root):

```xml
<?xml version="1.0" encoding="utf-8"?>
<RunSettings>
  <Playwright>
    <BrowserName>chromium</BrowserName>
    <LaunchOptions>
      <Headless>true</Headless>
    </LaunchOptions>
  </Playwright>
  <RunConfiguration>
    <EnvironmentVariables>
      <BASE_URL>{base_url}</BASE_URL>
    </EnvironmentVariables>
  </RunConfiguration>
</RunSettings>
```

**`GlobalUsings.cs`** (NUnit example):

```csharp
global using NUnit.Framework;
global using Microsoft.Playwright;
global using Microsoft.Playwright.NUnit;
```

**`GlobalUsings.cs`** (MSTest):

```csharp
global using Microsoft.VisualStudio.TestTools.UnitTesting;
global using Microsoft.Playwright;
global using Microsoft.Playwright.MSTest;
```

**`GlobalUsings.cs`** (xUnit):

```csharp
global using Xunit;
global using Microsoft.Playwright;
```

**`TestConstants.cs`** in `Infrastructure/`:

```csharp
namespace {AppName}.E2ETests.Infrastructure;

public static class TestConstants
{
    public static string BaseUrl =>
        Environment.GetEnvironmentVariable("BASE_URL") ?? "https://localhost:5001";
}
```

**`TestBase.cs`** in `Infrastructure/` — diagnostic instrumentation for screenshots and traces on failure:

**NUnit:**
```csharp
namespace {AppName}.E2ETests.Infrastructure;

/// <summary>
/// Base class for all E2E test fixtures. Provides automatic screenshot capture
/// and trace collection on test failure for diagnostic purposes.
/// </summary>
public class TestBase : PageTest
{
    [SetUp]
    public async Task StartTracing()
    {
        await Context.Tracing.StartAsync(new()
        {
            Screenshots = true,
            Snapshots = true,
            Sources = false
        });
    }

    [TearDown]
    public async Task CaptureOnFailure()
    {
        if (TestContext.CurrentContext.Result.Outcome.Status ==
            NUnit.Framework.Interfaces.TestStatus.Failed)
        {
            var testName = TestContext.CurrentContext.Test.Name;
            var artifactDir = Path.Combine(
                TestContext.CurrentContext.WorkDirectory, "test-artifacts");
            Directory.CreateDirectory(artifactDir);

            // Screenshot
            await Page.ScreenshotAsync(new()
            {
                Path = Path.Combine(artifactDir, $"{testName}.png"),
                FullPage = true
            });

            // Trace
            await Context.Tracing.StopAsync(new()
            {
                Path = Path.Combine(artifactDir, $"{testName}.zip")
            });
        }
        else
        {
            await Context.Tracing.StopAsync();
        }
    }
}
```

**MSTest:**
```csharp
namespace {AppName}.E2ETests.Infrastructure;

public class TestBase : PageTest
{
    public TestContext TestContext { get; set; } = null!;

    [TestInitialize]
    public async Task StartTracing()
    {
        await Context.Tracing.StartAsync(new()
        {
            Screenshots = true,
            Snapshots = true,
            Sources = false
        });
    }

    [TestCleanup]
    public async Task CaptureOnFailure()
    {
        if (TestContext.CurrentTestOutcome == UnitTestOutcome.Failed)
        {
            var testName = TestContext.TestName;
            var artifactDir = Path.Combine(
                Directory.GetCurrentDirectory(), "test-artifacts");
            Directory.CreateDirectory(artifactDir);

            await Page.ScreenshotAsync(new()
            {
                Path = Path.Combine(artifactDir, $"{testName}.png"),
                FullPage = true
            });

            await Context.Tracing.StopAsync(new()
            {
                Path = Path.Combine(artifactDir, $"{testName}.zip")
            });
        }
        else
        {
            await Context.Tracing.StopAsync();
        }
    }
}
```

**xUnit** (uses `IAsyncLifetime`):
```csharp
namespace {AppName}.E2ETests.Infrastructure;

/// <summary>
/// xUnit base class using IAsyncLifetime for Playwright lifecycle.
/// Subclass this and use this.Page for browser interactions.
/// </summary>
public class TestBase : IAsyncLifetime
{
    protected IPlaywright Playwright { get; private set; } = null!;
    protected IBrowser Browser { get; private set; } = null!;
    protected IBrowserContext Context { get; private set; } = null!;
    protected IPage Page { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        Playwright = await Microsoft.Playwright.Playwright.CreateAsync();
        Browser = await Playwright.Chromium.LaunchAsync();
        Context = await Browser.NewContextAsync();
        await Context.Tracing.StartAsync(new()
        {
            Screenshots = true,
            Snapshots = true,
            Sources = false
        });
        Page = await Context.NewPageAsync();
    }

    public async Task DisposeAsync()
    {
        await Context.Tracing.StopAsync();
        await Browser.CloseAsync();
        Playwright.Dispose();
    }

    /// <summary>
    /// Call this in a catch block or after a failed assertion to capture diagnostics.
    /// </summary>
    protected async Task CaptureFailureArtifactsAsync(string testName)
    {
        var artifactDir = Path.Combine(Directory.GetCurrentDirectory(), "test-artifacts");
        Directory.CreateDirectory(artifactDir);

        await Page.ScreenshotAsync(new()
        {
            Path = Path.Combine(artifactDir, $"{testName}.png"),
            FullPage = true
        });

        await Context.Tracing.StopAsync(new()
        {
            Path = Path.Combine(artifactDir, $"{testName}.zip")
        });
    }
}
```

**Important:** All generated test fixtures should inherit from `TestBase` instead of `PageTest` (NUnit/MSTest) or implement `IAsyncLifetime` (xUnit). This ensures diagnostic instrumentation is active for every test.

#### 3d: Generate auth fixture (if auth detected in Step 1d)

**NUnit auth fixture:**

```csharp
namespace {AppName}.E2ETests.Infrastructure;

/// <summary>
/// Creates and caches authenticated browser state.
/// Runs once before all tests that need auth, stores cookie state to a file
/// that individual tests load via BrowserNewContextOptions.StorageStatePath.
/// </summary>
public class AuthSetup
{
    private static readonly string StorageStatePath =
        Path.Combine(AppContext.BaseDirectory, ".auth-state.json");
    private static readonly SemaphoreSlim Lock = new(1, 1);

    public static string GetStorageStatePath() => StorageStatePath;

    /// <summary>
    /// Call once in a [OneTimeSetUp] to create the auth state file.
    /// Thread-safe — multiple parallel fixtures can call this safely.
    /// Navigates to the login page, fills credentials, submits, and saves state.
    /// </summary>
    public static async Task CreateAuthStateAsync(IPage page)
    {
        await Lock.WaitAsync();
        try
        {
            if (File.Exists(StorageStatePath))
                return; // Already created this run

            await page.GotoAsync($"{TestConstants.BaseUrl}/Identity/Account/Login");
            await page.WaitForLoadStateAsync(LoadState.DOMContentLoaded);

        // TODO: Update these selectors to match your app's login page
        await page.GetByLabel("Email").FillAsync("test@example.com");
        await page.GetByLabel("Password").FillAsync("Test123!");
        await page.GetByRole(AriaRole.Button, new() { Name = "Log in" }).ClickAsync();

        // Wait for redirect after successful login
        await page.WaitForURLAsync($"{TestConstants.BaseUrl}/**");

        // Save auth state
        await page.Context.StorageStateAsync(new()
        {
            Path = StorageStatePath
        });
        }
        finally
        {
            Lock.Release();
        }
    }
}
```

Generate a `TODO` comment in the auth fixture indicating the user must update the test credentials and selectors for their app. The skill should attempt to read the login page structure (from Razor view or MCP snapshot) to pre-fill selectors, but credentials must always be placeholder values.

If the app uses JWT instead of cookie auth, generate an API-based auth fixture that calls the token endpoint and stores the token for injection via `ExtraHTTPHeaders`.

#### 3e: Generate Page Object Models

For each POM candidate identified in Step 2b, generate a class following this pattern:

```csharp
namespace {AppName}.E2ETests.PageObjects;

/// <summary>
/// Page Object Model for the {PageName} page.
/// Route: {route}
/// </summary>
public class {PageName}Page
{
    private readonly IPage _page;

    public {PageName}Page(IPage page)
    {
        _page = page;
    }

    // --- Locators (properties) ---

    private ILocator PageHeading =>
        _page.GetByRole(AriaRole.Heading, new() { Name = "{page title}" });

    private ILocator {ElementName} =>
        _page.GetByRole(AriaRole.{Role}, new() { Name = "{accessible name}" });

    // --- Navigation ---

    public async Task NavigateAsync()
    {
        await _page.GotoAsync($"{TestConstants.BaseUrl}/{route}");
        await _page.WaitForLoadStateAsync(LoadState.DOMContentLoaded);
    }

    // --- Actions (methods) ---

    public async Task {ActionName}Async({parameters})
    {
        // Encapsulates a user interaction
    }

    public async Task FillFormAsync({parameter_list})
    {
        // Encapsulates filling a form — each field through a private locator
    }

    public async Task SubmitFormAsync()
    {
        // Encapsulates clicking the submit button
    }

    // --- Verification helpers (no assertions — return values for tests to assert) ---

    public Task<bool> IsLoadedAsync() =>
        PageHeading.IsVisibleAsync();
}
```

**POM rules:**
- Locators are private properties — tests access interactions through public methods
- Navigation is a public method — every POM has `NavigateAsync()`
- No assertions in POMs — return values or `Task<bool>` for tests to check
- Use `TestConstants.BaseUrl` for all URLs
- Constructor accepts `IPage` only

#### 3f: Generate test files

Generate test files for each route based on the test plan from Step 2a. Place in `Tests/`.

**See the Test Generation Patterns section below for complete templates per category.**

---

### Step 4: Execute

#### 4a: Install Playwright browsers

```powershell
$testProjectDir = "<path_to_test_project>"

# Build the test project to get the playwright.ps1 script
$buildOutput = dotnet build "$testProjectDir" 2>&1
if ($LASTEXITCODE -ne 0) {
    Write-Error "Test project build failed. Aborting test execution."
    Write-Error $buildOutput
    # Still generate the report — include build errors in the "Failed Tests" section
    # Skip Steps 4b–4c and go directly to Step 5 with build failure details
    return
}

# Find and run the Playwright install script
$pwScript = Get-ChildItem -Path "$testProjectDir" -Recurse -Filter "playwright.ps1" | Select-Object -First 1
if ($pwScript) {
    & $pwScript.FullName install chromium
    if ($LASTEXITCODE -ne 0) {
        Write-Error "Playwright browser install failed. Check network access and disk space."
        return
    }
} else {
    Write-Error "playwright.ps1 not found — cannot install browsers. Aborting."
    return
}
```

#### 4b: Start the application (if no URL was provided)

If the user did not provide a running URL, start the app:

```powershell
# Discover the app URL from launchSettings.json (prefer HTTPS)
$launchSettings = Join-Path (Split-Path "<app_csproj>") "Properties" "launchSettings.json"
$baseUrl = "https://localhost:5001"  # fallback default

if (Test-Path $launchSettings) {
    $settings = Get-Content $launchSettings -Raw | ConvertFrom-Json
    # Check profiles for applicationUrl
    $profiles = $settings.profiles.PSObject.Properties | Where-Object { $_.Value.applicationUrl }
    if ($profiles) {
        $urls = $profiles[0].Value.applicationUrl -split ';'
        $httpsUrl = $urls | Where-Object { $_ -match '^https://' } | Select-Object -First 1
        $baseUrl = if ($httpsUrl) { $httpsUrl.Trim() } else { $urls[0].Trim() }
    }
}

# Start in an async shell session — do NOT use Start-Process
dotnet run --project "<app_csproj>" --urls "$baseUrl"
```

**Critical:** Use an async shell session to start the app. Do NOT use `Start-Process` — the process will die between shell sessions. Remember the shell session ID for cleanup in Step 4d.

**App configuration prerequisites:** If the app requires connection strings, user secrets, or environment variables to start, these must be configured before running. Check for `appsettings.Development.json` and `dotnet user-secrets list`. If the app crashes on startup, capture stderr and report it — do not silently retry.

Wait for the app to become reachable:

```powershell
$maxRetries = 30
$retryCount = 0

while ($retryCount -lt $maxRetries) {
    try {
        $response = Invoke-WebRequest -Uri $baseUrl -UseBasicParsing -TimeoutSec 2 -SkipCertificateCheck
        if ($response.StatusCode -lt 400) { break }
    } catch { }
    $retryCount++
    Start-Sleep -Seconds 1
}

if ($retryCount -eq $maxRetries) {
    Write-Error "App did not start within 30 seconds at $baseUrl"
    Write-Error "Check: (1) connection strings configured, (2) port $baseUrl not in use, (3) dev cert trusted (run 'dotnet dev-certs https --trust')"
    # Include startup failure in the report and abort test execution
}
```

#### 4c: Run tests

```powershell
$trxPath = Join-Path $testProjectDir "TestResults" "results.trx"
$testOutput = dotnet test "$testProjectDir" `
    --logger "trx;LogFileName=results.trx" `
    --settings "$testProjectDir\.runsettings" `
    --no-build 2>&1

$testExitCode = $LASTEXITCODE
```

If tests fail, do not stop — collect all results for the report. Capture both stdout and stderr for failure analysis.

#### 4d: Cleanup

```powershell
# 1. If we started the app, stop it by terminating the async shell session
#    Use stop_powershell with the shell session ID from Step 4b

# 2. Clean up auth state temp file (if generated)
$authStatePath = Join-Path $testProjectDir ".auth-state.json"
if (Test-Path $authStatePath) {
    Remove-Item $authStatePath -Force
}

# 3. Parse TRX results
$passed = 0; $failed = 0; $skipped = 0
if (Test-Path $trxPath) {
    [xml]$trx = Get-Content $trxPath -Raw
    $counters = $trx.TestRun.ResultSummary.Counters
    $passed = [int]$counters.passed
    $failed = [int]$counters.failed
    $skipped = [int]$counters.total - $passed - $failed
}

# 4. Verify report was written (after Step 5)
```

---

### Step 5: Generate the report

Write a generation report to `<artifact_root>/reviews/`.

**Report naming:** `MMDDYYYY-E2ETestGen-<ProjectName>-Run<N>.md`

If a file with today's date already exists, increment the run number.

#### Report template

```markdown
# E2E Test Generation Report: {ProjectName}

**Generated:** {date}
**App type:** {Razor Pages / MVC / Blazor / Minimal API / Hybrid}
**Test framework:** {NUnit / MSTest / xUnit}
**Test project:** `{path to generated test project}`

---

## Application Surface

| Route | Type | Auth | Test Category | Status |
|-------|------|------|---------------|--------|
| /Home | Razor Page | Public | Smoke | ✅ Generated |
| /Products | MVC Controller | Public | Smoke + User Flow | ✅ Generated |
| /Cart | Razor Page | Authenticated | User Flow | ✅ Generated |
| /Admin | MVC Controller | Role: Admin | Not generated | ⚠️ Requires admin role |

**Routes discovered:** {N}
**Tests generated:** {N} across {N} files
**Page Object Models:** {N}

---

## Generated Files

### Page Object Models
| File | Covers |
|------|--------|
| `PageObjects/HomePage.cs` | Home page layout, nav, hero |
| `PageObjects/ProductListPage.cs` | Product grid, search, filters |

### Test Files
| File | Tests | Category |
|------|-------|----------|
| `Tests/HomeTests.cs` | 2 | Smoke |
| `Tests/ProductFlowTests.cs` | 4 | User Flow |
| `Tests/NavigationTests.cs` | 3 | Navigation |

### Infrastructure
| File | Purpose |
|------|---------|
| `Infrastructure/TestConstants.cs` | Base URL, shared config |
| `Infrastructure/AuthSetup.cs` | Login state caching |

---

## Test Results

| Test | Result | Duration |
|------|--------|----------|
| HomeTests.HomePage_Should_LoadSuccessfully | ✅ Passed | 1.2s |
| ProductFlowTests.SearchProducts_Should_FilterResults | ❌ Failed | 3.1s |

**Passed:** {N} | **Failed:** {N} | **Skipped:** {N} | **Total:** {N}

---

## Failed Tests

### {TestName}

**Error:** {error message}
**Likely cause:** {analysis — selector mismatch, auth wall, timing, missing data}
**Suggested fix:** {concrete action}

---

## Coverage Gaps

Routes that were discovered but not tested:

| Route | Reason |
|-------|--------|
| /Admin/Users | Requires admin role — no admin credentials available |
| /API/v1/products | API endpoint — generate API tests separately |

---

## Recommendations

{Prioritized list of next steps: fix failing tests, add auth credentials, extend to uncovered routes, run playwright-test-review for quality assessment}
```

---

## Test Generation Patterns

### Smoke Test (NUnit)

```csharp
namespace {AppName}.E2ETests.Tests;

[Parallelizable(ParallelScope.Self)]
[TestFixture]
public class {PageName}Tests : TestBase
{
    private {PageName}Page _{pageName}Page = null!;

    [SetUp]
    public void SetUp()
    {
        _{pageName}Page = new {PageName}Page(Page);
    }

    [Test]
    public async Task {PageName}_Should_LoadSuccessfully()
    {
        await _{pageName}Page.NavigateAsync();

        await Expect(Page).ToHaveTitleAsync(new Regex("{expected title pattern}"));
        await Expect(Page.GetByRole(AriaRole.Heading, new() { Name = "{heading}" }))
            .ToBeVisibleAsync();
    }
}
```

### User Flow Test (NUnit)

```csharp
[Test]
public async Task {FlowName}_Should_{ExpectedOutcome}()
{
    // Arrange — navigate to starting page
    var {startPage} = new {StartPage}Page(Page);
    await {startPage}.NavigateAsync();

    // Act — perform the user flow
    await {startPage}.{Action1}Async({params});

    // Navigate to result page if the flow transitions
    var {resultPage} = new {ResultPage}Page(Page);

    // Assert — verify the outcome
    await Expect(Page).ToHaveURLAsync(new Regex("{expected URL pattern}"));
    await Expect(Page.GetByRole(AriaRole.{Role}, new() { Name = "{element}" }))
        .ToBeVisibleAsync();
}
```

### Form Submission Test (NUnit)

```csharp
[Test]
public async Task {FormName}_Should_SubmitSuccessfully()
{
    var page = new {PageName}Page(Page);
    await page.NavigateAsync();

    // Fill form fields through POM methods
    await page.FillFormAsync("{valid value 1}", "{valid value 2}");

    // Submit through POM method
    await page.SubmitFormAsync();

    // Verify success — redirect or confirmation message
    await Expect(Page).ToHaveURLAsync(new Regex("{success URL pattern}"));
    await Expect(Page.GetByText("{success message}")).ToBeVisibleAsync();
    // Verify submitted data persisted (e.g., item appears in list)
    await Expect(Page.GetByRole(AriaRole.Cell, new() { Name = "{submitted value}" }))
        .ToBeVisibleAsync();
}

[Test]
public async Task {FormName}_Should_ShowValidationErrors_WhenEmpty()
{
    var page = new {PageName}Page(Page);
    await page.NavigateAsync();

    // Submit without filling — triggers validation
    await page.SubmitFormAsync();

    // Verify validation messages appear
    await Expect(Page.GetByText("{required field message}")).ToBeVisibleAsync();
    // Verify we stayed on the same page (no redirect)
    await Expect(Page).ToHaveURLAsync(new Regex("{current page pattern}"));
}
```

### Navigation Test (NUnit)

```csharp
[Test]
public async Task NavigateTo{PageName}_Should_ShowCorrectPage()
{
    // Start from a known page
    await Page.GotoAsync(TestConstants.BaseUrl);

    // Click navigation link
    await Page.GetByRole(AriaRole.Link, new() { Name = "{link text}" }).ClickAsync();

    // Verify destination
    await Expect(Page).ToHaveURLAsync(new Regex("{route pattern}"));
    await Expect(Page.GetByRole(AriaRole.Heading, new() { Name = "{heading}" }))
        .ToBeVisibleAsync();
}
```

### Authenticated Test (NUnit)

```csharp
[TestFixture]
public class {PageName}AuthTests : TestBase
{
    public override BrowserNewContextOptions ContextOptions()
    {
        return new BrowserNewContextOptions
        {
            StorageStatePath = AuthSetup.GetStorageStatePath()
        };
    }

    [OneTimeSetUp]
    public async Task GlobalSetup()
    {
        // Create auth state once for all tests in this fixture
        using var playwright = await Microsoft.Playwright.Playwright.CreateAsync();
        var browser = await playwright.Chromium.LaunchAsync();
        var context = await browser.NewContextAsync();
        var page = await context.NewPageAsync();
        await AuthSetup.CreateAuthStateAsync(page);
        await browser.CloseAsync();
    }

    [Test]
    public async Task {PageName}_Should_ShowAuthenticatedContent()
    {
        await Page.GotoAsync($"{TestConstants.BaseUrl}/{route}");

        await Expect(Page.GetByRole(AriaRole.Heading, new() { Name = "{heading}" }))
            .ToBeVisibleAsync();
        // Auth-specific content should be visible
        await Expect(Page.GetByText("{authenticated content}")).ToBeVisibleAsync();
    }
}
```

---

### Framework-Specific Variations

The templates above are NUnit. When generating MSTest or xUnit tests, apply these mappings:

| NUnit | MSTest | xUnit |
|-------|--------|-------|
| `[TestFixture]` | `[TestClass]` | *(none — class is plain)* |
| `[Test]` | `[TestMethod]` | `[Fact]` |
| `[SetUp]` | `[TestInitialize]` | Constructor or `IAsyncLifetime.InitializeAsync` |
| `[TearDown]` | `[TestCleanup]` | `IAsyncLifetime.DisposeAsync` |
| `[OneTimeSetUp]` | `[ClassInitialize]` (static) | `IClassFixture<T>` |
| `[Parallelizable(ParallelScope.Self)]` | *(parallel by default)* | *(parallel by default)* |
| `: TestBase` (inherits `PageTest`) | `: TestBase` (inherits `PageTest`) | `: TestBase` (inherits `IAsyncLifetime`) |
| `Expect(locator)` | `Expect(locator)` | `Expect(locator)` — same Playwright API |

**MSTest smoke test example:**
```csharp
namespace {AppName}.E2ETests.Tests;

[TestClass]
public class {PageName}Tests : TestBase
{
    private {PageName}Page _{pageName}Page = null!;

    [TestInitialize]
    public void SetUp()
    {
        _{pageName}Page = new {PageName}Page(Page);
    }

    [TestMethod]
    public async Task {PageName}_Should_LoadSuccessfully()
    {
        await _{pageName}Page.NavigateAsync();

        await Expect(Page).ToHaveTitleAsync(new Regex("{expected title pattern}"));
        await Expect(Page.GetByRole(AriaRole.Heading, new() { Name = "{heading}" }))
            .ToBeVisibleAsync();
    }
}
```

**xUnit smoke test example:**
```csharp
namespace {AppName}.E2ETests.Tests;

public class {PageName}Tests : TestBase
{
    [Fact]
    public async Task {PageName}_Should_LoadSuccessfully()
    {
        var page = new {PageName}Page(Page);
        await page.NavigateAsync();

        await Expect(Page).ToHaveTitleAsync(new Regex("{expected title pattern}"));
        await Expect(Page.GetByRole(AriaRole.Heading, new() { Name = "{heading}" }))
            .ToBeVisibleAsync();
    }
}
```

---

## POM Generation Patterns

### Standard Page POM

```csharp
namespace {AppName}.E2ETests.PageObjects;

public class {PageName}Page
{
    private readonly IPage _page;

    public {PageName}Page(IPage page) => _page = page;

    // --- Locators ---
    private ILocator Heading =>
        _page.GetByRole(AriaRole.Heading, new() { Name = "{title}" });

    // --- Navigation ---
    public async Task NavigateAsync()
    {
        await _page.GotoAsync($"{TestConstants.BaseUrl}/{route}");
        await _page.WaitForLoadStateAsync(LoadState.DOMContentLoaded);
    }

    // --- Actions ---
    public async Task {ClickAction}Async() =>
        await _page.GetByRole(AriaRole.{Role}, new() { Name = "{name}" }).ClickAsync();

    public async Task {FillAction}Async(string value) =>
        await _page.GetByLabel("{label}").FillAsync(value);

    // --- Queries ---
    public Task<bool> IsLoadedAsync() => Heading.IsVisibleAsync();
}
```

### Shared Component POM

```csharp
namespace {AppName}.E2ETests.PageObjects;

/// <summary>
/// Shared component that appears on multiple pages (e.g., nav bar).
/// Use by composing into page POMs or accessing directly from tests.
/// </summary>
public class {ComponentName}Component
{
    private readonly IPage _page;

    public {ComponentName}Component(IPage page) => _page = page;

    public async Task NavigateToAsync(string linkText) =>
        await _page.GetByRole(AriaRole.Navigation)
            .GetByRole(AriaRole.Link, new() { Name = linkText })
            .ClickAsync();

    public Task<bool> IsVisibleAsync() =>
        _page.GetByRole(AriaRole.Navigation).IsVisibleAsync();
}
```

---

## Print Summary

After generating the report, print a summary to the terminal:

```
╔══════════════════════════════════════════════════════════════╗
║  E2E Test Generation Complete: {ProjectName}                ║
╠══════════════════════════════════════════════════════════════╣
║  App type:       {Razor Pages / MVC / Blazor / Hybrid}      ║
║  Framework:      {NUnit / MSTest / xUnit}                    ║
║  Routes found:   {N}                                         ║
║  Tests generated:{N} ({N} user flows, {N} smoke, {N} other) ║
║  POMs generated: {N}                                         ║
║  Test results:   ✅ {N} passed  ❌ {N} failed  ⏭️ {N} skip  ║
║  Report:         {report_path}                               ║
║  Test project:   {test_project_path}                         ║
╚══════════════════════════════════════════════════════════════╝
```

If any tests failed, append:
```
⚠️ {N} tests failed — see report for failure analysis and suggested fixes.
💡 Run `playwright-test-review` on the generated tests for a full quality assessment.
```

---

## Troubleshooting

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| `dotnet build` fails on test project | Missing SDK or incompatible TFM | Verify `dotnet --list-sdks` includes the target TFM. Run `dotnet restore` manually. |
| `playwright.ps1 install` fails | Network blocked, disk full, or restricted environment | Try `npx playwright install chromium` if Node.js is available. Check proxy settings. |
| App won't start — port conflict | Another process on the same port | Check `launchSettings.json` for the configured port. Use `netstat -ano | findstr :<port>` to find conflicts. |
| App won't start — missing config | Connection strings, user secrets, or env vars not set | Run `dotnet user-secrets list` to check. Copy `appsettings.Development.json` from a working environment. |
| SSL certificate error in tests | Dev cert not trusted | Run `dotnet dev-certs https --trust`. If using `.runsettings`, ensure `IgnoreHTTPSErrors` is set. |
| Auth tests fail — redirect to login | Auth state file not created or expired | Check that `AuthSetup.CreateAuthStateAsync` ran successfully. Verify placeholder credentials in `AuthSetup.cs` have been replaced with real test credentials. |
| Tests time out | App is slow to respond or Playwright wait exceeded | Increase timeout in `.runsettings`. Check if the app needs warm-up time. Add `Page.WaitForLoadStateAsync(LoadState.NetworkIdle)` for slow pages. |
| Blazor WASM tests fail | WebAssembly not loaded before interaction | Use `LoadState.NetworkIdle` instead of `DOMContentLoaded`. Increase navigation timeout. |
| Tests pass locally but fail in CI | Headed mode, missing browsers, or env differences | Ensure `.runsettings` has `<Headless>true</Headless>`. Run `playwright.ps1 install` in CI pipeline. |

---

## Output Format

### Generation report

- **File name:** `MMDDYYYY-E2ETestGen-<ProjectName>-Run<N>.md`
- **Location:** `<artifact_root>/reviews/`
- **Content:** See the report template in Step 5

### Generated test project structure

```
<AppName>.E2ETests/
├── <AppName>.E2ETests.csproj
├── .runsettings
├── GlobalUsings.cs
├── Infrastructure/
│   ├── TestConstants.cs
│   └── AuthSetup.cs          (only if auth detected)
├── PageObjects/
│   ├── HomePage.cs
│   ├── LoginPage.cs
│   ├── NavBarComponent.cs
│   └── ...
└── Tests/
    ├── HomeTests.cs
    ├── LoginFlowTests.cs
    ├── NavigationTests.cs
    └── ...
```
