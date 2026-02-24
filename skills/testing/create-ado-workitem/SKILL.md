---
name: create-ado-workitem
description: >
  Create Azure DevOps User Story work items from a run-diagnosis markdown file.
  Accepts an optional issue number to file a specific issue; if not provided, lists all issues
  and asks the user to pick one. Only one issue is filed per invocation.
  Supports creating work items across multiple area paths in a single invocation.
  Extracts the selected issue's title for the work item title, posts the full markdown as a
  Discussion comment, and maps the issue's ROI level to ADO Priority. After creation, updates
  the source markdown file with a bullet list of all logged issues showing title, area path, and URL.
  Trigger phrases include "create ADO work item", "file ADO bug",
  "create work item from diagnosis", "log ADO issue", "file work item".
---

# Create ADO Work Item from Diagnosis Skill

## How to Use

Invoke this skill by asking Copilot to create an ADO work item from a run-diagnosis markdown file.

### Example prompts

```
Create ADO work item from diagnosis
```

```
File ADO bug from C:\path\to\run-diagnosis.md
```

```
Log ADO issue from this diagnosis
```

---

## Purpose

Automate creation of Azure DevOps User Story work items from run-diagnosis markdown files produced by the `run-diagnosis` skill. Extracts structured data from the markdown, creates the work item via the ADO MCP tool, and writes the work item link back into the source file.

---

## Dependencies

- **Azure DevOps MCP Server** _(internal only)_ — This skill requires the Azure DevOps MCP server to create work items and add comments. Install it from: [microsoft/azure-devops-mcp](https://github.com/microsoft/azure-devops-mcp/)

---

## Inputs

- **Markdown file path** — Ask the user for the path to the run-diagnosis markdown file. If not provided, search the current working directory recursively for a file named `run-diagnosis.md`.
- **Issue number** — Ask the user which issue number to file (e.g., `1`, `2`). The number corresponds to the `### Issue N` headings in the markdown. If the user doesn't specify, list all issues found in the file with their titles and ROI levels, and ask them to pick one. **Only one issue is filed per invocation.**
- **Area paths** — Ask the user which area path(s) to file work items under. The user may provide one or multiple area paths. A separate work item will be created for **each** area path, all with the same title, priority, tag, and discussion content.

---

## Hardcoded Values

| Field | Value |
|-------|-------|
| **Project** | `DevDiv` |
| **Work Item Type** | `User Story` |
| **Iteration Path** | `DevDiv` |
| **Tag** | `TestingAgentSkillFind` |
| **State** | `Proposed` |

**Default Area Path** (used if the user doesn't specify one): `DevDiv\NET Tools Prague\Code Testing Agent`

---

## Workflow

Execute the following steps in order.

### Step 1: Locate and read the markdown file

Ask the user for the path to the run-diagnosis markdown file. If they don't provide one, search the current directory recursively for `run-diagnosis.md`. Read the entire file contents.

### Step 2: Select the issue to file

Parse all issue headings in the markdown. These follow the pattern:

```
### Issue N (ROI: HIGH|MEDIUM|LOW): <issue title text>
```

If the user already specified an issue number, use that issue. Otherwise, list all issues found with their number, ROI level, and title, and ask the user to pick one. Example listing:

```
Found 2 issues in the diagnosis:
  1. (ROI: HIGH) Generated csproj references packages missing from Directory.Packages.props
  2. (ROI: MEDIUM) EF/ASP.NET version mismatch in Directory.Packages.props
Which issue number would you like to file?
```

**Only one issue is filed per invocation.** If the user wants to file multiple issues, they should invoke the skill again for each one.

### Step 3: Collect area paths

Ask the user which area path(s) to file the work item(s) under. They may provide one or more. If they don't specify any, use the default: `DevDiv\NET Tools Prague\Code Testing Agent`.

### Step 4: Extract the Title and Priority

From the selected issue heading, extract:

- **Title:** The text after `ROI: HIGH|MEDIUM|LOW): ` — use it verbatim as the work item title.
  - Example heading: `### Issue 1 (ROI: HIGH): VS ReportService crash terminates run before test generation`
  - Title: `VS ReportService crash terminates run before test generation`

- **Priority:** Map the issue's ROI level to ADO Priority:

| ROI Level | ADO Priority |
|-----------|-------------|
| HIGH | 1 |
| MEDIUM | 2 |
| LOW | 3 |

If no ROI level is found, default to Priority **2**.

### Step 5: Create work items (one per area path)

For **each** area path provided by the user, call the `ado-wit_create_work_item` MCP tool with:

- **project:** `DevDiv`
- **workItemType:** `User Story`
- **fields:**
  - `System.Title` — the extracted title from Step 4
  - `System.AreaPath` — the current area path
  - `System.IterationPath` — `DevDiv`
  - `System.Tags` — `TestingAgentSkillFind`
  - `System.State` — `Proposed`
  - `Microsoft.VSTS.Common.Priority` — the mapped priority from Step 4

Do **not** set `System.Description` — leave it empty.

### Step 5b: Add the full diagnosis as a Discussion comment

After **each** work item is created, call the `ado-wit_add_work_item_comment` MCP tool with:

- **project:** `DevDiv`
- **workItemId:** the ID returned from Step 5
- **comment:** the **entire** markdown file contents
- **format:** `markdown`

### Step 6: Update the source markdown file

After **all** work items are created successfully, add a bullet list of logged issues to the source markdown file, inserted just before the `## Run Metadata` heading. If a previous `> **Issues logged:**` block already exists, append the new bullets to it rather than creating a duplicate block.

The format is:

```
> **Issues logged:**
> - ✅ <Title> | `<Area Path>` | [#<ID>](https://devdiv.visualstudio.com/DevDiv/_workitems?id=<ID>)
> - ✅ <Title> | `<Area Path>` | [#<ID>](https://devdiv.visualstudio.com/DevDiv/_workitems?id=<ID>)
```

Example with two work items:

```
> **Issues logged:**
> - ✅ VS ReportService crash terminates run | `DevDiv\NET Tools Prague\Code Testing Agent` | [#2726734](https://devdiv.visualstudio.com/DevDiv/_workitems?id=2726734)
> - ✅ VS ReportService crash terminates run | `DevDiv\VS Core\Extensibility\ServiceHub` | [#2726773](https://devdiv.visualstudio.com/DevDiv/_workitems?id=2726773)
```

Use the `edit` tool to insert or append this block. Ensure a blank line separates it from `## Run Metadata`.

### Step 7: Report results

Tell the user:
- A table of all created work items with: ID, Title, Area Path, Priority, and URL
- Confirm the source markdown was updated with the logged issues list

---

## Rules

- Do not modify the markdown file contents other than inserting/appending the issues logged block in Step 6.
- If the markdown file has no issue headings, use the top-level heading (e.g., `# Run Diagnosis — ...`) as a fallback title.
- Only one issue is filed per invocation. If the user wants to file multiple issues, they must invoke the skill separately for each.
- The Description field should be left empty. All diagnosis content goes into the Discussion as a markdown comment.
- If a work item creation fails, report the error, skip that area path, and continue with remaining area paths. Only update the markdown with successfully created work items.
- When appending to an existing `> **Issues logged:**` block, add new bullets at the end — do not duplicate the header line.

---

## Important Notes

- The work item will be created under your ADO identity (Created By is automatic).
- Multiple area paths can be provided to file the same issue across different teams in one invocation.
- This skill is currently hardcoded for the DevDiv project. A future version will make the project configurable.
