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

If the user already specified an issue number, use that issue. Otherwise, list all issues found with their number, ROI level, and title, **plus an "all" option**, and ask the user to pick one. Example listing:

```
Found 2 issues in the diagnosis:
  1. (ROI: HIGH) Generated csproj references packages missing from Directory.Packages.props
  2. (ROI: MEDIUM) EF/ASP.NET version mismatch in Directory.Packages.props
  all. File ONE consolidated work item covering all issues
Which issue number would you like to file? (1, 2, or all)
```

**Only one work item is filed per area path per invocation.** If the user picks a single issue and wants to file others, they should invoke the skill again. If the user picks "all", a single consolidated work item is created (per area path) that covers every issue — see Step 4b.

### Step 3: Collect area paths

Ask the user which area path(s) to file the work item(s) under. They may provide one or more. If they don't specify any, use the default: `DevDiv\NET Tools Prague\Code Testing Agent`.

### Step 4: Extract the Title and Priority

#### Single-issue path (user picked a specific issue number)

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

#### All-issues path (user picked "all") — Step 4b

When the user chooses "all":

1. **Generate a suggested summary title.** Read the `## Outcome` line (e.g., "❌ No tests generated") and all issue titles to craft a concise summary. The title should capture the overall result and the most impactful root cause(s). Example:
   - Outcome: `❌ No tests generated`, Issues: `file_search loop`, `8-minute stall`, `Workspace path null`
   - Suggested title: *"Copilot agent produced zero tests — stuck in file_search loop with 8-min stall"*

2. **Present the suggested title to the user and ask them to confirm or edit it** before proceeding. Example:
   ```
   Suggested title: "Copilot agent produced zero tests — stuck in file_search loop with 8-min stall"
   Use this title, or provide your own?
   ```

3. **Priority:** Use the **highest** ROI level across all issues (e.g., if any issue is HIGH, priority = 1).

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

After **all** work items are created successfully, add a logged-issue block to the source markdown file, inserted just before the `## Run Metadata` heading. If previous `> **Bug for Issue #...` lines already exist, append the new lines after them rather than creating duplicates.

The format is:

```
> - ✅ **Bug for Issue #<N>:** [#<ID>](https://devdiv.visualstudio.com/DevDiv/_workitems?id=<ID>)
>   - <Title>
>   - `<Area Path>`
```

If the same issue was filed to multiple area paths, add one entry per area path:

```
> - ✅ **Bug for Issue #1:** [#2726734](https://devdiv.visualstudio.com/DevDiv/_workitems?id=2726734)
>   - VS ReportService crash terminates run
>   - `DevDiv\NET Tools Prague\Code Testing Agent`
> - ✅ **Bug for Issue #1:** [#2726773](https://devdiv.visualstudio.com/DevDiv/_workitems?id=2726773)
>   - VS ReportService crash terminates run
>   - `DevDiv\VS Core\Extensibility\ServiceHub`
```

If the user picked "all", use `All Issues` instead of a specific issue number:

```
> - ✅ **Bug for All Issues:** [#2726734](https://devdiv.visualstudio.com/DevDiv/_workitems?id=2726734)
>   - Copilot agent produced zero tests — stuck in file_search loop with 8-min stall
>   - `DevDiv\NET Tools Prague\Code Testing Agent`
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
- Only one issue (or "all" as a consolidated item) is filed per invocation. If the user wants to file individual issues separately, they must invoke the skill once per issue.
- The Description field should be left empty. All diagnosis content goes into the Discussion as a markdown comment.
- If a work item creation fails, report the error, skip that area path, and continue with remaining area paths. Only update the markdown with successfully created work items.
- When appending to existing `> **Bug for Issue #...` lines, add new lines after the last one — do not duplicate existing entries.

---

## Important Notes

- The work item will be created under your ADO identity (Created By is automatic).
- Multiple area paths can be provided to file the same issue across different teams in one invocation.
- This skill is currently hardcoded for the DevDiv project. A future version will make the project configurable.
