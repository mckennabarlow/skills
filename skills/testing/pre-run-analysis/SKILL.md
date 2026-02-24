---
name: pre-run-analysis
description: >
  Analyze a .NET project before running the Testing Agent or Copilot Agent Mode for test generation.
  Detects blockers, maps the testable surface, and produces a pre-run checklist with a quality bar.
  Works in VS Code, Visual Studio, and Copilot CLI.
  Trigger phrases include "pre-run analysis", "analyze before testing", "pre-run checklist",
  "check before test generation", "prepare for test run", "pre-flight check".
---

# Pre-Run Analysis Skill

## How to Use

Invoke this skill before running either the .NET Testing Agent or GH Copilot Agent Mode for test generation. Provide:

1. **The source file(s) or folder** you plan to target for test generation
2. **The repo root or solution folder** (so the skill can check project structure, packages, etc.)

### Example prompts

```
Pre-run analysis for C:\repos\MyApp\src\Services\OrderService.cs
```

```
Analyze before testing C:\repos\MyApp\src\Services\
```

```
Pre-flight check for test generation on #OrderService.cs
```

### What you get

A single markdown checklist saved to the artifacts folder:

| Report | Description |
|--------|-------------|
| `MMDDYYYY-PreRunAnalysis-<ProjectName>.md` | Blockers, warnings, testable surface map, suggested prompt, and quality bar |

### Cross-environment compatibility

This skill works in **VS Code**, **Visual Studio**, and **Copilot CLI**. It relies only on:

- Reading source files (`.cs`, `.csproj`, `.sln`, `.props`)
- Optionally running `dotnet build` and `dotnet test` (gracefully skipped if unavailable)
- Producing markdown output

No environment-specific APIs are required.

---

## Purpose

Prevent wasted test generation runs by detecting blockers, mapping what should be tested, and setting a clear quality bar — **before** you spend time and tokens on test generation.

This skill encodes every known gotcha and pattern observed across all prior evaluation runs (see "Known Issue Patterns" section below). It is designed to be the universal "step 0" regardless of which tool you use next.

---

## Inputs

You will be provided:

- **Target source file(s)** — the `.cs` file(s) or folder of source files to analyze. If a folder is provided, analyze all `.cs` files in the folder (excluding generated files, `obj/`, `bin/`, and `*.g.cs`).
- **Repo root / solution** — ask the user for the repo root or `.sln` path if not obvious from context. Needed to find `.csproj`, `Directory.Packages.props`, and existing test projects.
- **Which tool(s) will be used** — ask: "Will you run the Testing Agent, Copilot Agent Mode, or both?" This determines which tool-specific checks to include.
- **Output directory** — use the centralized artifact root for all output.
  - **Read artifact root:** Check `C:\Users\cathys\.copilot\unittest-artifact-root.txt` for the saved preference. If the file doesn't exist, ask the user where to save artifacts and save their choice there.
  - **Output location:** `<artifact_root>/pre-run/`
  - Create the `pre-run/` directory if it doesn't exist.

---

## Output

Generate one `.md` file inside the output directory.

### File Name

**Format:** `MMDDYYYY-PreRunAnalysis-<ProjectName>.md`

Extract `<ProjectName>` from the solution name, repo folder name, or target project. Do not hardcode any specific project name.

**Example:** `02112026-PreRunAnalysis-eShop.md`

---

## Analysis Steps

Execute these steps in order. Each step produces a section in the output report.

### Step 1: Discover Project Structure

Locate and read the following files (relative to the repo root or solution directory):

| File | What to extract |
|------|----------------|
| `*.sln` | Solution name, list of projects |
| `*.csproj` (target project) | Target framework, NuGet packages, project references |
| `Directory.Packages.props` | Whether Central Package Management (CPM) is enabled |
| `Directory.Build.props` | Global properties, `ManagePackageVersionsCentrally` |
| `*.csproj` (test projects, if any) | Test framework (MSTest/xUnit/NUnit), mock library (Moq/NSubstitute), existing packages |

**If no test project exists:** Note this clearly and skip all test-project-related checks. The checklist should recommend creating one and suggest which framework/mock library to use based on the target project's complexity.

### Step 2: Analyze Target Source Files

For each target `.cs` file, read the source code and extract:

| Signal | How to detect | Why it matters |
|--------|--------------|----------------|
| **Public methods** | Methods with `public` access modifier | These are the testable API surface |
| **Constructor parameters** | Constructor parameter types | Each parameter is a dependency that must be mocked or provided |
| **Sealed dependencies** | Constructor parameters whose types are `sealed` classes (not interfaces) | Cannot be mocked with standard frameworks — **BLOCKER** |
| **Static dependencies** | Static method calls or static class dependencies in method bodies | Cannot be mocked without wrappers — **BLOCKER** |
| **Interface dependencies** | Constructor parameters typed as interfaces (`IFoo`) | Clean mockability — no issues expected |
| **Abstract class dependencies** | Constructor parameters typed as abstract classes | Mockable but may have complex setup |
| **Branching complexity** | `if/else`, `switch`, ternary operators, null-coalescing (`??`), null-conditional (`?.`) in method bodies | Each branch = a test case; more branches = more tests needed |
| **Exception handling** | `try/catch` blocks, `throw` statements | Exception paths need explicit tests |
| **Async methods** | `async Task<T>` or `async ValueTask<T>` return types | Tests must use `async Task` with `await` |
| **CancellationToken parameters** | Parameters of type `CancellationToken` | Cancellation scenarios should be tested |
| **Nullable reference types** | Parameters or return types with `?` (nullable) | Null-guard branches need tests |
| **LINQ / lambda complexity** | `.Where()`, `.Select()`, `.Any()`, `.FirstOrDefault()`, complex lambdas | Filter/transform logic = behavioral test targets |
| **Logging calls** | `_logger.Log*()`, `ILogger` usage | Flag as low-value test targets — avoid trivial log-verification tests |
| **Data mapping** | Property-to-property assignments, `.Select(x => new Foo { ... })` patterns | Mapping accuracy is a common source of bugs — needs field-level assertion tests |
| **gRPC / API contracts** | Protobuf-generated types, `ServerCallContext` | Special setup required for test context |

### Step 3: Check for Known Blockers

Apply the following checks. Each produces a 🔴 (blocker), 🟡 (warning), or ✅ (clear) status.

#### 🔴 Blockers (will likely cause test generation failure)

| # | Check | Condition | Impact |
|---|-------|-----------|--------|
| B1 | **Sealed class dependencies** | Any constructor parameter is a `sealed` class (not an interface) | Testing Agent: fix-loop stall for 1h+. Copilot: will generate unmockable code. **Action:** Extract an interface or wrap in an adapter before running. |
| B2 | **Static method dependencies** | Method body calls static methods on concrete classes (e.g., `File.ReadAllText()`, `DateTime.Now`) | Both tools will generate tests that can't isolate the static call. **Action:** Wrap behind an interface or use a time provider pattern. |
| B3 | **No test project + CPM enabled** | No test project exists AND `Directory.Packages.props` is present | Both tools may create a test project that violates CPM. **Action:** Create the test project manually with the correct package references first. |
| B4 | **Target project doesn't build** | `dotnet build` fails on the target project (if build can be run) | Both tools will fail if the source doesn't compile. **Action:** Fix build errors first. |

#### 🟡 Warnings (may reduce quality or cause partial failure)

| # | Check | Condition | Impact |
|---|-------|-----------|--------|
| W1 | **Central Package Management** | `Directory.Packages.props` exists or `ManagePackageVersionsCentrally` is true | Copilot may add inline `Version` attributes causing NU1008. Testing Agent handles this better but may still stumble. **Action:** Note this; be ready to fix `.csproj` after the run. |
| W2 | **Existing test failures** | Existing tests fail (if `dotnet test` can be run) | Pre-existing failures confuse both tools' test discovery and validation. **Action:** Fix or skip failing tests first. |
| W3 | **Large file count** | More than 5 source files targeted | Testing Agent averages 8–15 min per file. Copilot may skip files silently. **Action:** Batch into groups of 3–5 files per run. |
| W4 | **No mocking framework in test project** | Test project exists but has no Moq/NSubstitute/FakeItEasy package | Both tools will try to add one; may conflict with CPM. **Action:** Pre-install your preferred mock framework. |
| W5 | **Complex constructors (5+ parameters)** | Constructor has 5+ parameters | Tests will have verbose setup; both tools may miss dependencies. **Action:** Consider testing through a builder pattern or factory. |
| W6 | **High existing coverage (>90%)** | Coverage report shows >90% on target files (if available) | Diminishing returns — generated tests may be redundant. **Action:** Prioritize low-coverage files instead. |
| W7 | **No existing tests** | Test project exists but has 0 test methods for the target class | Both tools must generate tests from scratch — no patterns to follow. **Action:** Expected behavior; just be aware of longer run times. |
| W8 | **Test discovery caching (VS only)** | User is running in Visual Studio | VS caches test discovery; new tests may show `[None]` status after generation. **Action:** Run `dotnet test` from terminal after generation to validate, or restart VS. |

#### ✅ Clear signals

| # | Check | Condition | What it means |
|---|-------|-----------|---------------|
| C1 | **All dependencies are interfaces** | Every constructor parameter is an interface type | Clean mockability — both tools should produce high-quality mocks |
| C2 | **Test project with matching framework** | Test project exists with a mock framework already installed | Tools can reuse the existing framework without adding packages |
| C3 | **Target builds cleanly** | `dotnet build` succeeds (if run) | No pre-existing compilation issues to worry about |
| C4 | **Few files, focused scope** | 1–3 target files | Both tools work best with focused, single-file prompts |

### Step 4: Map the Testable Surface

For each public method in each target file, produce a **test scenario table**:

```
### <ClassName>

| Method | Scenario | Branch/Condition | Expected test type |
|--------|----------|-----------------|-------------------|
| GetBasket | Valid user, basket exists | `items.Count > 0` | Behavioral — verify mapping |
| GetBasket | Valid user, empty basket | `items.Count == 0` | Behavioral — verify empty response |
| GetBasket | Null/empty userId | null guard | Behavioral — verify exception or empty |
| DeleteBasket | Repository throws | no try/catch | Behavioral — verify exception propagation |
| ... | ... | ... | ... |
```

This table becomes the **expected test list** — it defines what "complete" looks like for this target.

**Guidelines for scenario generation:**
- One row per distinct branch, null-guard, or edge case
- Mark each as Behavioral, Edge Case, or Error Path
- Do NOT include trivial scenarios like "constructor creates instance" or "logger.IsEnabled returns true"
- DO include negative scenarios (dependency throws, invalid input, cancelled token)
- DO include data mapping scenarios when the method transforms data (verify field-level mapping)
- Count the total scenarios — this is the expected minimum test count

### Step 5: Generate Tool-Specific Recommendations

Based on the analysis, produce recommendations tailored to the tool(s) the user will run.

#### For the .NET Testing Agent (`@Test` prompt)

| Consideration | Recommendation |
|--------------|----------------|
| **Sealed class workaround** | If sealed classes are detected (B1), recommend extracting interfaces before running. The Testing Agent cannot recover from sealed-class mocking failures — it will stall in fix loops. |
| **File batching** | If >5 files, recommend running `@Test Write unit tests for #<FileName>` one file at a time. The agent handles single-file prompts more reliably. |
| **Framework preference** | If no test project exists, note that the Testing Agent will create one with MSTest + NSubstitute by default. If the user prefers different frameworks, they should create the test project first. |
| **Coverage measurement** | The Testing Agent measures coverage automatically. Note the initial coverage so the user can verify the delta after the run. |
| **Fix iteration budget** | If the target has complex dependencies, warn that fix iterations may take 30+ minutes. The user should be prepared to wait or cancel after 20 minutes of no progress. |

#### For GH Copilot Agent Mode (Copilot Chat prompt)

| Consideration | Recommendation |
|--------------|----------------|
| **CPM workaround** | If CPM is detected (W1), recommend pre-installing the mock framework. Copilot will add inline `Version` attributes that break the build. |
| **Prompt engineering** | Generate a **suggested prompt** based on the analysis. Include specific instructions derived from the testable surface map (see Step 6). |
| **Build validation reminder** | Copilot does not run `dotnet build` or `dotnet test` after generating tests. Remind the user to validate manually. |
| **Test discovery** | If running in VS, warn about discovery caching (W8). Recommend running `dotnet test` from terminal to validate. |
| **File scope** | If >3 files, recommend separate prompts per file. Copilot may silently skip files in multi-file prompts. |

### Step 6: Generate Suggested Prompts

Based on the testable surface map and detected constraints, generate a ready-to-use prompt for each tool.

#### Testing Agent prompt

```
@Test Write unit tests for #<FileName>
```

The Testing Agent prompt is simple — most intelligence comes from the agent itself. But if specific constraints were detected, add a note for the user:

```
⚠️ Before running: [extract interface for SealedClassName] / [pre-install NSubstitute] / [fix build errors first]
```

#### Copilot Agent Mode prompt

Generate a detailed prompt that steers Copilot away from known issues:

```
Write unit tests for <ClassName> in <FileName>.

Requirements:
- Use <MSTest/xUnit/NUnit> with <Moq/NSubstitute> (match existing test project)
- Test all public methods: <list methods>
- Include negative scenarios: <list from testable surface map>
- Include exception propagation tests for methods that call dependencies without try/catch
- Do NOT generate trivial constructor-not-null tests
- Do NOT generate debug-logging verification tests (logger.IsEnabled checks)
- Use async Task test methods for async service methods
- Include at least one DidNotReceive() / Times.Never assertion for guard clause tests
- After generating tests, run `dotnet build` and `dotnet test` to validate
<if CancellationToken detected>
- Include a cancellation scenario test using a pre-cancelled CancellationToken
<endif>
<if data mapping detected>
- Include a mapping verification test that checks individual field values with 3+ items
<endif>
<if CPM detected>
- Do NOT add Version attributes to PackageReference elements — this repo uses Central Package Management
<endif>
```

### Step 7: Define the Quality Bar

Produce a quality bar table tailored to the specific target. This uses the testable surface map to set concrete expectations.

```
## 🎯 Quality Bar

| Dimension | Expected | How to verify |
|-----------|----------|---------------|
| **Total test count** | ≥<N> (from testable surface map) | Count [TestMethod] / [Fact] / [Test] methods |
| **Behavioral %** | ≥80% | Review test classification in the review report |
| **Branch coverage** | All branches in testable surface table have ≥1 test | Compare test methods against scenario table |
| **Negative assertions** | ≥<M> DidNotReceive/Times.Never calls | Search generated files for negative patterns |
| **Exception tests** | ≥<P> tests for exception propagation | Search for `.Throws` or `Assert.ThrowsAsync` |
| **No trivial tests** | 0 constructor-not-null tests, 0 debug-logging tests | Search for `Constructor_With` and `LogsDebugMessage` |
| **Async correctness** | All async method tests use `async Task` | Check test method signatures |
| **Build passes** | `dotnet build` succeeds | Run after generation |
| **Tests pass** | `dotnet test` succeeds | Run after generation |
```

The `<N>`, `<M>`, `<P>` values come from the testable surface map in Step 4.

### Step 8: Produce the Final Checklist

Combine all findings into a single actionable checklist at the top of the report:

```
## ✅ Pre-Run Checklist

### 🔴 Fix Before Running
- [ ] <blocker 1 — with specific action>
- [ ] <blocker 2 — with specific action>

### 🟡 Be Aware
- [ ] <warning 1 — what might happen and how to recover>
- [ ] <warning 2>

### ✅ Ready
- <clear signal 1>
- <clear signal 2>

### 📋 After the Run
- [ ] Run `dotnet build` on the test project
- [ ] Run `dotnet test` to validate all tests pass
- [ ] Compare generated tests against the Testable Surface Map (Step 4)
- [ ] Check for trivial tests (constructor-not-null, debug-logging)
- [ ] Verify at least one negative assertion (DidNotReceive / Times.Never)
- [ ] Run the review skill to get a scored evaluation
```

### Step 9: Generate Machine-Readable Instruction Files

Generate two `.instructions.md` files in the repo's `.github/instructions/` directory. These files are automatically consumed by Copilot (in VS, VS Code, and CLI) and by the Testing Agent's underlying LLM.

**Create the directory** `.github/instructions/` in the repo root if it doesn't exist.

#### File 1: `.github/instructions/pre-run-copilot-tests.instructions.md`

This file is consumed by **Copilot Agent Mode** (VS, VS Code, CLI) when generating tests. It should be written as direct instructions to the LLM, in second person ("you should", "do not", "always").

**Template:**

```markdown
---
applyTo: "**/*Tests*.cs"
---

# Test Generation Instructions — <ProjectName>

## Project Context
- Target framework: <target framework>
- EF Core version: <version> (use matching InMemory package version)
- Test framework: <MSTest/xUnit/NUnit> with <Moq/NSubstitute>
<if no test project>
- No test project exists yet. Create one named `<ProjectName>.Tests` targeting the same framework.
</if>

## Dependencies to Mock
<for each class analyzed>
- `<ClassName>`: mock <list interfaces>. Use InMemoryDatabase for `<DbContext>`.
<if non-injectable dependency>
- ⚠️ `<ClassName>` has a non-injectable dependency `<DepType>` (created via `new`). You cannot mock it. Test behavior through the real instance or skip assertions on it.
</if>
<if static calls>
- ⚠️ `<ClassName>` calls static methods: <list>. These cannot be mocked. For file I/O, use a temp directory via IWebHostEnvironment mock. For DateTime.Now, accept the value or use a tolerance.
</if>
</for>

## Required Test Scenarios
<for each class, list the testable surface map scenarios as bullet points>
### <ClassName>
- <Method>: <scenario description> → assert <expected behavior>
- ...
</for>

## Quality Rules
- Do NOT generate trivial constructor-not-null tests (unless the constructor has validation logic)
- Do NOT generate tests that only verify debug/info log calls (logger.IsEnabled, LogDebug, LogInformation)
- Do NOT generate tests for constant field values
- DO include at least one negative mock assertion (DidNotReceive / Times.Never / VerifyNoOtherCalls) per class with conditional logic
- DO include exception propagation tests for methods that call dependencies without try/catch
- DO include null-input / invalid-input tests for every method that has null guards
- DO use async Task test methods for async service methods (never sync void)
- DO drain any static shared state (e.g., ConcurrentQueue) in [TestInitialize] / constructor to prevent cross-test contamination
<if CancellationToken detected>
- DO include a cancellation test using a pre-cancelled CancellationToken
</if>
<if data mapping detected>
- DO include a mapping verification test that asserts individual field values
</if>

## Build & Validate
- After generating tests, run `dotnet build` on the test project
- Then run `dotnet test` to confirm all tests pass
- If build fails, fix the errors before generating more tests

<if CPM detected>
## Package Management
This repo uses Central Package Management (Directory.Packages.props). Do NOT add Version attributes to PackageReference elements. Add new package versions to Directory.Packages.props instead.
</if>
```

#### File 2: `.github/instructions/pre-run-testingagent-tests.instructions.md`

This file is consumed by the **Testing Agent's underlying LLM** in Visual Studio. The Testing Agent has its own orchestration, so these instructions focus on code-generation guidance rather than workflow commands (the agent handles build/test/fix itself).

**Key differences from the Copilot file:**
- Do NOT include "run `dotnet build`" or "run `dotnet test`" — the agent does this automatically
- Do NOT include workflow instructions — the agent has its own fix loop
- DO include mock setup guidance — the agent's LLM needs to know how to construct mocks
- DO include scenario lists — the agent uses them to decide what tests to generate
- Keep it concise — the agent's context window is shared with log data and fix iterations

**Template:**

```markdown
---
applyTo: "**/*Tests*.cs"
---

# Test Generation Guidance — <ProjectName>

## Mock Setup
<for each class>
- `<ClassName>(<params>)`: use <InMemoryDatabase / mock interface> for each dependency
<if non-injectable>
- `<ClassName>` creates `<DepType>` internally (not injected). Do not attempt to mock it. Test through the real instance.
</if>
<if sealed class>
- ⚠️ `<SealedType>` is sealed. Do NOT generate mock-based tests for this dependency. Either use its interface `<InterfaceName>` or skip tests that require mocking it.
</if>
<if static calls>
- `<ClassName>` uses static calls: <list>. Accept these in tests or use integration-style temp directories.
</if>
</for>

## What to Test
<for each class>
### <ClassName>
<list scenarios as concise bullet points, one per line>
- <Method>(<params>) — <scenario> → <expected>
</for>

## What NOT to Generate
- Constructor-only tests (assert not null / no throw) — skip unless constructor has branching
- Logger verification tests (IsEnabled, LogDebug) — skip unless logging has side effects
- Constant value assertion tests — skip (zero regression value)
- Redundant filter tests — if multiple methods share the same pattern, test one thoroughly and parameterize

## Patterns to Follow
- One negative assertion (DidNotReceive / Times.Never) per class with guard clauses
- Exception propagation tests for unguarded dependency calls
- Null-input tests for every null guard
- async Task test methods for async source methods
<if static shared state>
- Drain static state (e.g., ConcurrentQueue) in [TestInitialize] to isolate tests
</if>
<if EF Core>
- Use Microsoft.EntityFrameworkCore.InMemory version <exact version> for DbContext
</if>
```

#### After generating both files:

1. Confirm the `.github/instructions/` directory was created
2. List the two files and their paths in the report's "📎 Generated Instruction Files" section
3. Note: These files will be **automatically picked up** by Copilot and the Testing Agent on the next run — no manual attachment needed
4. Remind the user: "Make sure **Enable custom instructions** is turned on in VS/VS Code settings under GitHub Copilot → Chat"

---

## Known Issue Patterns

These patterns are derived from all prior evaluation runs. The skill uses them to detect issues proactively.

### From the Testing Agent backlog (18 issues, 7 reports)

| Pattern | What to detect | Impact if missed |
|---------|---------------|-----------------|
| Sealed class mocking | Constructor params with sealed types | 1h+ fix-loop stall, all tests fail |
| Fix loop stall | N/A (pre-run only) — but warn about risk if sealed classes detected | 52 min wasted compute |
| Missing using directives | N/A (pre-run only) — agent-side issue | CS0246 compilation failures |
| Long run times | >5 files targeted | 84+ min runs, files skipped |
| No progress visibility | N/A (agent-side) — but set expectations | User cancels prematurely |
| No negative mock verification | All conditional logic in target methods | Missing regression protection |
| Non-test methods in output | N/A (agent-side) | Compilation errors |
| Over-generation | Large files with many methods | 4× tests generated vs. kept |
| Trivial debug-logging tests | `ILogger` in constructor + `logger.IsEnabled` calls in methods | Low-value tests dilute quality |
| Missing mapping tests | Data transformation methods (Select, mapping constructors) | Mapping bugs missed |
| Constructor-not-null tests | Simple constructors with no branching | Trivial tests with zero value |
| Constant-value tests | Public `const` or `static readonly` fields | Zero regression protection |
| CancellationToken not tested | CancellationToken in method parameters | Missing async safety tests |
| Exception propagation | Methods calling dependencies without try/catch | Missing error path coverage |
| Redundant filter tests | Multiple methods with identical filter patterns | Bloated test suites |

### From the Copilot Agent backlog (12 issues, 7 reports)

| Pattern | What to detect | Impact if missed |
|---------|---------------|-----------------|
| No build validation | N/A (agent-side) — but remind user | "Generated" tests that don't compile |
| CPM package conflicts | `Directory.Packages.props` exists | NU1008 build errors |
| Sealed class risk | Constructor params with sealed types | Unmockable code generated |
| No negative mock verification | Conditional logic in methods | Missing regression protection |
| Files silently skipped | >3 files in prompt scope | Incomplete coverage, no notification |
| Test discovery caching | Running in Visual Studio | New tests show `[None]` status |
| Pre-existing test failures | Existing tests in red state | Confuses test discovery |
| No coverage measurement | N/A (agent-side) — remind user to run coverage after | Cannot quantify test value |
| Sync tests for async methods | Async methods in target | Wrong test patterns |
| Reflection-based testing | Private methods in target | Brittle implementation-coupled tests |
| CancellationToken not tested | CancellationToken in method parameters | Missing async safety tests |
| Exception propagation | Methods without try/catch | Missing error path coverage |

---

## Report Structure

The final markdown report should follow this structure:

```markdown
# Pre-Run Analysis — <ProjectName> (<MM/DD/YYYY>)

## Summary
- **Target:** <file(s) or folder>
- **Tool(s):** <Testing Agent / Copilot Agent Mode / Both>
- **Test project:** <path> or "None — will need to be created"
- **Overall readiness:** 🔴 Blockers found / 🟡 Warnings only / ✅ Ready to run

## ✅ Pre-Run Checklist
<Step 8 output>

## 🔍 Project Structure
<Step 1 output>

## 📊 Source Analysis
<Step 2 output — per-file signal table>

## 🚧 Blockers & Warnings
<Step 3 output — categorized checks>

## 🗺️ Testable Surface Map
<Step 4 output — per-class scenario table>

## 🛠️ Tool-Specific Recommendations
<Step 5 output>

## 💬 Suggested Prompts
<Step 6 output>

## 🎯 Quality Bar
<Step 7 output>

## 📎 Generated Instruction Files
<Step 9 output — paths to the generated .instructions.md files>
```

---

## Style Guidelines

- Use **bullet points** wherever possible instead of prose paragraphs
- Use **bold** for key metrics, file names, and status indicators
- Use **emoji** for status: 🔴 blocker, 🟡 warning, ✅ clear
- Use **tables** for structured data (signals, scenarios, checks)
- Use **checkboxes** (`- [ ]`) for actionable items the user should complete
- Keep analysis sections concise — focus on actionable findings, not exhaustive descriptions
- Include full file paths for all referenced files
- When a check cannot be performed (e.g., `dotnet build` not available), note "Skipped — build not available in this environment" rather than silently omitting

---

## What This Skill Does NOT Do

- **Does not run test generation** — it only analyzes and prepares
- **Does not modify source code** — it only reads and reports
- **Does not require specific tooling** — works with just file access
- **Does not replace the review skills** — run `testing-agent-review` or `copilot-test-review` after the test generation run to score the actual output
