# Testing Skills

A collection of [GitHub Copilot Skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) for .NET testing workflows.

> **⚠️ Prototype Notice:** All skills in this category are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## Skills

| Skill | Description |
|-------|-------------|
| [`code-coverage-analysis`](./code-coverage-analysis/) | One-shot code coverage analysis for .NET projects — runs tests, generates HTML report via ReportGenerator, and produces a markdown insights report with metrics, risk analysis, and prioritized recommendations |
| [`playwright-test-review`](./playwright-test-review/) | Review Playwright E2E tests for quality and reliability — audits selectors, wait patterns, Page Object Models, assertions, test isolation, and diagnostic instrumentation. Produces a scored reliability report with prioritized issues |
| [`playwright-accessibility-audit`](./playwright-accessibility-audit/) | Automated WCAG 2.1 AA accessibility audit using Playwright + axe-core. Crawls app pages, runs Deque.AxeCore.Playwright analysis, and produces a prioritized compliance report with remediation guidance and Copilot fix prompts |
