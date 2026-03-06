# GitHub Copilot Skills

A collection of [GitHub Copilot Skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) that extend GitHub Copilot CLI with specialized capabilities.

## Skills

| Skill | Description |
|-------|-------------|
| [`code-coverage-analysis`](./skills/testing/code-coverage-analysis/) | One-shot code coverage analysis for .NET projects — runs tests, generates HTML report via ReportGenerator, and produces a markdown insights report with metrics, risk analysis, and prioritized recommendations |

## Installation

### GitHub Copilot CLI

```
/skills add <path-to-skill-folder>
```

For example, after cloning this repo:

```
/skills add C:\path\to\skills\testing\code-coverage-analysis
```

### VS Code (GitHub Copilot Chat)

Copy the `skill.md` file into your repo at `.github/prompts/code-coverage-analysis.prompt.md` and reference it with `#prompt:code-coverage-analysis` in Copilot Chat.
