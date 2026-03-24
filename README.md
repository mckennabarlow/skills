# GitHub Copilot Skills

A collection of [GitHub Copilot Skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) that extend GitHub Copilot CLI with specialized capabilities.

## Skills

| Skill | Description |
|-------|-------------|
| [`code-coverage-analysis`](./skills/testing/code-coverage-analysis/) | One-shot code coverage analysis for .NET projects — runs tests, generates HTML report via ReportGenerator, and produces a markdown insights report with metrics, risk analysis, and prioritized recommendations |
| [`survey-report`](./skills/productivity/survey-report/) | Analyze survey results from SurveyMonkey (PDF or CSV export) and generate a comprehensive report suite: full report with evidence appendix, brief insights summary, charts, optional deep-dive topic reports, and follow-up contact lists. Supports multi-format output (Markdown, PDF, HTML, email), configurable severity indicators, mandatory data verification, wave comparison, and optional WorkIQ enrichment |

## Installation

### GitHub Copilot CLI

```
/skills add <path-to-skill-folder>
```

For example, after cloning this repo:

```
/skills add C:\path\to\skills\testing\code-coverage-analysis
/skills add C:\path\to\skills\productivity\survey-report
```

### VS Code (GitHub Copilot Chat)

Copy the `SKILL.md` file into your repo at `.github/prompts/<skill-name>.prompt.md` and reference it with `#prompt:<skill-name>` in Copilot Chat.
