---
name: llm-efficiency
description: >
  Analyze LLM usage in a .NET test generation run and find waste — unnecessary calls, token bloat,
  stalls, retry loops, and cache misses. Works with both .NET Testing Agent and GH Copilot Agent
  Mode logs. Produces a short efficiency report with concrete reduction strategies.
  Use this skill when asked to analyze LLM costs, reduce token usage, find wasted calls,
  optimize LLM efficiency, or review token consumption in a test run.
  Trigger phrases include "LLM efficiency", "reduce tokens", "token waste", "LLM cost analysis",
  "optimize LLM calls", "why so many calls", "token usage report", "reduce LLM cost".
---

# LLM Efficiency Analysis Skill

## Speed Contract

Complete in **≤ 3 tool-call rounds**:
1. **Round 1** — locate log + read head/tail + read run-metadata.md (parallel)
2. **Round 2** — one targeted grep for LLM call patterns across the full log
3. **Round 3** — write efficiency report and print summary

Do NOT read the full log line by line. Do NOT ask the user for metadata.

---

## How to Use

```
Analyze LLM efficiency for C:\path\to\repo
```
```
How can I reduce token usage in this test run?
```
```
Why did this run use so many LLM calls? C:\path\to\artifacts
```

### What you get

A short efficiency report with:
- LLM usage summary (calls, tokens, time)
- Identified waste patterns with estimated savings
- Concrete strategies to reduce cost
- Saved as `llm-efficiency.md` in the artifacts folder

---

## Rules

- **Read-only** — do not modify files, run tests, or build.
- **No quality scoring** — focus only on LLM resource consumption, not test quality.
- **No user prompts** — extract everything from logs.
- **≤ 5 waste findings** — this is a focused efficiency audit, not a full review.
- **Always quantify** — every finding must include estimated calls/tokens/time that could be saved.

---

## Workflow

### Step 1: Locate log + detect tool type (single command)

Same as `run-diagnosis` skill — find the latest artifacts folder, detect Testing Agent vs. Copilot by checking for `testingagent-logs/codetestingagent.log` vs. `copilot-output.log`.

```powershell
$path = "<user-provided-path>"

# Read artifact root from saved preference
$artifactRootFile = Join-Path $env:USERPROFILE ".copilot\unittest-artifact-root.txt"
if (Test-Path $artifactRootFile) {
  $artifactRoot = (Get-Content $artifactRootFile -Raw).Trim()
} else {
  $artifactRoot = Join-Path $path "artifacts"
}

$artDir = $null
foreach ($sub in @("testingagent", "copilot")) {
    $d = Get-ChildItem -Path (Join-Path $artifactRoot $sub) -Directory -ErrorAction SilentlyContinue |
        Sort-Object Name -Descending | Select-Object -First 1
    if ($d -and (!$artDir -or $d.Name -gt $artDir.Name)) { $artDir = $d }
}
if (!$artDir) { $artDir = Get-Item $path }

$taLog = Join-Path $artDir.FullName "testingagent-logs\codetestingagent.log"
$copilotLog = Join-Path $artDir.FullName "copilot-output.log"
if (Test-Path $taLog) { Write-Host "TOOL:TestingAgent"; Write-Host "LOG:$taLog" }
elseif (Test-Path $copilotLog) { Write-Host "TOOL:Copilot"; Write-Host "LOG:$copilotLog" }
Write-Host "ARTIFACTS:$($artDir.FullName)"
```

### Step 2: Extract LLM data (parallel reads)

Make these calls **simultaneously in a single response**:

1. **Read first 80 lines** of log — captures model, prompt, scope, configuration
2. **Read last 80 lines** of log — captures final outcomes, total duration
3. **Read `run-metadata.md`** if it exists — this is the **primary source** for identity metadata (Tool, Model, VS Version, Copilot Chat Version, Duration, etc.). If it contains all needed identity fields, skip step 4.
4. **Only if `run-metadata.md` is missing or incomplete:** Read first 350 lines of `copilot-output.log` if it exists — search for `Copilot chat version` to extract VS Version and Copilot Chat Version, and `PreferredModelFamily=` for the model. This is the fallback path only.
5. **Grep for LLM patterns** across the full log file (one grep, multiple patterns OR'd)

#### Testing Agent grep — single call:

```
Pattern: "Starting LLM call|LLM call completed|TotalElapsed|Duration:|get_type_info|Cancellation triggered|Processing project|fix cycle|Iteration|Build:"
```

This returns all LLM-related lines in one pass. From the results, extract:
- **Total LLM calls** — count `Starting LLM call` matches
- **Per-call duration** — parse `TotalElapsed` or `Duration:` values
- **Longest calls** — identify calls with duration > 60s
- **Tool call patterns** — count `get_type_info` calls (type search stalls)
- **Fix iterations** — count `fix cycle` / `Iteration` lines (retry loops)
- **Projects processed** — count `Processing project` (scope breadth)

#### Copilot grep — single call:

```
Pattern: "EventType\(9\)|InputTokenCount|OutputTokenCount|TotalTokenCount|cached_tokens|SemanticSearchStrategy|FileEditingState|Begin sending message"
```

From the results, extract:
- **LLM round-trips** — count `EventType(9)` matches
- **Token totals** — sum `InputTokenCount`, `OutputTokenCount`, `TotalTokenCount`
- **Cached tokens** — sum `cached_tokens` values
- **Cache hit rate** — cached / total input tokens
- **Semantic searches** — count `SemanticSearchStrategy` (context-gathering cost)
- **File edits** — count `FileEditingState` (useful work output)
- **Turns** — count `Begin sending message` (conversation turns)

### Step 3: Analyze and classify waste patterns

Using the extracted data, check for these patterns. Only report patterns that are **evidenced by the data** — do not speculate.

#### Waste Patterns — Testing Agent

| Pattern | Detection | Savings estimate |
|---------|-----------|-----------------|
| **Type search stalls** | `get_type_info` calls > 20, or a single LLM call > 5 min with `get_type_info` in surrounding lines | Estimate: time spent in stalled calls as % of total |
| **Broad scope overhead** | `Processing project` count > 5 but most projects produce 0 tests | Estimate: calls spent on zero-output projects |
| **Fix iteration loops** | > 3 fix iterations for a single file, or tests deleted during fix | Estimate: LLM calls in fix iterations vs. total |
| **Redundant builds** | Multiple `Build:` lines with same result | Estimate: time in redundant build+LLM cycles |
| **Cancelled mid-generation** | `Cancellation triggered` while LLM calls were in progress | Estimate: all calls after the last useful output |
| **Slow model choice** | Per-call duration averaging > 30s with a premium model | Suggest: faster model for iterative fix cycles |

#### Waste Patterns — Copilot Agent Mode

| Pattern | Detection | Savings estimate |
|---------|-----------|-----------------|
| **Low cache hit rate** | `cached_tokens` / `InputTokenCount` < 30% | Estimate: tokens that could be cached with better context reuse |
| **Token bloat per turn** | Average `OutputTokenCount` > 4000 per turn | Estimate: excess tokens vs. 2000-token baseline |
| **Excessive round-trips** | `EventType(9)` count > 15 for a single-file task | Estimate: calls beyond expected 5-8 for single-file generation |
| **Broad semantic search** | `SemanticSearchStrategy` count > 10 | Estimate: LLM calls spent processing search results |
| **No build validation** | File edits found but no `dotnet test` / `dotnet build` in log | Flag: unvalidated generation wastes user time, not LLM tokens |
| **Repeated context** | Multiple turns with similar `InputTokenCount` (within 5%) | Estimate: duplicate context tokens across turns |

### Step 4: Write the efficiency report

Compose and save the report, then print it. Use this format:

```markdown
# LLM Efficiency Report — <ProjectName> (<MM/DD/YYYY>)

## LLM Usage Summary

| Metric | Value |
|--------|-------|
| **Tool** | <from run-metadata.md, or detect from log> |
| **Model** | <from run-metadata.md, or extract from log> |
| **VS Version** | <from run-metadata.md, or auto-extract from Copilot log> |
| **Copilot Chat Version** | <from run-metadata.md, or auto-extract from Copilot log, or "Unknown"> |
| **Total LLM calls** | <N> |
| **Total duration** | <X min Y sec> |
| **Total tokens** | <N input + N output = N total> (Copilot only) |
| **Cached tokens** | <N (X% of input)> (Copilot only) |
| **Avg call duration** | <X sec> (Testing Agent only) |
| **Longest call** | <X sec — context of what it was doing> |
| **Fix iterations** | <N> |
| **Projects in scope** | <N total, M produced tests> |

## Efficiency Verdict: <emoji> <one-line summary>

<1-3 sentences: was this run efficient, wasteful, or somewhere in between? Quantify the waste.>

## Waste Findings

### Finding N: <title>

- **What happened:** <describe the waste pattern — cite log evidence>
- **Estimated waste:** <N calls / N tokens / N min that could be saved>
- **Reduction strategy:** <concrete, actionable change>
- **Where to fix:** <`LLM prompt/model` | `Agent orchestrator` | `User workflow` | etc.>

(Repeat for each finding. Max 5.)

## Cost Estimate (<model name>)

| Component | Tokens | Est. Cost |
|-----------|--------|-----------|
| Input tokens (<N>K) | <N> | ~$<X.XX> |
| Output tokens (<N>K) | <N> | ~$<X.XX> |
| **Total** | <N> | **~$<X.XX>** |
| **Per test** | <N> | **~$<X.XX>** |
| **Per working test** | <N> | **~$<X.XX>** |

#### Cost Methodology

- **Token source:** <describe where tokens were extracted from — e.g., "Exact per-call counts from VS Copilot diagnostic log `EventType(11)` entries" or "Recovered from VS Copilot diagnostic log for the same VS session">
- **Pricing basis:** <model pricing used, e.g., "Public Anthropic claude-opus rates ($15/MTok input, $75/MTok output)">. Actual costs through Copilot subscriptions may differ from raw API pricing.
- **Session isolation:** <note whether the log is scoped to this run only, or covers a broader VS session. Flag if cross-contamination from other interactions is possible.>
- **What's NOT included:** VS infrastructure overhead, network latency costs, or any Microsoft-internal platform fees.
- **Accuracy:** Token counts are exact; dollar amounts are directional estimates based on public pricing.

## Reduction Strategies Summary

| # | Strategy | Estimated savings | Effort |
|---|----------|------------------|--------|
| 1 | <strategy> | <calls/tokens/time saved> | <Low/Medium/High> |
| 2 | ... | ... | ... |

(Rank by savings descending. Include effort estimate: Low = config change or prompt tweak, Medium = orchestrator logic change, High = architecture change.)
```

Save as `llm-efficiency.md` in the artifacts folder, then **also copy** to the central `<artifact_root>/llmefficiency/` directory with a tool-specific name:
- If the run is a Copilot run: copy as `llm-efficiency-copilot.md`
- If the run is a Testing Agent run: copy as `llm-efficiency-testingagent.md`
- Create the `llmefficiency/` directory if it doesn't exist.

This ensures each run folder has its own copy AND the central folder has the latest of each tool type for easy comparison.

Print the report to console.

---

## Baseline Expectations (Reference)

Use these as rough baselines for "normal" LLM usage. Do not include this table in the output — use it to calibrate findings.

| Scenario | Expected LLM calls | Expected duration | Expected tokens (Copilot) |
|----------|-------------------|-------------------|--------------------------|
| Single file, simple class | 3-6 | 1-3 min | 15K-40K total |
| Single project (5-10 source files) | 10-25 | 5-15 min | 50K-150K total |
| Full solution (10+ projects) | 30-80 | 15-45 min | 150K-500K total |
| Fix iteration (per cycle) | 2-4 additional | 1-3 min additional | 10K-30K additional |

Runs significantly exceeding these baselines warrant waste findings.

---

## Common Reduction Strategies (Reference)

Pre-built strategies to recommend when patterns match. Adapt specifics to the run data.

| Strategy | When to recommend | Expected impact |
|----------|------------------|----------------|
| **Narrow the scope** | Solution-wide run with most projects producing 0 tests | 40-70% fewer calls |
| **Use a faster model for fix cycles** | Premium model (Opus/GPT-4) used for iterative compilation fixes | 30-50% time reduction |
| **Add custom instructions** | No `.github/copilot-instructions.md` detected, and agent made framework/pattern mistakes | 10-30% fewer fix iterations |
| **Scope type searches** | `get_type_info` called across all projects for types in one project | 50-80% reduction in stall time |
| **Pre-build before running** | Build errors in first iteration that are pre-existing, not agent-caused | 1-2 fewer LLM fix cycles |
| **Split into per-project runs** | Solution run where agent stalls on inter-project type resolution | Avoids stalls, more predictable duration |
| **Increase context reuse** | Low cache hit rate in Copilot mode | 20-40% token reduction |
| **Batch file generation** | Copilot making 1 file edit per turn instead of batching | 30-50% fewer round-trips |

---

## Cost Calculation (Reference)

### Model pricing table

Use the model detected in `run-metadata.md` or from the log to look up pricing. These are public API rates as of early 2026 — actual Copilot subscription costs may differ.

| Model | Input ($/MTok) | Output ($/MTok) | Notes |
|-------|---------------|-----------------|-------|
| claude-opus-4.6 | $15.00 | $75.00 | Premium tier |
| claude-sonnet-4.6 | $3.00 | $15.00 | Standard tier |
| gpt-4o | $2.50 | $10.00 | Standard tier |
| gpt-4.1 | $2.00 | $8.00 | Standard tier |

If the model is not in this table, note "Unknown pricing — costs not estimated" and skip the cost section.

### Token extraction by tool type

**Copilot Agent Mode:**
- Token data comes from `EventType(11)` entries in the Copilot diagnostic log (`copilot-output.log`).
- Each entry has `InputTokenCount` and `OutputTokenCount` — sum across all entries.
- The log is scoped to a single Copilot chat session, so cross-contamination is minimal.

**Testing Agent:**
- The TA's own log (`codetestingagent.log`) reports "No usage information available" — it does NOT track token usage.
- Token data must be recovered from the **VS Copilot diagnostic log for the same VS session**. This is the log under `%TEMP%\VSGitHubCopilotLogs\` that was active when the TA ran.
- To find the right log: search Copilot logs that reference the TA repo path, or match by timestamp overlap with the TA run window.
- The Copilot log captures all LLM API calls routed through the Copilot infrastructure, including TA calls as `EventType(11)`.
- **Caveat:** This log covers the entire VS session, not just the TA run. If other Copilot interactions occurred in the same VS window, their tokens may be included. Note this in the Cost Methodology section when applicable.

### Formulas

```
Total cost    = (input_tokens × input_rate / 1,000,000) + (output_tokens × output_rate / 1,000,000)
Per test      = total_cost / tests_generated
Per working   = total_cost / tests_passing
```

**"Per working test"** is the most meaningful cost-efficiency metric — it penalizes runs that generate broken tests (e.g., Moq on non-virtual methods), making it directly comparable across tools.

---

## Style

- **Quantify everything** — never say "too many calls" without a number
- **Lead with the verdict** — user wants to know "was this efficient?" immediately
- **Actionable strategies** — every finding must have a concrete "do this to save X"
- **Tables over prose** — use the summary table for scannable results
- **Emoji**: ✅ Efficient, ⚠️ Some waste, ❌ Significant waste, 💸 for cost callouts
- **Always use Unicode emoji** (❌, ✅, ⚠️, 💥) — never use shortcodes like `:x:` or `:boom:` as they don't render in all viewers
