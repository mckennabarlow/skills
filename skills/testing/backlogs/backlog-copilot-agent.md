# GitHub Copilot Agent — Issue Backlog

> **Auto-generated** from 12 reports on 02/12/2026
> **Re-run with:** `extract issues from <report folder paths>`

## Summary

- **Total unique issues:** 15
- **HIGH ROI:** 4 | **MEDIUM ROI:** 6 | **LOW ROI:** 5
- **Reports analyzed:**
  - Testing Agent Run 1 — netlandingpage (02/10/2026 - 174632)
  - Testing Agent Run 2 — netlandingpage (02/10/2026 - 210741)
  - Copilot Run 1 — netlandingpage (02/11/2026 - 085347)
  - Comparison — netlandingpage (02/11/2026)
  - Testing Agent Run 1 — eShop (02/11/2026)
  - Copilot Run 1 — eShop (02/11/2026)
  - Comparison — eShop (02/11/2026)
  - Testing Agent Run 1 — ContosoUniversity/copilot (02/12/2026)
  - Testing Agent Run 2 — ContosoUniversity/testingagent2 (02/12/2026)
  - Copilot Run 1 — ContosoUniversity/copilot2 (02/12/2026)
  - Comparison — ContosoUniversity (02/12/2026)

---

## Pre-Run Checklist

> **Run this before launching Copilot Agent Mode for test generation.** Many HIGH-ROI issues in this backlog could be prevented or mitigated by checking constraints upfront. This checklist is derived directly from issues observed across all analyzed runs.

### 🔍 Analyze Constraints

| Check | What to look for | Related issue |
|-------|-----------------|---------------|
| **Central Package Management** | Check if the repo has `Directory.Packages.props` or `ManagePackageVersionsCentrally` in `Directory.Build.props`. If CPM is enabled, Copilot will likely add packages with inline `Version` attributes that cause NU1008. You may need to manually fix the `.csproj` and `Directory.Packages.props` after the run. | Issue 2 |
| **Sealed/static dependencies** | Scan constructor parameters of target classes for sealed or static types (e.g., `AzureBlobClientManager`, `HttpClient`, Azure SDK clients). Copilot hasn't hit this yet but would fail on services that depend on sealed types. | Issue 3 |
| **Existing test project health** | Run `dotnet build` and `dotnet test` on the existing test project. If existing tests are failing (e.g., missing static files), fix those first — a broken baseline confuses Copilot's test discovery. | Issue 7 |
| **Test discovery state** | Close and reopen VS, or run `dotnet test` from the terminal once, to ensure test discovery cache is fresh. Stale caches cause new tests to show `[None]` status. | Issue 6 |
| **File count vs. prompt scope** | Copilot addressed only 3 of 10 open files in the netlandingpage run. If you need tests for many files, consider running separate prompts per file or per 2-3 files rather than "write tests for all open files." | Issue 5 |
| **Mocking framework availability** | Check if the test project already has a mocking framework (Moq, NSubstitute). If not, and CPM is enabled, you may need to pre-add the package yourself to avoid NU1008. | Issue 2 |

### 📋 Define the Test Plan

Before running, decide:

- **Which files to target** — Be explicit in your prompt: name the specific classes or files. Avoid vague prompts like "write tests for the open files" with many files open.
- **What must be covered** — Identify critical code paths: error handling, branching logic, async methods, edge cases. Include these in your prompt (e.g., "make sure to test the error path when `GetBlobAsync` throws").
- **What patterns to use** — Specify in your prompt: mocking framework (Moq vs. NSubstitute), test framework (MSTest vs. xUnit), and whether to test async methods with `async Task` patterns.
- **What to avoid** — Mention constraints in your prompt: "Don't test private methods via reflection", "Don't add trivial constructor-not-null tests", "Use the existing test project at `path/to/Tests.csproj`."

### 🎯 Define the Quality Bar

Set expectations before reviewing the output:

| Dimension | Minimum bar | How to check |
|-----------|------------|--------------|
| **Build validation** | All generated tests must compile (`dotnet build` succeeds) | Run `dotnet build` on the test project after Copilot finishes |
| **Test execution** | All generated tests must be runnable (`dotnet test` succeeds) | Run `dotnet test --filter "FullyQualifiedName~TestClassName"` |
| **Behavioral %** | ≥70% of tests should be Behavioral (not Trivial/Redundant) | Review the test classification table in the review report |
| **Async coverage** | Async service methods should have `async Task` test methods with `await` | Check test method signatures — all should be `async Task`, not `void` |
| **No reflection-based testing** | Private methods should be tested through the public API | Search for `GetMethod` or `BindingFlags` in generated files |
| **Negative assertions** | At least 1 `Times.Never` or `VerifyNoOtherCalls()` per file with conditional logic | Search generated files for negative mock patterns |
| **All files addressed** | Every file mentioned in the prompt should have tests or a documented reason for skipping | Compare your target list against the generated test files |

---

## HIGH ROI

### Issue 1 (ROI: HIGH) (🔺 2): Validate generated tests via `dotnet build` + `dotnet test`

- **Problem:** Copilot generated tests and reported success without running `dotnet build` or `dotnet test`. In the netlandingpage run, tests couldn't compile (NU1008). In the eShop run, tests happened to pass but were unverified.
- **Fix location:** Copilot Agent orchestrator
- **Suggested fix:** After editing test files, run `dotnet build` on the test project. If build succeeds, run `dotnet test --filter` to confirm new tests pass. Report results in the summary.
- **Why ROI:** Build validation is the single most important step — the difference between "generated code" and "verified tests."
- **Observed in:** Copilot netlandingpage, Comparison netlandingpage, Copilot eShop, Comparison eShop (4 of 7 reports)
- **Recurrence:** 2 of 2 Copilot runs — universal gap

### Issue 2 (ROI: HIGH) (🔺 1): Handle Central Package Management when adding NuGet packages

- **Problem:** Copilot added `<PackageReference Include="Moq" Version="4.20.72" />` inline, but the repo uses CPM via `Directory.Packages.props`. This caused NU1008 and prevented all tests from compiling.
- **Fix location:** Copilot Agent orchestrator + NuGet/MSBuild tooling
- **Suggested fix:** Before adding a NuGet package, check for `ManagePackageVersionsCentrally` in `Directory.Build.props` or `Directory.Packages.props` in parent directories. If CPM is enabled, add the version to `Directory.Packages.props` and use a version-less `PackageReference`.
- **Why ROI:** CPM is increasingly standard in enterprise .NET repos — this will block every Copilot test generation run that adds a new package.
- **Observed in:** Copilot netlandingpage, Comparison netlandingpage (2 of 7 reports)
- **Recurrence:** 1 of 2 Copilot runs (netlandingpage; eShop already had NSubstitute)

### Issue 3 (ROI: HIGH) (🔺 0): Detect sealed classes before generating mock-dependent tests

- **Problem:** `AzureBlobClientManager` is sealed. The Testing Agent wasted 1h45m trying to mock it. Copilot avoided this issue only because it picked simpler files — it would hit the same wall on DotnetReleaseService.
- **Fix location:** Roslyn analyzers + Copilot Agent orchestrator + LLM prompt/model
- **Suggested fix:** Before generating tests, scan constructor dependencies for sealed/static types. Either (1) mock via interface, (2) wrap in adapter, or (3) skip with clear explanation.
- **Why ROI:** Sealed Azure SDK types (BlobClient, HttpClient, etc.) are among the most common test generation blockers across .NET projects.
- **Observed in:** Comparison — netlandingpage (1 of 12 reports)
- **Recurrence:** Not yet hit directly in Copilot runs, but identified as a latent risk

### Issue 4 (ROI: HIGH) (🔺 1): `create_file` resolves paths incorrectly in multi-project solutions

- **Problem:** Copilot's `create_file(filePath: ContosoUniversity.Tests\Services\NotificationServiceTests.cs)` created the file at `ContosoUniversity\ContosoUniversity.Tests\` (inside the main project) instead of the solution-sibling `ContosoUniversity.Tests\`. This caused 45 CS0246 errors and 4 failed builds. The agent spent ~2 minutes (half the session) on path resolution. Also left a 0-byte duplicate file behind.
- **Fix location:** Copilot Agent orchestrator
- **Suggested fix:** `create_file` should resolve relative paths from the `.csproj` that will compile the file, not from the VS workspace root. When creating a file in a test project, verify the target path is within the test project's directory tree. Clean up any 0-byte duplicates.
- **Why ROI:** Path confusion wastes half the generation session and occurs whenever the workspace root differs from the solution root — a very common multi-project layout.
- **Observed in:** Copilot Run 1 — ContosoUniversity/copilot2, Comparison — ContosoUniversity (2 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs (ContosoUniversity)

---

## MEDIUM ROI

### Issue 5 (ROI: MEDIUM) (🔺 8): No negative mock verification in any test file

- **Problem:** Both tools generate `.Received(1)` but neither generates `DidNotReceive()` or `.Received(0)`. Guard clause tests should verify the repository is NOT called when authentication fails.
- **Fix location:** LLM prompt/model
- **Suggested fix:** For guard-clause tests, add negative verification on the dependency that should NOT have been invoked.
- **Why ROI:** Negative assertions catch regressions where guard clauses are removed but tests still pass.
- **Observed in:** All 12 reports (9 explicit mentions)
- **Recurrence:** 3 of 3 Copilot runs + 5 of 5 Testing Agent runs — universal gap

### Issue 6 (ROI: MEDIUM) (🔺 1): Report which files were NOT addressed

- **Problem:** Copilot skipped 7 of 10 files in the netlandingpage run. It did not report which files were skipped or why.
- **Fix location:** Copilot Agent orchestrator
- **Suggested fix:** At completion, emit a summary table listing each requested file and its status (tested, skipped with reason, or failed).
- **Why ROI:** Transparent reporting prevents user confusion and allows informed follow-up.
- **Observed in:** Copilot netlandingpage, Comparison netlandingpage (2 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs (not applicable to single-file runs)

### Issue 7 (ROI: MEDIUM) (🔺 1): Test discovery caching prevents execution of new tests

- **Problem:** After generating tests, Copilot discovered 69 tests (42 + 27) but the 27 new tests showed `[None]` status. Copilot spent ~6 minutes trying different test runner approaches without success.
- **Fix location:** VS test runner + Copilot Agent orchestrator
- **Suggested fix:** Instead of using the VS test runner, fall back to `dotnet test --filter "FullyQualifiedName~TestClassName"` via the terminal tool to bypass discovery caching.
- **Why ROI:** Test runner caching issues are common in VS; having a terminal fallback would reliably validate generated tests.
- **Observed in:** Copilot netlandingpage, Comparison netlandingpage (2 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs (not observed in eShop or ContosoUniversity)

### Issue 8 (ROI: MEDIUM) (🔺 1): Pre-existing test failures not diagnosed

- **Problem:** The existing 42 netlandingpage tests were all failing due to a missing `fluent-icons.svg` file. Copilot identified this during investigation but did not fix it.
- **Fix location:** Copilot Agent orchestrator
- **Suggested fix:** When existing tests are failing, either (1) fix the root cause if it's simple, or (2) explicitly note the pre-existing failures in the summary so the user knows the test suite was already broken.
- **Why ROI:** A broken test baseline obscures whether new tests pass, reducing trust in the entire test suite.
- **Observed in:** Copilot netlandingpage (1 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs

### Issue 9 (ROI: MEDIUM) (🔺 3): Measure code coverage after test generation

- **Problem:** The Testing Agent measured coverage (0% → 82.7%) in both ContosoUniversity runs. Copilot has no coverage measurement capability, so the user cannot quantify the value of generated tests without manual tooling.
- **Fix location:** Copilot Agent orchestrator
- **Suggested fix:** After validating tests pass, run `dotnet test --collect "XPlat Code Coverage"` and report the coverage delta in the summary.
- **Why ROI:** Coverage metrics are essential for evaluating test quality and justifying time spent on generation.
- **Observed in:** Comparison — eShop, Comparison — ContosoUniversity, Copilot — ContosoUniversity (3 of 12 reports)
- **Recurrence:** Structural limitation — affects all 3 Copilot runs

### Issue 10 (ROI: MEDIUM) (🔺 1): Trivial no-op tests add limited value

- **Problem:** Copilot generated 3 trivial tests for ContosoUniversity (1 × MarkAsRead, 2 × Dispose) — 14.3% of total. While better than the Testing Agent's 29.2%, these still test empty method bodies that cannot fail.
- **Fix location:** LLM prompt/model
- **Suggested fix:** When the agent detects a method body is empty or contains only comments, generate at most 1 "does not throw" test per method. Flag in comments that the implementation is a no-op.
- **Why ROI:** Reduces test noise, though impact is limited since Copilot already generates fewer trivials than the Testing Agent.
- **Observed in:** Copilot Run 1 — ContosoUniversity, Comparison — ContosoUniversity (2 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs

---

## LOW ROI

### Issue 11 (ROI: LOW) (🔺 1): Neither tool tests exception propagation from repository

- **Problem:** `DeleteBasket` and `UpdateBasket` call repository methods without try/catch. Neither tool generated a test verifying repository exceptions propagate correctly.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Add a test where the dependency throws and verify the exception propagates unchanged.
- **Why ROI:** Minor — documents expected behavior, useful for regression safety.
- **Observed in:** Testing Agent eShop, Copilot eShop, Comparison eShop (3 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs (eShop)

### Issue 12 (ROI: LOW) (🔺 1): Only synchronous tests generated despite async service methods

- **Problem:** All 27 Copilot-generated netlandingpage tests are synchronous. The target services have async methods (`GetBlogPostsAsync`, `GetTutorial`, `GetEBooksAsync`).
- **Fix location:** LLM prompt/model
- **Suggested fix:** When the target service has `async Task<T>` methods, generate `async Task` test methods with proper `await` patterns.
- **Why ROI:** Minor — affects test design quality but the tests themselves are all behavioral.
- **Observed in:** Copilot netlandingpage, Comparison netlandingpage (2 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs (not observed in eShop or ContosoUniversity — no async methods)

### Issue 13 (ROI: LOW) (🔺 1): Reflection-based private method testing

- **Problem:** `BannerAutomationServiceTests` uses reflection to test the private `GetRegionLink` method. Creates brittle tests coupled to implementation details.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Test through the public API surface instead, or suggest extracting the private method into a testable helper.
- **Why ROI:** Minor — only 4 tests affected, but sets a pattern for fragile private method testing.
- **Observed in:** Copilot netlandingpage (1 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs

### Issue 14 (ROI: LOW) (🔺 1): Neither tool tests CancellationToken handling

- **Problem:** `ServerCallContext` includes a `CancellationToken` property. Neither tool generated tests for cancellation scenarios.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Add a test that sets `CancellationToken` to a cancelled token and verifies the service handles it appropriately (either throws `OperationCanceledException` or passes it to the repository).
- **Why ROI:** Minor — cancellation handling is important for production resilience but rarely causes test failures.
- **Observed in:** Comparison — eShop (1 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs (eShop)

### Issue 15 (ROI: LOW) (🔺 1): No null parameter guard tests

- **Problem:** No Copilot test passes null for `entityType` or `entityId` to `SendNotification`. The source doesn't have explicit null guards, so passing null would create a `Notification` with null `EntityType`/`EntityId` — a valid (and possibly unexpected) behavior worth documenting in tests.
- **Fix location:** LLM prompt/model
- **Suggested fix:** For every public method parameter that is a reference type, generate at least one test with null input — either verifying graceful handling or documenting the unguarded behavior.
- **Why ROI:** Null parameter tests are low-effort, high-documentation-value, but impact is limited for this specific service.
- **Observed in:** Copilot Run 1 — ContosoUniversity (1 of 12 reports)
- **Recurrence:** 1 of 3 Copilot runs
