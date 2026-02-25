# GitHub Copilot Skills & Agents

A collection of prototype [GitHub Copilot Skills](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) and [Custom Agents](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-custom-agents) that extend GitHub Copilot CLI with specialized capabilities. Skills are organized by category under the `skills/` folder, and agents live in `agents/`.

> **⚠️ Prototype Notice:** All skills and agents in this repo are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## What Are Skills and Agents?

**Skills** are reusable prompt-driven modules that teach Copilot how to perform specialized tasks. Each skill lives in its own folder and contains a `SKILL.md` file that defines:

- **Trigger phrases** — natural-language prompts that activate the skill
- **Inputs** — what the skill needs from the user (files, folders, parameters)
- **Outputs** — what the skill produces (reports, files, analysis)
- **Detailed instructions** — step-by-step logic Copilot follows to execute the task

**Agents** are custom agent profiles that orchestrate multiple skills into structured pipelines. Each agent is defined in a `.agent.md` file and specifies the workflow, parameters, and execution order.

## Repo Structure

```
├── README.md                              ← you are here
├── agents/                                ← custom agent definitions
│   └── unit-test-eval-pipeline.agent.md
├── skills/
│   ├── testing/                           ← .NET unit testing skills
│   │   ├── README.md
│   │   ├── collect-test-copilot-logs/
│   │   ├── collect-test-testingagent-logs/
│   │   ├── copilot-test-review/
│   │   ├── create-ado-workitem/
│   │   ├── extract-issues/
│   │   ├── llm-efficiency/
│   │   ├── pre-run-analysis/
│   │   ├── run-diagnosis/
│   │   ├── testing-agent-review/
│   │   └── unit-test-comparison/
│   ├── productivity/                      ← general-purpose DevOps & workflow skills
│   │   ├── README.md
│   │   └── file-ado-workitem/
│   ├── workiq-communications/             ← return-to-work briefing skills (WorkIQ)
│   │   ├── catchup-ic/
│   │   └── catchup-manager/
│   └── <future-category>/                 ← add new categories here
```

## Agents

### [`unit-test-eval-pipeline`](./agents/unit-test-eval-pipeline.agent.md)

Orchestrates a structured analysis pipeline over .NET unit test generation runs from GH Copilot Agent Mode and/or the .NET Testing Agent. Supports quick and full modes, optional sources, and parallel execution where safe.

**Pipeline phases:** Collect → Diagnose → Review → Compare + LLM Efficiency

**Skills used:** `collect-test-copilot-logs`, `collect-test-testingagent-logs`, `run-diagnosis`, `copilot-test-review`, `testing-agent-review`, `unit-test-comparison`, `llm-efficiency`

## Skill Categories

### [Testing](./skills/testing/)

Skills for reviewing and comparing .NET unit test output from the .NET Testing Agent and GH Copilot Agent Mode. Includes artifact collection, test quality analysis, side-by-side comparison, and issue extraction.

| Skill | Description |
|-------|-------------|
| [`collect-test-copilot-logs`](./skills/testing/collect-test-copilot-logs/) | Collect test artifacts and Copilot diagnostic logs after a Copilot Agent Mode run |
| [`collect-test-testingagent-logs`](./skills/testing/collect-test-testingagent-logs/) | Collect test artifacts, Copilot logs, and Testing Agent logs after a Testing Agent run |
| [`copilot-test-review`](./skills/testing/copilot-test-review/) | Review and score a GH Copilot Agent Mode test generation run |
| [`create-ado-workitem`](./skills/testing/create-ado-workitem/) | Create Azure DevOps work items from run diagnosis reports |
| [`extract-issues`](./skills/testing/extract-issues/) | Extract, deduplicate, and prioritize issues from review reports into backlogs |
| [`llm-efficiency`](./skills/testing/llm-efficiency/) | Analyze LLM usage — find wasted calls, token bloat, stalls, and get reduction strategies in ≤3 tool-call rounds |
| [`pre-run-analysis`](./skills/testing/pre-run-analysis/) | Analyze a .NET project before test generation — detects blockers, maps testable surface, sets quality bar |
| [`run-diagnosis`](./skills/testing/run-diagnosis/) | Fast triage of any test run — classifies outcome, extracts key events, identifies root cause in ≤3 tool-call rounds |
| [`testing-agent-review`](./skills/testing/testing-agent-review/) | Review and score a .NET Testing Agent test generation run |
| [`unit-test-comparison`](./skills/testing/unit-test-comparison/) | Side-by-side comparison of Testing Agent vs Copilot Agent Mode |

> See the [testing README](./skills/testing/README.md) for the recommended workflow pipeline, detailed skill descriptions, and quick-start examples.

### [Productivity](./skills/productivity/)

General-purpose skills for streamlining DevOps workflows — filing work items, managing backlogs, and automating repetitive tasks. These skills are team-agnostic and work across any project.

| Skill | Description |
|-------|-------------|
| [`file-ado-workitem`](./skills/productivity/file-ado-workitem/) | Create Azure DevOps work items from any file or inline description — auto-discovers project, area path, and work item type |

**Prerequisites:** Requires [Azure DevOps MCP server](https://github.com/microsoft/azure-devops-mcp) to be connected.

> See the [productivity README](./skills/productivity/README.md) for detailed skill descriptions and quick-start examples.

### [WorkIQ Communications](./skills/workiq-communications/)

Return-to-work briefing skills powered by [WorkIQ](https://github.com/microsoft/work-iq-mcp) (Microsoft 365 Copilot). When you come back from time away, these skills scan your email, Teams messages, meetings, meeting recordings, and documents to build a comprehensive catch-up briefing with actionable follow-ups. Two variants are available depending on your role:

| Skill | Description |
|-------|-------------|
| [`catchup-ic`](./skills/workiq-communications/catchup-ic/) | For individual contributors — focuses on your own work items, deadlines, manager analysis, top collaborator summaries, and a consolidated briefing with follow-ups |
| [`catchup-manager`](./skills/workiq-communications/catchup-manager/) | For managers — auto-discovers direct reports and generates per-person deep dives with 1:1 agendas, manager analysis, top collaborator summaries, and a consolidated briefing with follow-ups |

**Prerequisites:** Requires [WorkIQ MCP server](https://github.com/microsoft/work-iq-mcp) to be connected and functional.

## Installation

### Installing Skills (GitHub Copilot CLI)

The Copilot CLI has built-in skill management via the `/skills` slash command.

1. **Add a skill from a local folder:**

   ```
   /skills add <path-to-skill-folder>
   ```

   For example, to install `testing-agent-review` after cloning this repo:

   ```
   /skills add C:\path\to\skills\testing\testing-agent-review
   ```

   This copies the `SKILL.md` into `~/.copilot/skills/<skill-name>/`.

2. **List installed skills:**

   ```
   /skills list
   ```

3. **Get info about an installed skill:**

   ```
   /skills info testing-agent-review
   ```

4. **Remove a skill:**

   ```
   /skills remove testing-agent-review
   ```

Once installed, simply use one of the skill's trigger phrases in your conversation and Copilot will automatically activate it.

### Installing Agents

Agents can be installed at three levels:

| Level | Location | Scope |
|-------|----------|-------|
| **User-level** (global) | `~/.copilot/agents/` | All projects |
| **Repo-level** | `.github/agents/` in a repo | That repo only |
| **Org/Enterprise** | `/agents/` in `.github-private` repo | All org projects |

To install an agent globally, copy the `.agent.md` file to `~/.copilot/agents/`. To use it, launch `copilot` and either use `/agent` to browse available agents, or reference the agent by name in your prompt.

### VS Code (GitHub Copilot Chat)

You can use a skill in VS Code by placing the `SKILL.md` content into one of the custom instruction locations that Copilot Chat reads automatically:

1. **As a prompt file** (recommended for on-demand use):
   - Copy the `SKILL.md` file into your repo at `.github/prompts/<skill-name>.prompt.md`
   - In Copilot Chat, reference it with `#prompt:<skill-name>` or click the **+** button to attach it

2. **As an instructions file** (for automatic activation):
   - Copy the `SKILL.md` file into your repo at `.github/instructions/<skill-name>.instructions.md`
   - Copilot will automatically include the instructions when the context matches

> **Note:** VS Code uses the `.github/copilot-instructions.md` file and the `.github/instructions/` folder for custom instructions. Make sure the **"Enable custom instructions"** setting is turned on in VS Code under **Settings → GitHub Copilot → Chat**.

## Adding New Skills or Agents

### Adding a Skill

1. Identify the category folder (e.g., `skills/testing/`, `skills/debugging/`) — create it if it's a new category
2. Create a new skill folder inside the category (e.g., `skills/testing/my-new-skill/`)
3. Add a `SKILL.md` file following the format used by existing skills
4. Define trigger phrases, inputs, outputs, and detailed instructions
5. Update the category `README.md` with a summary of the new skill
6. Update this root `README.md` to include the skill in the category table

### Adding an Agent

1. Create a new `.agent.md` file in `agents/`
2. Include YAML frontmatter with `name` and `description`
3. Define the agent's pipeline, parameters, and which skills it orchestrates
4. Update this root `README.md` to include the agent in the Agents section
