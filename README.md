# Copilot Skills

A collection of prototype [Copilot Skills](https://docs.github.com/en/copilot/copilot-extensions/copilot-skills) that extend GitHub Copilot CLI with specialized capabilities. Skills are organized by category, with each category in its own folder.

> **⚠️ Prototype Notice:** All skills in this repo are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## What Are Skills?

Skills are reusable prompt-driven modules that teach Copilot how to perform specialized tasks. Each skill lives in its own folder and contains a `SKILL.md` file that defines:

- **Trigger phrases** — natural-language prompts that activate the skill
- **Inputs** — what the skill needs from the user (files, folders, parameters)
- **Outputs** — what the skill produces (reports, files, analysis)
- **Detailed instructions** — step-by-step logic Copilot follows to execute the task

## Repo Structure

```
skills/
├── README.md              ← you are here
├── testing/               ← .NET unit testing skills
│   ├── README.md
│   ├── pre-run-analysis/
│   ├── collect-test-copilot-logs/
│   ├── collect-test-testingagent-logs/
│   ├── copilot-test-review/
│   ├── testing-agent-review/
│   ├── unit-test-comparison/
│   └── extract-issues/
└── <future-category>/     ← add new categories here
```

## Skill Categories

### [Testing](./testing/)

Skills for reviewing and comparing .NET unit test output from the .NET Testing Agent and GH Copilot Agent Mode. Includes artifact collection, test quality analysis, side-by-side comparison, and issue extraction.

| Skill | Description |
|-------|-------------|
| [`pre-run-analysis`](./testing/pre-run-analysis/) | Analyze a .NET project before test generation — detects blockers, maps testable surface, sets quality bar |
| [`collect-test-copilot-logs`](./testing/collect-test-copilot-logs/) | Collect test artifacts and Copilot diagnostic logs after a Copilot Agent Mode run |
| [`collect-test-testingagent-logs`](./testing/collect-test-testingagent-logs/) | Collect test artifacts, Copilot logs, and Testing Agent logs after a Testing Agent run |
| [`copilot-test-review`](./testing/copilot-test-review/) | Review and score a GH Copilot Agent Mode test generation run |
| [`testing-agent-review`](./testing/testing-agent-review/) | Review and score a .NET Testing Agent test generation run |
| [`unit-test-comparison`](./testing/unit-test-comparison/) | Side-by-side comparison of Testing Agent vs Copilot Agent Mode |
| [`extract-issues`](./testing/extract-issues/) | Extract, deduplicate, and prioritize issues from review reports into backlogs |

> See the [testing README](./testing/README.md) for the recommended workflow pipeline, detailed skill descriptions, and quick-start examples.

## Using These Skills

### GitHub Copilot CLI

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

### VS Code (GitHub Copilot Chat)

You can use a skill in VS Code by placing the `SKILL.md` content into one of the custom instruction locations that Copilot Chat reads automatically:

1. **As a prompt file** (recommended for on-demand use):
   - Copy the `SKILL.md` file into your repo at `.github/prompts/<skill-name>.prompt.md`
   - In Copilot Chat, reference it with `#prompt:<skill-name>` or click the **+** button to attach it

2. **As an instructions file** (for automatic activation):
   - Copy the `SKILL.md` file into your repo at `.github/instructions/<skill-name>.instructions.md`
   - Copilot will automatically include the instructions when the context matches

> **Note:** VS Code uses the `.github/copilot-instructions.md` file and the `.github/instructions/` folder for custom instructions. Make sure the **"Enable custom instructions"** setting is turned on in VS Code under **Settings → GitHub Copilot → Chat**.

## Adding a New Skill

1. Identify the category folder (e.g., `testing/`, `debugging/`) — create it if it's a new category
2. Create a new skill folder inside the category (e.g., `testing/my-new-skill/`)
3. Add a `SKILL.md` file following the format used by existing skills
4. Define trigger phrases, inputs, outputs, and detailed instructions
5. Update the category `README.md` with a summary of the new skill
6. Update this root `README.md` to include the skill in the category table
