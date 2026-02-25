# Productivity Skills

A collection of [GitHub Copilot Skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) for streamlining day-to-day development workflows — filing work items, managing backlogs, and automating repetitive DevOps tasks. These skills extend GitHub Copilot CLI with capabilities that work across any project and team.

> **⚠️ Prototype Notice:** All skills in this category are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## Prerequisites

- **Azure DevOps MCP server** must be connected and functional for ADO-related skills

## Skills

| Skill | Description |
|-------|-------------|
| [`file-ado-workitem`](./file-ado-workitem/) | Create Azure DevOps work items from any file or inline description — auto-discovers project, area path, and work item type on first run, caches config for future use |

## Skill Details

### [`file-ado-workitem`](./file-ado-workitem/)

**Create Azure DevOps work items from any source — files, reports, or inline descriptions.**

This is a generic, team-agnostic skill for filing ADO work items. Unlike the testing-specific `create-ado-workitem` skill, this one works with any input format and auto-discovers your ADO configuration. It:

- **Auto-discovers your ADO setup** — detects project from git remote or lists available projects; discovers area paths from your work item history ranked by usage frequency
- **Caches configuration** — saves your defaults to a JSON config file (repo-level or user-level) so subsequent runs skip all setup questions
- **Works with any input** — markdown files, text files, or just an inline description in your prompt
- **Flexible overrides** — every value (type, area path, priority, tags) can be overridden per invocation at the confirmation step or inline in the prompt
- **Posts full context** — attaches the source file contents as a Discussion comment on the work item

#### Usage Pattern

```
file [type] [priority] [in area\path] [from file]: [title or description]
```

#### Quick Start

```
file bug: null reference in AuthService login flow
```

```
file bug from C:\logs\test-report.md
```

```
file P1 bug in DevDiv\NuGet: Package restore fails on .NET 9 projects
```

#### Configuration

On first run, the skill walks you through setup and saves a config file:

- **Repo-level** (`.copilot/file-ado-workitem.json`) — shared with your team
- **User-level** (`~/.copilot/file-ado-workitem.json`) — personal defaults

To manage your config:

```
show my ADO config
reset my ADO defaults
change my default area path
```
