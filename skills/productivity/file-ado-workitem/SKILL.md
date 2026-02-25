---
name: file-ado-workitem
description: >
  Create Azure DevOps work items from any markdown or text file. Auto-detects ADO project
  and team from MCP tools, caches settings for reuse, and supports any work item type.
  Works with any input format — extracts issues using structured parsing or LLM-assisted
  extraction. Requires the Azure DevOps MCP server to be installed.
  Trigger phrases include "file ADO work item", "create ADO bug", "log ADO issue",
  "file work item", "create work item from file", "file bug in ADO".
---

# File ADO Work Item

## How to Use

Invoke this skill by asking Copilot to file an ADO work item. Everything except the title or source file is optional — defaults come from your saved config.

### Usage pattern

```
file [type] [priority] [in area\path] [from file]: [title or description]
```

### Example prompts

**Quick one-liners (inline description):**

```
file bug: null reference in AuthService login flow
```

```
file P1 bug: build pipeline is completely broken
```

```
file user story in DevDiv\NuGet: Package restore fails on .NET 9 projects
```

```
file task in DevDiv\CoreFxTools: Update CI pipeline to use latest SDK
```

**From a file:**

```
file work item from ./diagnosis.md
```

```
file bug from C:\logs\test-report.md
```

```
create ADO bug from C:\path\to\notes.md in DevDiv\Foundry Tooling
```

**Config management:**

```
reset my ADO defaults
```

```
change my default area path
```

```
show my ADO config
```

---

## Purpose

Create Azure DevOps work items from any source — markdown files, text files, or a plain description provided by the user. Automatically discovers the user's ADO project and team via MCP tools, caches configuration for future runs, and supports flexible input formats.

---

## Inputs

- **Source** — A file path or inline description from the user. If a file is provided, read it. If not, ask the user what the work item should be about.
- **Issue selection** — If the file contains multiple discrete issues, list them and let the user pick one (or "all" for a consolidated item).
- **Area path override** — The user may optionally specify one or more area paths. If not provided, use the cached default.

---

## Workflow

Execute the following steps in order.

### Step 0: Verify ADO MCP is available

Call `ado-core_list_projects` with `top: 1`. If the tool is not available or returns an error, **stop immediately** and tell the user:

> "This skill requires the Azure DevOps MCP tools to be installed and configured. Please install the ADO MCP server and try again."

Do not proceed with any further steps.

### Step 1: Load or create configuration

Look for a configuration file in this order (first found wins):

1. **Repo-level:** `.copilot/file-ado-workitem.json` in the current git repository root
2. **User-level:** `~/.copilot/file-ado-workitem.json`

If a config file is found and contains all required fields (`project`, `defaultAreaPath`), load it and skip to Step 5.

If no config file exists or required fields are missing, proceed to Step 2.

### Step 2: Discover ADO project

**First**, try to auto-detect from the git remote URL. Run `git remote -v` and check if any remote matches an ADO pattern:

- `https://dev.azure.com/{org}/{project}/_git/{repo}`
- `https://{org}.visualstudio.com/{project}/_git/{repo}`
- `git@ssh.dev.azure.com:v3/{org}/{project}/{repo}`

If a match is found, extract the **org**, **project**, and **orgUrl** and confirm with the user:

> "I detected you're working in the **{project}** project at **{orgUrl}**. Is that where you want to file the work item?"

If the user confirms, use those values. If not, or if no ADO remote was found, fall back to discovery:

Call `ado-core_list_projects` to get all projects the user has access to. Present the project names as a pick list and ask the user to choose one. Use the selected project's `url` field to derive the org URL.

### Step 3: Discover area path

Call `ado-wit_my_work_items` with the selected project and `type: myactivity` to get the user's recent work items. Then call `ado-wit_get_work_items_batch_by_ids` with the returned IDs and `fields: ["System.AreaPath"]` to extract the area paths.

Deduplicate and rank area paths by frequency. Present them as a pick list sorted by most-used first:

```
Which area path?
  1. DevDiv\NET Tools Prague\Code Testing Agent (used 12 times)
  2. DevDiv\NET Developer Experience\VS Testing (used 2 times)
  3. DevDiv\VS Core\VS AI\VS Copilot (used 2 times)
  4. Different path...
```

If `ado-wit_my_work_items` returns no results (new user with no work item history), fall back to calling `ado-core_list_project_teams` with `mine: true` and presenting team names instead.

If the user prefers to type an area path directly, allow that as well.

### Step 3b: Discover work item type

Ask the user which work item type they typically file. Present common options: Bug, User Story, Task, Feature. Do **not** assume a default — let the user choose. Their choice will be saved in the config for future runs.

### Step 4: Save configuration

Ask the user:

> "Save these defaults for this repo (your teammates will get them too) or just for you?"
> - **This repo** — saves to `.copilot/file-ado-workitem.json` in the repo root
> - **Just me** — saves to `~/.copilot/file-ado-workitem.json`

Write the configuration file:

```json
{
  "orgUrl": "https://devdiv.visualstudio.com",
  "project": "DevDiv",
  "defaultAreaPath": "DevDiv\\Team Name",
  "iterationPath": "DevDiv",
  "workItemType": "Bug"
}
```

- **`iterationPath`** defaults to the project name (the project root iteration).
- **`workItemType`** is whatever the user selected in Step 3b.

Tell the user: *"Configuration saved. You can edit this file anytime or override values per invocation."*

### Step 5: Read and parse the input

If the user provided a **file path**, read the entire file.

If the user provided an **inline description** (no file), use that text directly — skip to Step 6 with the description as the title and body.

#### Parsing issues from a file

Try structured parsing first. Look for headings that represent discrete issues. Common patterns:

- `### Issue N (ROI: HIGH|MEDIUM|LOW): <title>` (run-diagnosis format)
- `### Issue N: <title>`
- `## <title>` or `### <title>` (any heading-per-issue format)
- Numbered lists: `1. <title>`

If multiple issues are found, list them with their titles and any severity/priority indicators, plus an "all" option:

```
Found 3 issues in the file:
  1. AuthService null reference on login
  2. Database timeout under load
  3. Missing input validation on /api/users
  all. File one consolidated work item covering all issues
Which issue would you like to file? (1, 2, 3, or all)
```

**Only one work item is filed per area path per invocation.** If the user wants to file multiple issues separately, they must invoke the skill again.

If no structured issues are found, treat the entire file as a single issue:
- Use the first heading (`# ...` or `## ...`) as the title
- If no heading exists, ask the user to provide a title

#### Extracting priority

Look for priority indicators in the content:

| Pattern found | ADO Priority |
|---------------|-------------|
| `ROI: HIGH`, `Priority: 1`, `Severity: Critical`, `P1`, `🔴` | 1 |
| `ROI: MEDIUM`, `Priority: 2`, `Severity: Major`, `P2`, `🟡` | 2 |
| `ROI: LOW`, `Priority: 3`, `Severity: Minor`, `P3`, `🟢` | 3 |
| Nothing found | 2 (default) |

When "all" is selected, use the **highest** priority across all issues.

#### "All" consolidation

When the user picks "all":
1. Generate a suggested summary title from the issue titles
2. Present it to the user for confirmation or editing before proceeding

### Step 6: Confirm and collect overrides

Before creating the work item, present a summary and ask the user to confirm:

```
Ready to create:
  Title:     AuthService null reference on login
  Type:      User Story
  Project:   DevDiv
  Area Path: DevDiv\Team Name
  Priority:  2

Create this work item? (or tell me what to change)
```

The user can override any value at this point, including:
- **Work item type** — Bug, Task, User Story, etc.
- **Area path** — one or multiple (a separate work item is created per area path)
- **Priority**
- **Title**
- **Tags** — optional, none by default

### Step 7: Create work items

For **each** area path, call `ado-wit_create_work_item` with:

- **project:** from config
- **workItemType:** from config or user override
- **fields:**
  - `System.Title` — the extracted or user-provided title
  - `System.AreaPath` — the current area path
  - `System.IterationPath` — from config (defaults to project root)
  - `Microsoft.VSTS.Common.Priority` — the mapped priority
  - `System.Tags` — only if the user specified tags
  - `System.State` — do **not** set (let ADO use the default for new items)

Do **not** set `System.Description` — leave it empty.

### Step 8: Add source content as a Discussion comment

After **each** work item is created, call `ado-wit_add_work_item_comment` with:

- **project:** from config
- **workItemId:** the ID returned from Step 7
- **comment:** the full file contents (if a file was provided) or the user's description
- **format:** `markdown`

### Step 9: Optionally update the source file

If the input was a file (not inline text), ask the user:

> "Want me to add the work item link(s) back into the source file?"

If yes, append a section at the end of the file:

```markdown

## Filed Work Items

- ✅ **Work Item:** [#<ID>](<orgUrl>/<project>/_workitems?id=<ID>)
  - <Title>
  - `<Area Path>`
```

If the user says no, skip this step.

### Step 10: Report results

Tell the user:
- A table of all created work items with: ID, Title, Area Path, Priority, Type, and URL
- Whether the source file was updated

---

## Rules

- **Step 0 is mandatory.** If the ADO MCP tools are not available, the skill cannot function. Stop early with a clear message.
- Only one issue (or "all" as a consolidated item) is filed per invocation.
- The Description field should be left empty. All source content goes into the Discussion as a markdown comment.
- If a work item creation fails, report the error, skip that area path, and continue with remaining area paths.
- Never overwrite or destructively modify the user's source file. The "Filed Work Items" section is always appended, never inserted in the middle of existing content.
- If the config file exists but the user wants to change a value for this run, allow the override without modifying the config file. Only update the config if the user explicitly asks to change their defaults.

---

## Configuration Reference

The config file (`file-ado-workitem.json`) supports these fields:

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `orgUrl` | Yes | (auto-detected) | ADO organization URL |
| `project` | Yes | (auto-detected) | ADO project name |
| `defaultAreaPath` | Yes | (user-selected) | Default area path for new work items |
| `iterationPath` | No | Same as project name | Iteration path for new work items |
| `workItemType` | No | `User Story` | Default work item type |
| `defaultTags` | No | (none) | Comma-separated tags to apply by default |

---

## Important Notes

- The work item will be created under your ADO identity (Created By is automatic).
- Multiple area paths can be provided to file the same issue across different teams in one invocation.
- Config files can be checked into the repo for team-wide defaults or kept in the user home directory for personal defaults.
- This skill works with any text or markdown input — it is not tied to any specific tool or report format.
