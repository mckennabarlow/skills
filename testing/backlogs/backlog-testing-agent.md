# .NET Testing Agent — Issue Backlog

> **Auto-generated** from 12 reports on 02/12/2026
> **Re-run with:** `extract issues from <report folder paths>`

## Summary

- **Total unique issues:** 22
- **HIGH ROI:** 7 | **MEDIUM ROI:** 10 | **LOW ROI:** 5
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

> **Run this before launching the Testing Agent.** Many HIGH-ROI issues in this backlog could be prevented or mitigated by checking constraints upfront. This checklist is derived directly from issues observed across all analyzed runs.

### 🔍 Analyze Constraints

| Check | What to look for | Related issue |
|-------|-----------------|---------------|
| **Sealed/static dependencies** | Scan constructor parameters of target classes for sealed or static types (e.g., `AzureBlobClientManager`, `HttpClient`, Azure SDK clients). These cannot be mocked with Moq/NSubstitute. | Issue 1 |
| **Missing interfaces** | If a sealed dependency exists, check whether it implements an interface that can be mocked instead. If not, consider wrapping it behind an adapter before running. | Issue 1 |
| **Existing test project** | Verify the test project builds cleanly (`dotnet build`) and existing tests pass before running the agent. Pre-existing failures contaminate results. | Issue 12 |
| **NuGet package management** | Check if the repo uses Central Package Management (`Directory.Packages.props`). If so, note that any new packages the agent adds may need version entries in that file. | — |
| **File count vs. time budget** | The agent averages 8–15 minutes per file. If you have 10+ files, expect 1.5–2+ hours. Consider batching files across multiple runs. | Issue 4 |
| **Existing coverage** | Run `dotnet test --collect "XPlat Code Coverage"` to get a baseline. Files with 90%+ coverage may not need agent-generated tests — prioritize 0% or low-coverage files. | Issue 4, 8 |
| **No-op / stub methods** | Scan the target class for empty method bodies (e.g., `MarkAsRead`, `Dispose` with only comments, no-op interface implementations). Expect ~1 trivial test per stub method; if the agent produces 3–4 variants per stub, flag in review. | Issue 7 |
| **Target needs mocking?** | Check whether the target class has constructor-injected interfaces. If it has no DI (e.g., `NotificationService` with only `new()` constructor), the agent will still add Moq/NSubstitute as a dependency — this is wasted overhead and the mock framework will go unused. | Issue 17 |

### 📋 Define the Test Plan

Before running, decide:

- **Which files to target** — List the specific source files you want tests for. Avoid "all open files" if you have >5 files; batch them.
- **What must be covered** — Identify the critical code paths: error handling, branching logic, async methods, edge cases. Note these so you can check them against the output.
- **What patterns to use** — Decide on mocking framework (Moq vs. NSubstitute), test framework (MSTest vs. xUnit vs. NUnit), and whether to use `InMemoryDatabase` for EF Core tests.
- **What to avoid** — Flag sealed classes, static helpers, and private methods that shouldn't be tested directly. Note any dependencies that require special setup (e.g., Azure connections, file system access).

### 🎯 Define the Quality Bar

Set expectations before reviewing the output:

| Dimension | Minimum bar | How to check |
|-----------|------------|--------------|
| **Behavioral %** | ≥70% of tests should be Behavioral (not Trivial/Redundant) | Review the test classification table in the review report |
| **No trivial constructor tests** | Constructor-only tests should be absent unless constructor has branching logic | Search for `Constructor_With` in generated files |
| **No trivial debug-logging tests** | Logger.IsEnabled tests should be absent unless logging has side effects | Search for `LogsDebugMessage` in generated files |
| **Negative assertions** | At least 1 `DidNotReceive()` or `Times.Never` per file with conditional logic | Search generated files for negative mock patterns |
| **Async coverage** | Async service methods should have `async Task` test methods | Check test method signatures |
| **No deleted behavioral tests** | If the agent deletes tests, check whether they tested meaningful logic | Review the "Deleted Tests" section in the review report |
| **All files addressed** | Every requested file should have tests or a documented skip reason | Compare your target list against the output |

---

## HIGH ROI

### Issue 1 (ROI: HIGH) (🔺 2): Detect sealed classes before generating mock-dependent tests

- **Problem:** `AzureBlobClientManager` is sealed. The Testing Agent wasted 1h45m trying to mock it. Copilot would hit the same wall on DotnetReleaseService.
- **Fix location:** Roslyn analyzers + Testing Agent orchestrator + LLM prompt/model
- **Suggested fix:** Before generating tests, scan constructor dependencies for sealed/static types. Either (1) mock via interface, (2) wrap in adapter, or (3) skip with clear explanation.
- **Why ROI:** Sealed Azure SDK types (BlobClient, HttpClient, etc.) are among the most common test generation blockers across .NET projects.
- **Observed in:** Testing Agent Run 1, Testing Agent Run 2, Comparison — netlandingpage (3 of 7 reports)
- **Recurrence:** 2 of 2 netlandingpage Testing Agent runs; not hit in eShop (interface dependencies)

### Issue 2 (ROI: HIGH) (🔺 1): Fix loop stall detection — agent spins indefinitely when failures share same root cause

- **Problem:** The Testing Agent ran 22 fix iterations, with the last 4 making zero progress (28→28→28→28). The agent consumed ~52 minutes of LLM compute on iterations 12–22 without reducing the failure count below 28.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** Implement a stall detector: if 3 consecutive iterations produce no net reduction in failing tests, group remaining failures by root cause, report to user, and stop.
- **Why ROI:** This is the single largest time waste observed — 52 minutes of invisible compute with no progress.
- **Observed in:** Testing Agent Run 2, Comparison — netlandingpage (2 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs (Run 2 netlandingpage)

### Issue 3 (ROI: HIGH) (🔺 1): Missing using directives cause CS0246 compilation failures

- **Problem:** EBookServiceTests.cs uses `Dictionary<,>` without `using System.Collections.Generic;` and DomainServiceTests.cs uses `InvalidOperationException` without `using System;`. These are CS0246 errors that prevented all tests from compiling.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** Auto-include common using directives (`System`, `System.Collections.Generic`, `System.Linq`, `System.Threading.Tasks`) in all generated test files, or detect missing usings from CS0246 errors and add them during fix iterations.
- **Why ROI:** Missing standard using directives is a trivially fixable error that blocks entire test runs.
- **Observed in:** Testing Agent Run 1 — netlandingpage (1 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs

### Issue 4 (ROI: HIGH) (🔺 1): Run duration exceeded 84 minutes — only 8 of 10 files processed

- **Problem:** The agent ran for 1 hour 24 minutes and was still generating tests for DotnetReleaseService when cancelled. BackgroundTimerService and CarouselAutomationService were never processed.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** Implement progress estimation and prioritize files by coverage gap (0% coverage files first). Files with 92%+ coverage should be deprioritized or skipped with a note.
- **Why ROI:** Extremely long runs without visible progress cause users to cancel, losing all work in progress.
- **Observed in:** Testing Agent Run 1 — netlandingpage (1 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs

### Issue 5 (ROI: HIGH) (🔺 1): No user-visible progress during fix iterations

- **Problem:** Each fix iteration takes 3–6 minutes with zero user-facing output. During iterations 12–22, the agent appeared completely frozen for 52 minutes.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** Emit a progress message at the start of each fix iteration with current failure count. Also emit when a test is removed.
- **Why ROI:** User confidence and informed cancellation decisions require visibility into what the agent is doing.
- **Observed in:** Testing Agent Run 2 — netlandingpage (1 of 12 reports)
- **Recurrence:** 1 of 5 Testing Agent runs

### Issue 6 (ROI: HIGH) (🔺 2): Custom instruction files in `.github/instructions/` not discovered

- **Problem:** Both ContosoUniversity Testing Agent runs logged "No custom instruction files found" despite `.github/instructions/pre-run-testingagent-tests.instructions.md` existing in one repo. The agent's `Discovering custom instruction files` step checks the repository root but does not find the `.github/instructions/` folder.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** The `Discovering custom instruction files` step should recursively search `.github/instructions/` from the repository root and load files matching `*.instructions.md` with `applyTo` frontmatter. These files contain test scenarios, mock setup guidance, and quality rules that would significantly improve generation quality.
- **Why ROI:** Instruction files are the primary mechanism for users to pre-configure test generation — if silently ignored, the entire pre-run guidance pipeline is bypassed.
- **Observed in:** TA Run 1 — ContosoUniversity/copilot, TA Run 2 — ContosoUniversity/testingagent2, Comparison — ContosoUniversity (3 of 12 reports)
- **Recurrence:** 2 of 5 Testing Agent runs (both ContosoUniversity runs)

### Issue 7 (ROI: HIGH) (🔺 4): Excessive trivial tests generated for no-op methods

- **Problem:** The Testing Agent generated 9 trivial tests for ContosoUniversity's no-op methods: 3 MarkAsRead variants (positive ID, zero, negative) and 4 Dispose variants (single, multiple, after-send, using-statement) plus 1 constructor-not-null. These test empty method bodies that cannot fail. In the testingagent2 run, 7 of 24 tests (29.2%) were trivial.
- **Fix location:** LLM prompt/model + Testing Agent orchestrator
- **Suggested fix:** When the agent detects a method body is empty or contains only comments/no-ops, generate at most 1 "does not throw" test per method. Do not generate parameter variants for no-op methods. Consolidates prior Issues 16/17 (constructor-not-null, constant-value assertions).
- **Why ROI:** Reduces trivial test inflation across all projects — no-op IDisposable and stub methods are extremely common in C# codebases.
- **Observed in:** TA Run 1 — ContosoUniversity/copilot, TA Run 2 — ContosoUniversity/testingagent2, Comparison — ContosoUniversity, TA Run 1 — netlandingpage, TA Run 2 — netlandingpage (5 of 12 reports)
- **Recurrence:** 4 of 5 Testing Agent runs

---

## MEDIUM ROI

### Issue 6 (ROI: MEDIUM) (🔺 5): No negative mock verification in any test file

- **Problem:** Both tools generate `.Received(1)` but neither generates `DidNotReceive()` or `.Received(0)`. Guard clause tests should verify the repository is NOT called when authentication fails.
- **Fix location:** LLM prompt/model
- **Suggested fix:** For guard-clause tests, add negative verification on the dependency that should NOT have been invoked.
- **Why ROI:** Negative assertions catch regressions where guard clauses are removed but tests still pass.
- **Observed in:** All 7 reports (6 explicit mentions)
- **Recurrence:** 3 of 3 Testing Agent runs + 2 of 2 Copilot runs — universal gap

### Issue 7 (ROI: MEDIUM) (🔺 2): Non-test method included in generated test files

- **Problem:** Generated test file includes a production method signature (`Write`) that is not a test.
- **Fix location:** LLM prompt/model + Testing Agent orchestrator
- **Suggested fix:** Validate that all generated methods have test attributes (`[Fact]`, `[Theory]`, `[Test]`, `[TestMethod]`). Strip non-test methods before writing final file.
- **Why ROI:** Non-test methods in test files cause compilation errors when they reference production types.
- **Observed in:** Testing Agent Run 1, Testing Agent Run 2 — netlandingpage (2 of 7 reports)
- **Recurrence:** 2 of 3 Testing Agent runs

### Issue 8 (ROI: MEDIUM) (🔺 1): Over-generation — 274 tests generated, 180 kept, 58 passing

- **Problem:** DotnetReleaseService received 4 batches totaling 240 tests. Final file has 58. Agent generated 4× more tests than it kept.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** After batch 1, check coverage delta before generating batch 2. If batch 1 covers most methods, skip or reduce subsequent batches.
- **Why ROI:** Reducing over-generation proportionally reduces fix-loop time and LLM token spend.
- **Observed in:** Testing Agent Run 2, Comparison — netlandingpage (2 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs

### Issue 9 (ROI: MEDIUM) (🔺 1): Report which files were NOT addressed

- **Problem:** The Testing Agent skipped 7 of 11 files. Neither tool reported which files were skipped or why.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** At completion, emit a summary table listing each requested file and its status (tested, skipped with reason, or failed).
- **Why ROI:** Transparent reporting prevents user confusion and allows informed follow-up.
- **Observed in:** Testing Agent Run 2, Comparison — netlandingpage (2 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs (not applicable to single-file eShop runs)

### Issue 10 (ROI: MEDIUM) (🔺 1): Preserve deleted test methods as commented-out code

- **Problem:** 11 behavioral tests were permanently deleted during fix iterations. All shared the same root cause (sealed class) and could have been retained as commented code.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** When deleting a test due to a shared root cause, preserve the test body as a comment with a `// BLOCKED: sealed class AzureBlobClientManager` annotation.
- **Why ROI:** Preserving test intent lets users manually fix blocked tests rather than losing the generated logic entirely.
- **Observed in:** Testing Agent Run 2, Comparison — netlandingpage (2 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs

### Issue 11 (ROI: MEDIUM) (🔺 1): Trivial debug-logging tests reduce behavioral percentage

- **Problem:** 2 of 11 eShop tests verify `logger.IsEnabled(LogLevel.Debug)` conditional path. These test infrastructure behavior, not business logic, reducing behavioral percentage to 81.8%.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Skip debug-logging path tests unless the logging performs side effects beyond the log call itself. Replace with behavioral tests like field mapping verification.
- **Why ROI:** Trivial tests dilute test quality metrics and take budget away from meaningful scenarios.
- **Observed in:** Testing Agent eShop Run 1, Comparison — eShop (2 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs (eShop)

### Issue 12 (ROI: MEDIUM) (🔺 1): All 42 tests fail at final run with no successful fix cycle

- **Problem:** The final test run shows 42 Tests (0 Passed, 42 Failed, 0 Skipped). The agent never reached a state where any tests pass.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** After each batch, run a quick build+test validation. If all tests fail, prioritize fixing existing failures before generating more tests.
- **Why ROI:** A fail-fast approach would surface issues like missing usings and sealed classes early, saving time on later batches.
- **Observed in:** Testing Agent Run 1 — netlandingpage (1 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs

### Issue 13 (ROI: MEDIUM) (🔺 1): Redundant filter method tests across BlogService

- **Problem:** BlogServiceTests has 47 methods with significant overlap — `RemovePostsWithTag`, `KeepPostsWithTag`, etc. all have near-identical test patterns. At least 7 are effectively redundant.
- **Fix location:** LLM prompt/model
- **Suggested fix:** When multiple methods share the same filter pattern, generate a parameterized test or test fewer variants for subsequent methods.
- **Why ROI:** Reduces test count by ~15% without sacrificing coverage, making test suites more maintainable.
- **Observed in:** Testing Agent Run 1 — netlandingpage (1 of 7 reports)
- **Recurrence:** 1 of 3 Testing Agent runs

### Issue 14 (ROI: MEDIUM) (🔺 1): Missing data mapping verification tests

- **Problem:** GetBasket maps BasketItem entities to CustomerBasketResponse items (ProductId, Quantity). No test verifies this mapping with multiple items or checks individual field values.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Add a test that sets up a basket with 3+ items and asserts each item's `ProductId` and `Quantity` map correctly.
- **Why ROI:** Mapping bugs are common in gRPC services and are caught by tests that verify field-level accuracy.
- **Observed in:** Testing Agent eShop Run 1 (1 of 12 reports)
- **Recurrence:** 1 of 5 Testing Agent runs (Copilot generated this test; Testing Agent did not)

### Issue 15 (ROI: MEDIUM) (🔺 2): No queue overflow boundary test generated

- **Problem:** `NotificationService` enforces `_maxQueueLength = 500` with `while (_queue.Count > _maxQueueLength && _queue.TryDequeue(out _)) { }`. Neither Testing Agent run generated a test that sends >500 items to verify the cap. Copilot's batch test sends only 10; Testing Agent sends at most 2 per test.
- **Fix location:** LLM prompt/model + Testing Agent orchestrator
- **Suggested fix:** When source code contains numeric constants used in boundary conditions (e.g., `_maxQueueLength = 500`), generate at least one boundary test that exercises the limit.
- **Why ROI:** Boundary conditions represent explicit design decisions that could break silently during refactoring.
- **Observed in:** TA Run 1 — ContosoUniversity, TA Run 2 — ContosoUniversity, Comparison — ContosoUniversity (3 of 12 reports)
- **Recurrence:** 2 of 5 Testing Agent runs (both ContosoUniversity runs)

### Issue 16 (ROI: MEDIUM) (🔺 2): No exception handling path tests

- **Problem:** Both `SendNotification` and `ReceiveNotification` wrap their logic in try/catch blocks that write to `Debug.WriteLine`. No Testing Agent run generated tests that force exceptions to verify graceful handling. These try/catch blocks are behavioral code (catch + log + continue) that accounts for part of the uncovered 17.3%.
- **Fix location:** LLM prompt/model
- **Suggested fix:** When source contains try/catch blocks, generate tests that exercise the exception path — e.g., verify that the method doesn't throw and returns gracefully when an internal operation fails.
- **Why ROI:** Exception-swallowing patterns are a common source of silent failures; testing them catches regression if someone removes the try/catch.
- **Observed in:** TA Run 1 — ContosoUniversity, TA Run 2 — ContosoUniversity, Comparison — ContosoUniversity (3 of 12 reports)
- **Recurrence:** 2 of 5 Testing Agent runs

### Issue 17 (ROI: MEDIUM) (🔺 1): Testing Agent adds Moq but never uses it

- **Problem:** The Testing Agent added Moq (4.20.72) as a package dependency in the ContosoUniversity testingagent2 run but generated zero tests that use Moq. `NotificationService` has no injectable dependencies, so mocking isn't needed. The unused package adds build overhead.
- **Fix location:** Testing Agent orchestrator
- **Suggested fix:** Analyze the target class for injectable dependencies before adding mock frameworks. If the class has no constructor-injected interfaces, skip adding Moq/NSubstitute.
- **Why ROI:** Unnecessary packages slow builds and confuse developers reviewing generated test projects.
- **Observed in:** TA Run 2 — ContosoUniversity/testingagent2, Comparison — ContosoUniversity (2 of 12 reports)
- **Recurrence:** 1 of 5 Testing Agent runs

---

## LOW ROI

### Issue 18 (ROI: LOW) (🔺 1): Neither tool tests exception propagation from repository

- **Problem:** `DeleteBasket` and `UpdateBasket` call repository methods without try/catch. Neither tool generated a test verifying repository exceptions propagate correctly.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Add a test where the dependency throws and verify the exception propagates unchanged.
- **Why ROI:** Minor — documents expected behavior, useful for regression safety.
- **Observed in:** Testing Agent eShop, Copilot eShop, Comparison eShop (3 of 12 reports)
- **Recurrence:** 1 of 5 Testing Agent runs (eShop)

### Issue 19 (ROI: LOW) (🔺 3): Constructor-not-null trivial tests

- **Problem:** `Constructor_WithValidParameters_InitializesService()` tests just assert the constructor doesn't throw and instance is not null. Merged into Issue 7 for ContosoUniversity, but still appears independently in netlandingpage runs.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Skip constructor-only tests unless the constructor has validation logic, branching, or default value computation.
- **Why ROI:** Minor noise reduction — each trivial test dilutes the behavioral percentage metric.
- **Observed in:** Testing Agent Run 1, Testing Agent Run 2 — netlandingpage, TA Run 2 — ContosoUniversity (3 of 12 reports)
- **Recurrence:** 3 of 5 Testing Agent runs

### Issue 20 (ROI: LOW) (🔺 2): Constant-value assertion tests in DotnetReleaseServiceTests

- **Problem:** 4 tests assert public constants equal hardcoded values. Zero regression protection.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Skip tests that only assert a constant equals a hardcoded expected value.
- **Why ROI:** Minor — 4 tests, but sets a pattern for cleaner generation.
- **Observed in:** Testing Agent Run 1, Testing Agent Run 2 — netlandingpage (2 of 12 reports)
- **Recurrence:** 2 of 5 Testing Agent runs

### Issue 21 (ROI: LOW) (🔺 1): Neither tool tests CancellationToken handling

- **Problem:** `ServerCallContext` includes a `CancellationToken` property. Neither tool generated tests for cancellation scenarios.
- **Fix location:** LLM prompt/model
- **Suggested fix:** Add a test that sets `CancellationToken` to a cancelled token and verifies the service handles it appropriately (either throws `OperationCanceledException` or passes it to the repository).
- **Why ROI:** Minor — cancellation handling is important for production resilience but rarely causes test failures.
- **Observed in:** Comparison — eShop (1 of 12 reports)
- **Recurrence:** 1 of 5 Testing Agent runs (eShop)

### Issue 22 (ROI: LOW) (🔺 2): Missing `GenerateMessage` default switch arm test

- **Problem:** The `GenerateMessage` method has a `_ => $"{displayText} operation: {operation}"` default case in its switch expression. No test covers this branch. While C# enums make it unlikely to hit in normal use, it's reachable via future enum additions or casting.
- **Fix location:** LLM prompt/model
- **Suggested fix:** When source contains switch expressions with a default/wildcard case, generate a test that exercises it or flag it as an intentionally uncovered branch.
- **Why ROI:** Default branches are where bugs hide when enums are extended, but impact is limited.
- **Observed in:** TA Run 1 — ContosoUniversity, TA Run 2 — ContosoUniversity, Comparison — ContosoUniversity (3 of 12 reports)
- **Recurrence:** 2 of 5 Testing Agent runs
