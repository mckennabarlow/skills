---
name: survey-report
description: >
  Analyze survey results from SurveyMonkey (PDF or CSV export) and generate a comprehensive
  stakeholder-ready report suite: full report with evidence appendix, brief insights summary,
  optional deep-dive topic reports, and follow-up contact lists. Supports multi-format output
  (Markdown, PDF, HTML, email), charts, configurable severity indicators, data verification,
  and optional trend comparison across survey waves.
  Use this skill when asked to analyze survey results, summarize a survey, create a survey
  report, or generate findings from a SurveyMonkey export.
  Trigger phrases include "analyze survey", "survey report", "summarize survey results",
  "survey analysis", "survey findings", "analyze survey results", "create survey report",
  "deep-dive", "contact report", "follow-up contacts".
---

# Survey Report Skill

## How to Use

```
analyze survey at C:\path\to\survey-export.csv
```

```
survey report from C:\path\to\results.csv — this survey is about feature X and we need to decide whether to ship it
```

```
compare survey waves: C:\path\to\wave1.pdf and C:\path\to\wave2.pdf
```

If the repository contains a `survey-config.yaml`, the skill reads it automatically for column
mappings, thresholds, and output settings. Otherwise, it auto-detects the CSV structure.

---

## Purpose

Parse a SurveyMonkey survey export (PDF or CSV), analyze the responses, and produce a
comprehensive report suite suitable for sharing with stakeholders. The suite includes:

- **Full report** — Detailed findings with data tables, qualitative themes, pain points,
  recommendations, and an evidence appendix (Markdown + PDF)
- **Insights summary** — A 2-minute executive scan with key metrics and indicators
  (Markdown + PDF + HTML + email-ready EML)
- **Charts** — PNG visualizations of key metrics (matplotlib)
- **Deep-dive reports** (optional) — Focused analysis on high-mention themes
- **Contact reports** (optional) — Follow-up lists with draft outreach emails
- **Product recommendation** (optional) — Decision-context-driven recommendation section
- **Cross-team context** (optional) — WorkIQ-enriched workplace discussion summary

---

## Inputs

The skill gathers these inputs interactively. Do NOT ask all questions at once — ask them
sequentially as needed.

### Required

- **Survey file path** — Path to the primary SurveyMonkey export (PDF or CSV). If the user
  doesn't provide one, ask for it.

### Optional (ask the user)

- **Decision context** — A plain-language description of what decision this survey informs.
  If provided, the report includes a Product Recommendation section.
  Ask: *"What decision or question does this survey inform? (Skip if you just want findings.)"*
- **Previous wave file** — Path to an earlier export of the same survey for trend comparison.
  Ask: *"Do you have an earlier wave of this survey to compare against? If so, provide the
  file path."*
- **WorkIQ enrichment** — Whether to pull related workplace discussions for cross-team context.
  Ask: *"Should I pull in related team discussions from your email, chat, and meetings for
  additional context? If so, what teams or topics should I search for?"*
- **Report author name** — Name for the report byline. Ask only if not obvious from context.
- **Output path** — Where to save reports. Default: `Reports/[NAME]/` in the working
  directory, or `[SurveyTitle]-Report.md` in the user's Documents folder if no config exists.

---

## Configuration (survey-config.yaml)

If a `survey-config.yaml` file exists at the repository root, read it to determine survey
parameters, column mappings, thresholds, and what optional outputs to generate. If absent,
auto-detect the CSV structure from headers.

### Full Schema

```yaml
# ─── Global Settings ───────────────────────────────────────────────
settings:
  # Output root directory (relative to repo root)
  output_dir: Reports

  # Required tools (ensure available on PATH or as Python packages)
  tools:
    pandoc: true   # For PDF/HTML generation
    python: true   # For charts, EML, verification scripts

  # Severity indicator thresholds (used across all reports)
  indicators:
    # For Likert averages (1–5 scale)
    likert:
      green: { min_avg: 3.5, min_positive_pct: 55 }
      yellow: { min_avg: 3.2, min_positive_pct: 45 }
      # Below yellow → red

    # For satisfaction / agreement percentages
    satisfaction:
      green_min_pct: 60
      yellow_min_pct: 40
      # Below 40% → red

    # For qualitative theme mention counts
    theme_mentions:
      red_min: 40
      yellow_min: 15
      # Below 15 → green

  # Minimum contactable respondents to generate a contact report
  contact_report_min_respondents: 3

  # Deep-dive auto-detection: themes ≥ this threshold are candidates
  deep_dive_mention_threshold: 15

# ─── Surveys ───────────────────────────────────────────────────────
surveys:
  - name: Example Survey           # Internal name
    display_name: "My Product Survey"  # Used in report titles
    csv_file: "survey-export.csv"  # Path relative to repo root
    output_folder: MyProduct       # → Reports/MyProduct/

    # Column mapping — tells analysis how to interpret the CSV
    # SurveyMonkey CSVs have two header rows: row 1 = question, row 2 = sub-label
    columns:
      respondent_id: 0             # Column index for respondent ID
      start_date: 2                # Column index for start date
      end_date: 3                  # Column index for end date
      email: 25                    # Column index for email (for contact reports)

      # Structured fields (ratings, multiple-choice)
      structured:
        - { col: 10, label: "Satisfaction", type: single-choice }
        - { col: 11, label: "Frequency", type: single-choice }
        - { col: 12, label: "Quality Rating", type: likert-5 }
        # Supported types: single-choice, multi-select, likert-5, open-text

      # Open-ended text columns
      open_text:
        - { col: 18, label: "Feedback" }
        - { col: 19, label: "Suggestions" }

    # Deep-dive topics: 'auto' to detect from themes, or list specific ones
    deep_dives: auto

    # Contact reports
    contact_reports:
      enabled: true
      include_email_drafts: true

    # Custom sections — keyword-driven report sections
    custom_sections:
      - name: "AI Feedback"
        description: "Feedback specifically about AI features"
        keywords: ["copilot", "AI", "artificial intelligence", "machine learning"]
```

When a config exists, `[NAME]` throughout this skill refers to the survey's `output_folder`.

---

## Output Contract

### Output Location

Write all output for each survey into: `Reports/[NAME]/`

Where `[NAME]` comes from `surveys[].output_folder` in `survey-config.yaml`, or a
slug derived from the survey title if no config exists. Create the folder if needed.

### Required Output Files (6 per survey)

For each survey, generate these SIX core files:

| # | File | Purpose |
|---|------|---------|
| 1 | `[NAME]-full-report.md` | Detailed findings with evidence appendix |
| 2 | `[NAME]-full-report.pdf` | PDF of full report |
| 3 | `[NAME]-insights.md` | 2-minute executive summary |
| 4 | `[NAME]-insights.pdf` | PDF of insights |
| 5 | `[NAME]-insights.html` | Browser-viewable insights |
| 6 | `[NAME]-insights.eml` | Email-ready insights (Outlook-compatible) |

### PDF Generation

Generate PDFs using Pandoc after Markdown files are finalized:

```
pandoc Reports/[NAME]/[NAME]-full-report.md -o Reports/[NAME]/[NAME]-full-report.pdf
pandoc Reports/[NAME]/[NAME]-insights.md -o Reports/[NAME]/[NAME]-insights.pdf
```

If `pdflatex` is unavailable, try `--pdf-engine=weasyprint` or fall back to Python-based
PDF generators (`markdown-pdf`, `md2pdf`). If PDF generation fails, fix and retry — do not skip.

### HTML Generation

```
pandoc Reports/[NAME]/[NAME]-insights.md -o Reports/[NAME]/[NAME]-insights.html --standalone
```

HTML must render cleanly in a browser. Keep styling minimal (inline or small `<style>` block).

### EML Generation

Generate an Outlook-compatible `.eml` file embedding the insights HTML:

- Use Python's `email.mime` package
- MIME type: `multipart/alternative`
- Include both `text/plain` (from insights markdown) and `text/html` (from insights HTML)
- Subject: `"[Display Name] survey – key insights"`
- From/To: use placeholders if unknown

### Chart Generation

Generate charts as PNG images using Python (matplotlib recommended):

- Save inside `Reports/[NAME]/`
- Use deterministic filenames (e.g., `satisfaction-distribution.png`)
- Reference in Markdown with width constraints:

```markdown
![Chart](./chart-name.png){ width=90% }
```

For HTML `<img>` tags, use: `style="max-width:90%; height:auto;"`

If chart generation fails, provide a Markdown table instead and explain why.

---

## Severity Indicators

Use emoji indicators consistently across all reports:

- 🔴 Bad / High risk
- 🟡 Mixed / Needs attention
- 🟢 Good / Strong positive

### Configurable Thresholds

Use thresholds from `survey-config.yaml` if available, otherwise use these defaults:

| Context | 🟢 Green | 🟡 Yellow | 🔴 Red |
|---------|----------|-----------|--------|
| Likert average (1–5) | avg ≥ 3.5 AND positive% ≥ 55% | avg ≥ 3.2 AND positive% ≥ 45% | Below yellow |
| Satisfaction / agreement % | ≥ 60% | 40–59% | < 40% |
| Theme mention count | < 15 | 15–39 | ≥ 40 |

### Mandatory Legend Rule

**Every table or list that uses emoji indicators MUST include a legend** immediately above it
defining the thresholds in use. Examples:

*Indicators: 🟢 ≥ 60% positive · 🟡 40–59% · 🔴 < 40%*

*Severity: 🔴 ≥ 40 mentions · 🟡 15–39 · 🟢 < 15*

---

## Table Sorting

When a table has a key numeric value (e.g., Positive %, Vote %, Mentions), sort rows by that
value to highlight the most critical items first:

- Satisfaction/positive metrics: sort **ascending** (worst at top)
- Vote counts/mentions: sort **descending** (most at top)

---

## Date and Survey Identification

- Use the **actual date range** from CSV data (Start Date / End Date columns), not "today"
- Format: `"Feb 24 – Mar 4, 2026"`
- Use `display_name` from config as the survey name (not generic labels)
- End reports with a data source line:
  *Data source: [Display Name], N responses, [date range].*

---

## Workflow

Execute the following steps in order.

### Step 1: Load configuration

If `survey-config.yaml` exists, read it for column mappings, thresholds, and output settings.
If absent, proceed to parsing and auto-detect structure from CSV headers.

### Step 2: Parse the survey file

Based on the file extension, choose the appropriate parsing strategy.

#### PDF parsing

Use Python with PyMuPDF (fitz) to extract text. Install if needed: `pip install pymupdf --quiet`

**Important:** Write parsing code to a temporary `.py` script file and execute it (not inline
`python -c "..."`). Inline commands may trigger permission prompts in autopilot mode.

```python
# _parse_survey.py
import fitz
doc = fitz.open(r'<file_path>')
for i, page in enumerate(doc):
    print(f'=== PAGE {i+1} ===')
    print(page.get_text())
```

Run with `python _parse_survey.py`, then delete the script.

#### CSV parsing

Read the CSV directly. SurveyMonkey CSV exports typically have:

- **Summary format:** Columns for question, answer choice, percentage, count
- **Individual responses format:** One row per respondent, one column per question
  (often with 2 header rows: row 1 = question text, row 2 = sub-label)

If config provides column mappings, use them. Otherwise, detect format by inspecting headers.

#### What to extract

For each question, extract:

- **Question number and text**
- **Question type:** multiple choice, rating scale, open-ended, or matrix
- **Answer choices** with percentages and response counts
- **Total respondents** (Answered / Skipped)
- **Verbatim text responses** for open-ended questions and "Other" write-ins

#### Data storage

Store parsed data in a structured format. If the SQL session database is available:

```sql
CREATE TABLE survey_meta (key TEXT PRIMARY KEY, value TEXT);

CREATE TABLE survey_questions (
    id TEXT PRIMARY KEY, question_number INTEGER, question_text TEXT,
    question_type TEXT, total_answered INTEGER, total_skipped INTEGER,
    wave TEXT DEFAULT 'current'
);

CREATE TABLE survey_choices (
    id INTEGER PRIMARY KEY AUTOINCREMENT, question_id TEXT, choice_text TEXT,
    percentage REAL, response_count INTEGER, wave TEXT DEFAULT 'current'
);

CREATE TABLE survey_verbatims (
    id INTEGER PRIMARY KEY AUTOINCREMENT, question_id TEXT, response_text TEXT,
    response_date TEXT, wave TEXT DEFAULT 'current'
);
```

### Step 3: Parse previous wave (if provided)

Parse using the same strategy as Step 2, storing records with `wave = 'previous'`.
Compute deltas for each question. Flag any delta ≥ 5 percentage points as a notable trend.

### Step 4: Enrich with WorkIQ (if requested)

If the user opted in and the `workiq-ask_work_iq` tool is available, query for related
discussions using the decision context. Store the response for the Cross-Team Context section.

### Step 5: Quantitative analysis

For each structured question, compute from the raw data:

- **Counts and percentages** for each response choice (label the denominator N)
- **Positive / Neutral / Negative** groupings with percentages
- **Averages** for Likert scales (map to 1–5 numeric values)
- **Cross-tabulations** where relevant (e.g., satisfaction × reliability)

Apply insight rules:

- **Dominant response:** ≥ 50% → clear majority signal
- **Split opinion:** Top two within 10 points → note division
- **Notable minorities:** 15–30% → meaningful secondary signal
- **Negligible:** ≤ 3% → note as negligible
- **Trend signals:** ≥ 5 point move between waves → highlight direction and magnitude
- **Rating distributions:** Note clustering (consensus) vs. spread (polarization)
- **Combine categories** for executive-friendly numbers (e.g., "moderately + very concerned = 46%")

State exclusions for any questions with missing data.

If NPS exists (0–10 scale): Promoters = 9–10, Passives = 7–8, Detractors = 0–6,
NPS = %Promoters − %Detractors.

### Step 6: Qualitative analysis

Analyze open-ended responses to extract themes:

- **6–12 themes maximum** — group by keyword matching and semantic similarity
- **Count occurrences** per theme (keyword-based search across all open-text columns)
- **Include 1–2 verbatim quotes** per theme with row number references
- **Categorize** themes into: Bugs, UX friction, Missing features, Performance issues,
  Pricing/value (or categories appropriate to the survey topic)
- **Identify sub-themes** within each major theme for potential deep-dive analysis
- **Cross-tabulate** themes with structured fields where insightful
  (e.g., "67% of reliability complainers also rated satisfaction as 'Dissatisfied'")

### Step 7: Custom sections

If the config includes `custom_sections`, add a dedicated section in the report for each:
- Search open-ended responses for the specified keywords
- Summarize the feedback
- Include 2–3 representative quotes with row references
- Provide specific recommended actions

### Step 8: Synthesis

Provide:
- **Top 3 pain points** — the most impactful negative findings, with evidence
- **Top 3 most-loved aspects** — what users value most
- **Ranked recommendations** — ordered by impact, with supporting data references
- **Follow-up survey questions** — suggestions for future surveys

### Step 9: Generate report files

Write the full report and insights Markdown files following the templates below.

### Step 10: Data verification (MANDATORY)

After writing report Markdown files, **independently recompute every numeric claim** using a
Python script that reads the raw CSV:

1. Recompute all counts, percentages, averages, and distributions
2. Extract all numeric claims from the report Markdown files
3. Compare each claim: print PASS/FAIL for each check
4. **Tolerance:** values within ±1 percentage point or ±0.05 on averages are acceptable
5. Fix all discrepancies in BOTH the full report and insights files
6. Re-verify until all checks PASS

**Do NOT generate PDFs until all verification checks PASS.**

### Step 11: Generate derived formats

After verification passes:
1. Generate PDFs (full report + insights) via pandoc
2. Generate HTML (insights) via pandoc --standalone
3. Generate EML (insights) via Python email package
4. Generate charts as PNG files via matplotlib

### Step 12: Deep-dive reports (optional)

If `deep_dives` is configured (or if themes exceed the mention threshold):

1. Present themes above the threshold as deep-dive candidates with mention counts
2. Ask the user which to generate (or "all" / "skip")
3. For each selected topic, generate a deep-dive report (see template below)

### Step 13: Contact reports (optional)

If `contact_reports.enabled: true` in config:

1. For each deep-dive theme, count respondents with email addresses who mentioned the theme
2. Present candidates with contactable counts
3. For themes meeting the minimum threshold, generate contact reports (see template below)
4. If `include_email_drafts: true`, include personalized draft emails

### Step 14: Finalize

Present the final file inventory to the user. Offer to generate an email preamble for sharing.
Clean up any temporary Python scripts.

---

## Report Templates

### Full Report Structure

File: `Reports/[NAME]/[NAME]-full-report.md`

```markdown
# [Display Name] Product Feedback Report

## Executive Summary
[3-4 paragraphs: strongest signals with percentages, relation to decision context,
notable trends, clear recommendation statement. Be opinionated.]

## Dataset Overview
[Table: survey platform, collection period, total responses, structured vs open-ended
question counts, email response count]

## Quantitative Findings
[Per-question sections with data tables, charts, insight paragraphs.
Use severity indicators with legends. Sort tables by key metric.]

## Qualitative Themes
[Theme table with mention counts and severity indicators.
Per-theme sections with quotes and row references.]

## [Custom Sections — one per entry in custom_sections config]

## Cross-Team Context
[Only if WorkIQ enrichment was used]

## Top Pain Points
[Top 3, with severity indicators and evidence references]

## Most Loved Aspects
[Top 3, with evidence references]

## Recommendations
[Ranked list with severity indicators, supporting data, and actionable specifics.
Include expected impact and urgency for each.]

## Product Recommendation
[Only if decision context was provided. Structured recommendation
tied to specific data points.]

## Appendix A — Evidence Index
[Table: Ref | Row | Column | Verbatim Excerpt]
```

**Evidence rules:** No orphan references. Every `[N]` citation in the report must appear in
Appendix A with Reference ID, Row/Response ID, Column name, and verbatim excerpt.
Do NOT fabricate quotes.

### Insights Structure

File: `Reports/[NAME]/[NAME]-insights.md`

Purpose: shareable, scannable in under ~2 minutes. Must reference the full report filename.

```markdown
# [Display Name] Key Insights

## At a Glance
[Summary table of key metrics with 🔴 🟡 🟢 indicators and legend]

## Top Findings (5–10 bullets)
[Use 🔴 🟡 🟢 indicators per bullet]

## [Key Custom Section highlights — if configured and noteworthy]

## Recommended Actions (3–7 bullets)
[Each action: expected impact and urgency]

## Where to find details
[Reference: [NAME]-full-report.md (and PDF)]
```

Do NOT include long tables or the evidence appendix in the insights file.

### Deep-Dive Template

File: `Reports/[NAME]/[topic-slug].md`

```markdown
# [NAME] — [Topic Name] Deep Dive

> [Vote count] votes ([rank] improvement area, [%] of [N] respondents)
> + [mention count] open-ended mentions · [Date range]

---

## Sub-Themes
[Table with severity legend. 5–10 sub-themes sorted by mentions descending.]

## Most Telling Quotes
[3–6 diverse verbatim quotes with Row [N] references]

## Key Patterns
[3–5 recurring patterns with evidence counts]

## User Impact
[Table: Impact | Evidence — productivity loss, workarounds, trust erosion]

## Recommended Actions
### Immediate
[🔴 items with data justification]
### High Priority
[🟡 items with data justification]

---
*Source: [NAME]-full-report.md · [N] responses · [Date range]*
```

### Contact Report Template

File: `Reports/[NAME]/TODO-[area-slug]-follow-up-contacts.md`

**CRITICAL PRIVACY RULE: NEVER include email addresses in the report.**
Reference respondents by row number only. Readers look up emails in the original CSV.

```markdown
# Follow-Up Contact List — [NAME] [Feedback Area]

> Respondents who mentioned [description] **and** provided an email address.
> Email addresses are in the original CSV — look up by row number.

## Summary
**[N] contactable respondents** reported [area]-related issues.

| Row | Improvement Vote | Key Quote (truncated to ~120 chars) |
|---:|---|---|

## Details

#### Row [N]
- **[Structured field]:** [value]
- **[Open-text label]:**
  > [Full verbatim quote]

**Draft email:**
[Personalized follow-up — see email draft rules below]

---
[Repeat for each contactable respondent]
```

Only generate if ≥ `contact_report_min_respondents` (default: 3) match the theme.

### Email Draft Rules

For each contactable respondent, generate a personalized outreach email:

**Format:**
```
Hello, thanks for completing the [Display Name]. We really appreciate your feedback.

In the free text reply you mentioned:
[Respondent's verbatim quote — most relevant portion]

[1-2 sentences acknowledging their specific experience]

[1-3 specific follow-up questions tailored to their response]
```

**Rules:**
- Professional but warm tone, under 150 words
- Reference their actual words — no generic placeholders
- For crashes/restarts: ask for Report a Problem feedback item with logs
- For feature gaps: ask about specific scenario and project type
- For vague feedback: skip with note "*Skipped — feedback too vague*"
- For already contacted (quote starts with `[sent]`): "*Skipped — already sent*"
- For hostile feedback: remain professional and empathetic
- Do NOT include email addresses or make promises about fixes/timelines
- Plain text only — no HTML formatting or blockquote markers (`> `)

---

## Rules

### Data Integrity
- **Never fabricate data.** Every percentage, count, and quote must come from the parsed
  survey file. Note missing or ambiguous data explicitly.
- **Never generate PDFs before data verification passes.**
- **Verbatim responses from a previous wave** are valid if the current wave omits them —
  note the sourcing clearly.

### Privacy
- **Never include email addresses** in any report file. Reference respondents by row number.

### Presentation
- **Always include severity legends** on every table using emoji indicators.
- **Always sort tables** by key metric value (worst-first for satisfaction, most-first for counts).
- **Always use actual date ranges** from CSV data, not approximations or "today's date."
- **Use `display_name`** from config, not generic labels like "Weekly Pulse."
- **End with a data source line**, not a generic footer.
- **Finding titles should be descriptive** — "Flow Speed Is Not the Problem" not "Question 6."
- **The executive summary should be opinionated** — state what the data suggests, not just
  summarize neutrally.

### Interaction
- **Ask questions sequentially**, not all at once. Parse the file first, then ask about
  optional features.
- **Use the SQL session database** when available for structured data storage and queries.
- **Combine related response categories** for executive-friendly messaging (e.g.,
  "moderately + very concerned = 46%"). Always show the breakdown in the data table.
- **If wave comparison shows notable trends** (≥ 5 point moves), highlight them prominently.

### Efficiency
- Write Python code to temporary `.py` script files (not inline). Delete after use.
- For multiple surveys, process in parallel where possible.
- Don't ask permission for obvious steps (e.g., fixing verification failures).
- DO ask permission for judgment calls (e.g., which themes to deep-dive).

---

## SurveyMonkey Export Format Notes

### PDF format quirks
- Questions span multiple pages; question text appears at the top of each page.
- "Show responses" links contain no text — verbatims appear on subsequent pages.
- "Other (please specify)" write-ins are on separate pages with response text + date.
- Page headers repeat survey title and page number (e.g., "1 / 13").
- Chart data is embedded as text alongside visual charts.
- Some exports omit verbatim pages — if a question shows "Answered: 10" but no text follows,
  note the gap.

### CSV format quirks
- First two rows are often headers (row 1 = question text, row 2 = sub-label).
- Open-ended responses appear as free text in their column.
- Multi-select questions may have one column per choice with 0/1 values.
- Percentage and count columns may use different delimiters.

---

## Important Notes

- This skill is designed for SurveyMonkey exports but works with similarly structured
  survey data from other platforms.
- The WorkIQ enrichment step requires the WorkIQ MCP tools. If unavailable, skip and note it.
- The Product Recommendation section requires decision context. If not provided, generate
  findings without it — don't guess at what decision the survey informs.
- For wave comparison, both files must be from the same survey (same questions). If questions
  don't match, note the discrepancy and only compare matching questions.
- The config file is optional. Without it, the skill auto-detects CSV structure and uses
  default thresholds. With it, the skill is more precise and generates richer output.
