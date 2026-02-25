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
$artDir = $null
foreach ($sub in @("artifacts\test-runs-testingagent", "artifacts\test-runs-copilot")) {
    $d = Get-ChildItem -Path (Join-Path $path $sub) -Directory -ErrorAction SilentlyContinue |
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

#### Cross-log token extraction for Testing Agent runs

The Testing Agent log (`codetestingagent.log`) often reports `HasUsage: False`, meaning token counts are unavailable from that log. However, **the Copilot diagnostic log (`copilot-output.log`) captures token usage for the same LLM calls** because it sits at a higher layer in the VS/VS Code Copilot infrastructure.

**Always check for `copilot-output.log`** in the artifacts folder, even when the primary tool is Testing Agent. If found, grep it for token data:

```
Pattern: "InputTokenCount|OutputTokenCount|TotalTokenCount|cached_tokens|EventType\(9\)"
```

From the results, extract:
- **Token totals** — sum `InputTokenCount`, `OutputTokenCount`, `TotalTokenCount`
- **Cached tokens** — sum `cached_tokens` values
- **Cache hit rate** — cached / total input tokens

If token data is found in the Copilot log, include it in the report summary table under a "Token Usage (from Copilot log)" section. This replaces the "Not available" placeholder that would otherwise appear when the TA log lacks usage data.

**Correlation:** The Copilot log may contain LLM calls from other Copilot features (e.g., completions, chat) in addition to the Testing Agent calls. To correlate, match by timestamp window — only count token data from calls that fall within the Testing Agent run's start/end timestamps (from `run-metadata.md` or the TA log header/footer).

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
| **No smart model switching** | Same premium model used for all calls, including simple tasks (prompt analysis, classification, insight consolidation) that produce short outputs (<2K chars) | Estimate cost using model pricing: calculate current cost at premium rate, then re-price eligible calls at a cheaper model rate. Report the delta as potential savings. |

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
| **Total LLM calls (TA log)** | <N> primary agent calls (Testing Agent only) |
| **Total LLM calls (Copilot log)** | <N> API calls (includes tool calls, type resolution) |
| **Total run duration** | <X min Y sec> |
| **Aggregate LLM time** | <~X min Y sec> (parallelized across threads) (Testing Agent only) |
| **Total tokens (from Copilot log)** | <N> input + <N> output = **<N> total** |
| **Cached tokens** | <N> (<X%> of input) |
| **Estimated run cost*** | 💸 **$X.XX** (at <model> pricing) — [see Cost Methodology](#cost-methodology) |
| **Avg call duration** | <X sec> (primary calls), varies for tool calls (Testing Agent only) |
| **Longest call** | <X sec> — <context of what it was doing> |
| **Fix iterations** | <N> explicit fix cycles (<note if builds suggest implicit loops>) |
| **Projects in scope** | <N> total, <M> produced tests |
| **get_type_info calls** | <N> (Testing Agent only) |
| **Build count** | <N> (Testing Agent only) |
| **Test executions** | <N> (<M> Success: True, <K> Success: False) |

Notes on the summary table:
- For Testing Agent runs, show both TA log call count (primary agent calls) and Copilot log call count (total API calls including tool/type resolution). The Copilot log count is typically much higher.
- For Copilot Agent Mode runs, there is only one log — show `Total LLM calls` as a single row.
- Omit rows that don't apply (e.g., `get_type_info calls` for Copilot runs, `Aggregate LLM time` for Copilot runs).

### LLM Call Breakdown (<N> primary agent calls from <TA/Copilot> log)

| # | Agent | Duration | Response Length | Notes |
|---|-------|----------|----------------|-------|
| 1 | <AgentName> | <Xs / Xm Xs> | <N> chars | <brief note: prompt analysis, code gen, consolidation, etc.> |
| 2 | ... | ... | ... | ... |

(List each primary LLM call from the agent log. For Testing Agent, these are the "Starting LLM call" entries. For Copilot, these are the conversation turns. Bold the longest call. Include the agent name, duration, response length in characters, and a brief note on what the call was doing.)

### Token Distribution by Call Category (from Copilot log — <N> API calls)

| Category | Calls | Input Tokens | Output Tokens | % of Input | % of Cost |
|----------|-------|-------------|---------------|-----------|-----------|
| Tool/Type resolution (output ≤ 200) | <N> | <N> | <N> | <X%> | <X%> |
| Code generation (output > 500) | <N> | <N> | <N> | <X%> | <X%> |
| Medium tasks (output 201–500) | <N> | <N> | <N> | <X%> | <X%> |
| **Total** | **<N>** | **<N>** | **<N>** | **100%** | **100%** |

(Include this table when token data is available from the Copilot log. Categorize by output token count as a proxy for call complexity. This reveals where tokens and cost concentrate.)

## Efficiency Verdict: <emoji> <one-line summary>

<1-3 sentences: was this run efficient, wasteful, or somewhere in between? Quantify the waste. Reference the dominant cost driver and the potential savings percentage.>

## Waste Findings

### Finding N: <title>

- **What happened:** <describe the waste pattern — cite log evidence>
- **Estimated waste:** <N calls / N tokens / $X.XX / N min that could be saved>
- **Reduction strategy:** <concrete, actionable change>
- **Where to fix:** <`LLM prompt/model` | `Agent orchestrator` | `User workflow` | etc.>

(Repeat for each finding. Max 5. Include dollar amounts when token data is available. Use 💸 emoji prefix on the finding title when the finding has a quantified dollar waste.)

## Cost Estimate 💸

| Category | Calls | Input Tokens | Output Tokens | Current (<model>) | Optimized | Savings |
|----------|-------|-------------|---------------|-------------------|-----------|---------|
| Code generation | <N> | <N> | <N> | $X.XX | $X.XX (<model>) | $0.00 |
| Tool/type resolution | <N> | <N> | <N> | $X.XX | $X.XX (<cheaper model>) | **$X.XX** |
| Medium tasks | <N> | <N> | <N> | $X.XX | $X.XX (<mid-tier model>) | **$X.XX** |
| **Total** | **<N>** | **<N>** | **<N>** | **$X.XX** | **$X.XX** | **$X.XX (Y%)** |

(Include this section whenever token data is available or can be estimated from response lengths. Split input and output tokens per category. Show which model each category would use in the optimized column.)

### Model pricing used (per 1M tokens)

| Model | Input | Output | Recommended for |
|-------|-------|--------|----------------|
| <Premium model> | $X.XX | $X.XX | Code generation only |
| <Mid-tier model> | $X.XX | $X.XX | Prompt analysis, consolidation |
| <Fast/cheap model> | $X.XX | $X.XX | Tool/type resolution, simple lookups |

### At-scale impact

| Metric | Per run | 10 runs/day | Monthly (200 runs) |
|--------|---------|------------|-------------------|
| Current cost | $X.XX | $X.XX | $X.XX |
| Optimized cost | $X.XX | $X.XX | $X.XX |
| **Savings** | **$X.XX** | **$X.XX** | **$X.XX** |

(Include this sub-section to show the compounding impact of per-run savings. Adjust the runs/day and monthly figures to match the team's typical usage.)

## Reduction Strategies Summary

| # | Strategy | Estimated savings | Effort |
|---|----------|------------------|--------|
| 1 | <strategy> | <$X.XX/run (Y%) or time saved> | <Low/Medium/High> |
| 2 | ... | ... | ... |

(Rank by savings descending. Include effort estimate: Low = config change or prompt tweak, Medium = orchestrator logic change, High = architecture change. Use dollar amounts when available, time savings otherwise.)

## Efficiency Metrics vs. Baseline

| Metric | This Run | Baseline (single project, 5–10 files) | Assessment |
|--------|----------|---------------------------------------|------------|
| Primary LLM calls | <N> | 10–25 | <✅/⚠️/❌> |
| Total API calls | <N> | 50–150 | <✅/⚠️/❌> |
| Duration | <X min Y sec> | 5–15 min | <✅/⚠️/❌> |
| Total tokens | <N> | 50K–150K | <✅/⚠️/❌> |
| Builds | <N> | 5–15 | <✅/⚠️/❌> |
| Test executions | <N> (<M> passing) | 5–10 | <✅/⚠️/❌> |
| get_type_info | <N> | 50–100 | <✅/⚠️/❌> |
| Cache hit rate | <X%> | 30–50% | <✅/⚠️/❌> |
| Estimated cost | $X.XX | $5–$20 | <✅/⚠️/❌> |

(Compare key metrics against baselines from the Baseline Expectations table. Use ✅ for within range, ⚠️ for slightly above, ❌ for significantly above. Omit rows that don't apply to the tool type.)

## Cost Methodology

The cost estimates in this report are **approximate and intended for relative comparison**, not billing predictions. Here's how they are calculated:

### Token source

- <State where token counts came from. Options:>
  - **Primary log:** Token data was extracted directly from the <TA/Copilot> log.
  - **Cross-log extraction:** The <TA> log reported `HasUsage: False`. Token data was extracted from the **Copilot diagnostic log** (`copilot-output.log`), which captures `InputTokenCount`, `OutputTokenCount`, and `CachedInputTokenCount` for every LLM API call made through the VS Copilot service layer.
  - **Estimated from response lengths:** No token data was available. Tokens were estimated using ~4 chars per token for output, with input tokens assumed at 3–5× output.
- **Time-window filtering:** Only API calls within the run window (<start>–<end> UTC, from `run-metadata.md`) were included.

### Call categorization

API calls were classified by **output token count** as a proxy for call complexity:

| Category | Output tokens | Rationale |
|----------|--------------|-----------|
| **Tool/Type resolution** | ≤ 200 | Short responses typical of `get_type_info`, tool dispatches, and type lookups |
| **Medium tasks** | 201–500 | Prompt analysis, consolidation, and mid-complexity responses |
| **Code generation** | > 500 | Substantial code output typical of test file generation |

This heuristic is imperfect — some tool calls may produce >200 tokens, and some code gen calls may be short — but it provides a reasonable split for cost estimation.

### Pricing model

Costs are calculated using **publicly available list prices** (as of <month/year>) for the model families. These are **not** the actual prices charged by the Copilot service, which may differ based on enterprise agreements, bundled pricing, or internal cost structures.

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Used for |
|-------|----------------------|----------------------|----------|
| <Premium model> | $X.XX | $X.XX | Current: all calls. Optimized: code generation only |
| <Mid-tier model> | $X.XX | $X.XX | Optimized: medium-complexity tasks |
| <Fast/cheap model> | $X.XX | $X.XX | Optimized: tool/type resolution |

### Formula

```
cost = (input_tokens × input_rate / 1,000,000) + (output_tokens × output_rate / 1,000,000)
```

Applied per category, then summed for totals. Cached tokens use a discounted input rate (typically 10% of standard), but if cache hit rate is 0%, no discount is applied.

### Limitations

- **Not actual billing:** These estimates reflect raw API token costs, not what is charged to the user or organization through Copilot licensing.
- **Copilot log may include extra calls:** The Copilot service log captures all LLM traffic during the time window. Some calls may be from background Copilot features (completions, suggestions) running concurrently, not from the agent itself. This could slightly inflate totals.
- **Category heuristic:** Output-token-based classification is approximate. A more precise approach would correlate each Copilot log entry with the corresponding agent call by request ID, but this cross-referencing is not currently available.
- **Prices change:** Model pricing is volatile. Use the savings percentages (not dollar amounts) for durable comparisons.
```

Save as `llm-efficiency.md` in the artifacts folder and print to console.

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
| **Smart model switching** | Same premium model used for all calls including simple classification/consolidation tasks | 5-15% total cost reduction; use premium for code gen, cheaper model for prompt analysis + insight consolidation |

---

## Cost Estimation Guidance

When token data is available (either from the primary log or cross-referenced from `copilot-output.log`), estimate costs using these reference rates. These are approximate and may change — use them for relative comparisons, not billing predictions.

### Model pricing reference (per 1M tokens, approximate)

| Model | Input | Output | Cached Input |
|-------|-------|--------|-------------|
| Claude Opus 4 | $15.00 | $75.00 | $1.50 |
| Claude Sonnet 4 | $3.00 | $15.00 | $0.30 |
| Claude Haiku 4 | $0.80 | $4.00 | $0.08 |
| GPT-4.1 | $2.00 | $8.00 | $0.50 |

### Smart model switching analysis

When all calls use a single premium model, always check if cheaper models could handle non-code-generation calls. Classify each LLM call into one of these categories:

| Category | Examples | Recommended model tier |
|----------|----------|----------------------|
| **Code generation** | ToolBasedCSharpCodeGenAgent, file creation, test writing | Premium (Opus/GPT-4) — quality matters |
| **Prompt analysis** | FreeFormPromptAnalyzerAgent_Config, _Location | Fast/cheap (Haiku/GPT-4.1) |
| **Consolidation** | InsightConsolidationAgent, summary generation | Mid-tier (Sonnet) or fast/cheap |
| **Fix iteration** | Compilation error fixes, test failure fixes | Mid-tier (Sonnet) — speed > quality |

In the report, include a **Cost Estimate** section (after the LLM Call Breakdown table) showing:
1. **Current estimated cost** — total tokens × premium model rate
2. **Optimized estimated cost** — re-price each call category at its recommended tier
3. **Potential savings** — the delta, both absolute and as a percentage

If token data is unavailable, estimate tokens from response lengths using the heuristic: ~4 chars per token for output, and assume input tokens are ~3-5× output tokens for code generation tasks.

---

## Style

- **Quantify everything** — never say "too many calls" without a number
- **Lead with the verdict** — user wants to know "was this efficient?" immediately
- **Actionable strategies** — every finding must have a concrete "do this to save X"
- **Tables over prose** — use the summary table for scannable results
- **Emoji**: ✅ Efficient, ⚠️ Some waste, ❌ Significant waste, 💸 for cost callouts
- **Always use Unicode emoji** (❌, ✅, ⚠️, 💥) — never use shortcodes like `:x:` or `:boom:` as they don't render in all viewers
