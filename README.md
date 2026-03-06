# GitHub Copilot Skills

A collection of [GitHub Copilot Skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) that extend GitHub Copilot CLI with specialized capabilities.

## Skills

| Skill | Description |
|-------|-------------|
| [`code-coverage-analysis`](./skills/testing/code-coverage-analysis/) | One-shot code coverage analysis for .NET projects — runs tests, generates HTML report via ReportGenerator, and produces a markdown insights report with metrics, risk analysis, and prioritized recommendations |
| [`survey-report`](./skills/productivity/survey-report/) | Analyze survey results from SurveyMonkey (PDF or CSV export) and generate a comprehensive stakeholder-ready report with executive summary, per-question findings, optional trend comparison across survey waves, optional workplace context via WorkIQ, and a product recommendation section |

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
