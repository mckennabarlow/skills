---
name: survey-report
description: >
  Analyze survey results from SurveyMonkey (PDF or CSV export) and generate a comprehensive
  stakeholder-ready report with executive summary, per-question findings, optional trend
  comparison across survey waves, optional workplace context via WorkIQ, and a product
  recommendation section. Supports both PDF and CSV SurveyMonkey export formats.
  Use this skill when asked to analyze survey results, summarize a survey, create a survey
  report, or generate findings from a SurveyMonkey export.
  Trigger phrases include "analyze survey", "survey report", "summarize survey results",
  "survey analysis", "survey findings", "analyze survey results", "create survey report".
---

# Survey Report Skill

## How to Use

```
analyze survey at C:\path\to\survey-export.pdf
```

```
survey report from C:\path\to\results.csv — this survey is about feature X and we need to decide whether to ship it
```

```
compare survey waves: C:\path\to\wave1.pdf and C:\path\to\wave2.pdf
```

---

## Purpose

Parse a SurveyMonkey survey export (PDF or CSV), analyze the responses, and produce a
comprehensive markdown report suitable for sharing with stakeholders. The report includes an
executive summary, per-question findings with data tables, an optional product recommendation,
and an optional section enriched with workplace context from WorkIQ.

---

## Inputs

The skill gathers these inputs interactively. Do NOT ask all questions at once — ask them
sequentially as needed.

### Required

- **Survey file path** — Path to the primary SurveyMonkey export (PDF or CSV). If the user
  doesn't provide one, ask for it.
- **Decision context** — A plain-language description of what decision this survey informs.
  Ask: *"What decision or question does this survey inform? This helps me frame the
  recommendation."*

### Optional (ask the user)

- **Previous wave file** — Path to an earlier export of the same survey for trend comparison.
  Ask: *"Do you have an earlier wave of this survey to compare against? If so, provide the
  file path."*
- **WorkIQ enrichment** — Whether to pull related workplace discussions for cross-team context.
  Ask: *"Should I pull in related team discussions from your email, chat, and meetings for
  additional context? If so, what teams or topics should I search for?"*
- **Report author name** — Name for the report byline. Ask only if not obvious from context.
- **Output path** — Where to save the report. Default: user's Documents folder with a
  filename derived from the survey title.

---

## Workflow

Execute the following steps in order.

### Step 1: Parse the survey file

Based on the file extension, choose the appropriate parsing strategy.

#### PDF parsing

Use Python with PyMuPDF (fitz) to extract text from the PDF. Install if needed:

```
pip install pymupdf --quiet
```

Extract text from every page:

```python
import fitz
doc = fitz.open(r'<file_path>')
for i, page in enumerate(doc):
    text = page.get_text()
    print(f'=== PAGE {i+1} ===')
    print(text)
```

#### CSV parsing

Read the CSV directly. SurveyMonkey CSV exports typically have one of two formats:

- **Summary format:** Columns for question, answer choice, percentage, count
- **Individual responses format:** One row per respondent, one column per question

Detect the format by inspecting headers and parse accordingly.

#### What to extract

For each question, extract:

- **Question number and text** (e.g., "Q1: What are you trying to accomplish?")
- **Question type:** multiple choice, rating scale, open-ended, or matrix
- **Answer choices** with their percentages and response counts
- **Total respondents** for that question (Answered / Skipped)
- **Verbatim text responses** for open-ended questions and "Other (please specify)" write-ins

Store all parsed data in a structured format (use the SQL session database with a table like
`survey_questions` and `survey_responses`) for analysis.

#### Data storage schema

```sql
CREATE TABLE survey_meta (
    key TEXT PRIMARY KEY,
    value TEXT
);

CREATE TABLE survey_questions (
    id TEXT PRIMARY KEY,
    question_number INTEGER,
    question_text TEXT,
    question_type TEXT,
    total_answered INTEGER,
    total_skipped INTEGER,
    wave TEXT DEFAULT 'current'
);

CREATE TABLE survey_choices (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    question_id TEXT,
    choice_text TEXT,
    percentage REAL,
    response_count INTEGER,
    wave TEXT DEFAULT 'current',
    FOREIGN KEY (question_id) REFERENCES survey_questions(id)
);

CREATE TABLE survey_verbatims (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    question_id TEXT,
    response_text TEXT,
    response_date TEXT,
    wave TEXT DEFAULT 'current',
    FOREIGN KEY (question_id) REFERENCES survey_questions(id)
);
```

### Step 2: Parse previous wave (if provided)

If the user provided a previous wave file, parse it using the same strategy as Step 1 but
store all records with `wave = 'previous'`.

After both waves are parsed, compute deltas for each multiple-choice question:

```sql
SELECT
    c.question_id,
    c.choice_text,
    c.percentage AS current_pct,
    p.percentage AS previous_pct,
    c.percentage - p.percentage AS delta_pct,
    c.response_count AS current_n,
    p.response_count AS previous_n
FROM survey_choices c
LEFT JOIN survey_choices p
    ON c.question_id = p.question_id
    AND c.choice_text = p.choice_text
    AND p.wave = 'previous'
WHERE c.wave = 'current'
ORDER BY c.question_id, c.percentage DESC;
```

Flag any delta ≥ 5 percentage points as a notable trend to highlight in the report.

### Step 3: Enrich with WorkIQ (if requested)

If the user opted into WorkIQ enrichment, use the `workiq-ask_work_iq` tool to query for
related discussions. Use the decision context provided by the user to form the query.

Example query pattern:

```
What have been the recent discussions, decisions, or feedback about [topic from decision
context]? Include conversations with [teams the user mentioned] about [key themes from
the survey].
```

Store the WorkIQ response for use in the Cross-Team Context section.

### Step 4: Analyze and generate insights

For each question, generate an insight by applying these rules:

#### Multiple choice questions

- **Dominant response:** If one choice has ≥ 50%, call it out as the clear majority signal.
- **Split opinion:** If the top two choices are within 10 percentage points, note the division.
- **Notable minorities:** Call out any choice at 15-30% as a meaningful secondary signal.
- **Negligible responses:** Choices at ≤ 3% can be noted as negligible.
- **Trend signals (if wave comparison available):** Highlight any choice that moved ≥ 5 points
  between waves, noting the direction and magnitude.

#### Open-ended questions

- **Theme extraction:** Group verbatim responses into 3-5 themes. For each theme, provide
  a label, representative quote, and count of responses that fit.
- **Sentiment:** Note if responses are predominantly positive, negative, or mixed.

#### Rating / Likert scale questions

- **Distribution shape:** Note if responses cluster (consensus) or spread (polarization).
- **Combine categories:** For concern/satisfaction scales, combine related levels
  (e.g., "moderately + very concerned = X%") for executive-friendly numbers.

### Step 5: Generate the report

Produce a markdown report with the following structure. Use the actual survey data throughout —
never fabricate numbers.

```markdown
# [Survey Title]
## Survey Results & Product Recommendation Report
**Date:** [today's date] | **Survey respondents:** [n] | **Author:** [name]

---

## Executive Summary

[3-4 paragraph summary of the most important findings, written for a time-pressed
executive. Include:]
- The 3-4 strongest signals from the data, cited with percentages
- How these signals relate to the decision context the user provided
- If wave comparison is available, the most notable trend
- If WorkIQ context is available, how team discussions align with survey findings
- A clear recommendation statement

[If the progressive-flow / training insight applies — i.e., the survey shows users
avoiding action-oriented options due to awareness, trust, or uncertainty — include a
paragraph about how the recommended approach addresses this by creating a natural
on-ramp that trains users toward higher-value workflows.]

---

## Survey Results: Key Findings

### [N]. [Finding Title — a descriptive label, not just the question text]

| [Column headers appropriate to question type] |
|---|---|---|
| [Data rows] |

**Insight:** [Analysis paragraph for this question]

[If wave comparison: include delta column and trend callout]

[Repeat for each question]

---

## Cross-Team Context

[Only include if WorkIQ enrichment was requested]

[Organize by team, summarizing relevant discussions and how they align with survey data]

---

## Product Recommendation

### [One-line recommendation statement]

[Structured recommendation with numbered sub-sections, each tied back to specific
survey data points. Include:]

1. Primary recommendation with supporting data
2. Implementation approach
3. Transition / rollout considerations
4. Naming or labeling proposals (if applicable)
5. Open design & technical questions
6. Open items requiring resolution (table format with Owner and Status columns)

---

## Appendix: Survey Methodology
- Survey title, platform, respondent count, collection period
- Question structure (required vs optional, question types)
- If wave comparison: note both waves with dates and sample sizes
- Limitations (sample size, internal vs external, etc.)
```

### Step 6: Save the report

Save the markdown report to the output path. Default naming convention:

```
[SurveyTitle]-Report.md
```

in the user's Documents folder (e.g., `C:\Users\[username]\Documents\`).

Tell the user where the file was saved.

### Step 7: Generate email preamble

After saving the report, offer to generate a short email preamble for sharing:

> *"Want me to draft a few lines of email preamble for sharing this report with your team?"*

If yes, generate a 3-4 sentence email introduction that:
- Names the survey and sample size
- Summarizes the key takeaway in one sentence
- Points to the attached report for details
- Opens the floor for discussion and offers to meet if there are open questions

Present the preamble to the user (do not save to a file).

---

## Rules

- **Never fabricate data.** Every percentage, count, and quote in the report must come from
  the parsed survey file. If data is missing or ambiguous, note it explicitly.
- **Verbatim responses from a previous wave are valid** if the current wave's export doesn't
  include them — note the sourcing clearly in the report (e.g., "Verbatim responses sourced
  from Wave 1 export; Wave 2 export did not include open-ended text.").
- **Ask questions sequentially**, not all at once. Gather the file path first, parse it,
  then ask about wave comparison, WorkIQ, and decision context.
- **Use the SQL session database** to store parsed survey data. This makes trend analysis
  queries straightforward and keeps the data structured.
- **Finding titles should be descriptive**, not just question numbers. Instead of
  "Question 1 Results," write "User Intent: Understanding First, Acting Second."
- **The executive summary should be opinionated.** It should clearly state what the data
  suggests, not just summarize neutrally. The user is asking for a recommendation.
- **If wave comparison shows notable trends** (≥ 5 point moves), highlight them prominently
  in both the executive summary and the per-question findings with directional language
  (e.g., "increased from 31% to 44%").
- **Combine related response categories** for executive-friendly messaging when it makes the
  insight clearer (e.g., "moderately + very concerned = 46%"). Always show the breakdown
  in the data table.
- **Output format is always markdown.** Do not generate HTML, PDF, or other formats.

---

## SurveyMonkey Export Format Notes

### PDF format quirks
- Questions span multiple pages; the question text appears at the top of each page that
  contains its data.
- "Show responses" links appear in the data but contain no actual text — the verbatim
  responses appear on subsequent pages.
- Write-in responses for "Other (please specify)" are on separate pages with a table
  containing response text and date.
- Page headers repeat the survey title and page number (e.g., "1 / 13").
- Chart data is embedded as text (answer choices, percentages, counts) even though the
  visual chart is also present.
- Some exports omit verbatim response pages entirely — if Q7 shows "Answered: 10" but no
  response text follows, note this gap in the report.

### CSV format quirks
- The first two rows are often headers (survey title + question text).
- Percentage and count columns may use different delimiters.
- Open-ended responses appear as free text in their column.
- Multi-select questions may have one column per choice with 0/1 values.

---

## Important Notes

- This skill is designed for SurveyMonkey exports but may work with similarly structured
  survey data from other platforms.
- The WorkIQ enrichment step requires the WorkIQ MCP tools to be available. If not available,
  skip that step and note it.
- The recommendation section requires decision context from the user. If they don't provide
  it, ask before generating — don't guess at what decision the survey informs.
- For wave comparison, both files must be from the same survey (same questions). If questions
  don't match between waves, note the discrepancy and only compare matching questions.
