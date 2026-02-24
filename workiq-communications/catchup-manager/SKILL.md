---
name: catchup-manager
description: "Generates a structured return-to-work briefing using workplace activity data (email, chat, meetings/recordings, and documents) via Work IQ. Produces a per-direct executive summary of what the manager likely missed and what requires action."
---

# Manager Catch-Up Skill (Copilot CLI + Work IQ)

> **If you're a manager with direct reports, you're in the right place.**  
> If you're an individual contributor, use the [catchup-ic](../catchup-ic/SKILL.md) skill instead — it's tailored for your own work, teammates, and collaborators.

## Prerequisites

This skill requires WorkIQ to be connected and functional.

## Purpose

**What this skill does:**  
When a manager returns from time away (vacation, leave, conference, etc.), this skill
builds a comprehensive catch-up briefing by scanning their Microsoft 365 activity —
email, Teams/chat messages, meetings, meeting recordings, and shared documents — for
the period they were out. It synthesizes everything into a set of actionable markdown
reports so the manager can quickly get up to speed.

**How to use it:**  
Tell the assistant to "catch me up" or "run manager-catchup" and provide your first day
away (e.g., "I was out starting 2/16"). The skill handles the rest:

1. **Asks you three things:** how long you were out, where to save the reports, and
   (optionally) any focus areas like specific projects or keywords.
2. **Auto-discovers your org context** — your name, direct reports, manager, and top
   cross-team collaborators — from the org chart and activity data. No manual entry needed.
3. **Queries Work IQ** across the time window to gather activity, decisions, risks,
   blockers, and follow-ups for each person.
4. **Generates markdown report files:**
   - One file per direct report (deep dive + 1:1 agenda)
   - One file for your manager (what's top of mind + 1:1 agenda)
   - One file for your top collaborators (high-level summaries)
   - One main briefing that links to all the above, with a consolidated follow-up
     list, meeting recordings to watch, risk summary, and a first-day-back plan.

**What you get:**  
A folder of `.md` files you can read, share, or use to prep for 1:1s. The main briefing
gives you the full picture in 5 minutes; the individual files let you go deeper on
any person.

**Requirements:**  
- Work IQ access (the `workiq-ask_work_iq` tool must be available)
- The user must be a manager with direct reports discoverable in the org chart

**Key features:**
- Fully automatic org discovery — no hardcoded names or paths
- Remembers your preferred save location across runs
- Filters meeting recordings to only those where decisions were made or you were mentioned
- Includes date-verification guardrails to avoid stale data
- Consolidated follow-up list across all reports so nothing falls through the cracks

## Required Inputs

1. Time window (ask the user)
   - Number of days away OR start date

2. Direct reports
   - **Auto-discover** by querying Work IQ: "Who are my direct reports?"
   - Do NOT ask the user to list them manually.
   - Confirm the discovered list with the user before proceeding.

3. Report output location (ask the user)
   - Ask: "Where would you like me to save the reports?"
   - Store the user's chosen **base path** in the SQL session_state table (key: `catchup_report_path`).
   - On subsequent runs, retrieve the stored path and offer it as the default:
     "Last time you used `<path>`. Use the same location?"
   - If no prior path exists, ask without a default.
   - Inside the base path, create a subfolder named `Week of YYYY-MM-DD` using the Monday of the
     briefing's end-date week. Write all report files into that subfolder.

4. Optional focus areas (ask the user)
   - Priority projects
   - Teams channels
   - Keywords

Confirm timeframe in plain language before running queries.

---

## Data Source
Use Work IQ (the `workiq-ask_work_iq` tool) for all workplace activity data:

- Email (inbox, sent, threads)
- Chat messages (Teams or equivalent)
- Meetings and recordings
- Documents and comments (shared files, wikis)

Work IQ queries the user's Microsoft 365 data automatically.
Always include the timeframe in queries.

---

## Workflow Overview

Run in this order:

1. Global situational awareness
2. Per-direct deep analysis (repeat for each direct)
3. Manager (skip-level up) analysis
4. Top collaborator analysis
5. Cross-team synthesis
6. Manager re-entry plan
7. Output structured briefing

---

## 1. Global Situational Awareness

Meetings overview
workiq ask -q "Summarize my meetings from the last N days. Highlight major decisions, action items assigned to me, and anything time sensitive."

Meeting recordings to watch
workiq ask -q "In the last N days, list meeting recordings where either (1) a decision was made or (2) I was directly mentioned by name. For each, include the meeting title, date, who attended, what decision was made or why I was mentioned, and a link to the recording if available. Do not include routine standups or meetings where nothing consequential happened. Rank by impact."

Email overview
workiq ask -q "Summarize important email threads from the last N days that need my response, decision, or awareness. Identify who is waiting on me."

Teams overview
workiq ask -q "Summarize my Teams messages from the last N days. Focus on mentions of me, decisions, unresolved questions, and escalations."

Documents overview
workiq ask -q "List important documents created or significantly updated in the last N days that I have access to. Prioritize items tied to my team or projects and summarize what changed."

---

## 2. Per-Direct Deep Analysis (repeat for each direct)

For each <DirectName>, run all queries A–F.

A) Activity and accomplishments
workiq ask -q "In the last N days, summarize <DirectName>'s work activity across email, Teams, meetings, and documents. Include deliverables, progress, and notable work."

B) Decisions and authority exercised
workiq ask -q "In the last N days, what decisions did <DirectName> make or participate in. Identify scope, impact, and whether the decision changed priorities, delivery, or direction."

C) Risks, blockers, and escalation signals
workiq ask -q "In the last N days, what risks, blockers, or escalation signals involved <DirectName>. Include delivery risk, dependency risk, or organizational tension."

D) What they need from me now
workiq ask -q "In the last N days, what does <DirectName> need from me. Include approvals, prioritization, feedback, or unblocking support."

E) What I likely missed that matters most
workiq ask -q "Based on <DirectName>'s activity in the last N days across email, Teams, meetings, and documents, identify the most important things I likely missed while away. Focus only on consequential changes, major decisions, risks, priority shifts, or relationship dynamics. Rank the top 3 to 5 by impact."

F) Suggested 1:1 agenda
workiq ask -q "Create a focused 1:1 agenda for me and <DirectName> based on the last N days. Provide 6 to 10 discussion bullets including at least two coaching or support questions."

---

## 3. Manager (Skip-Level Up) Analysis

Auto-discover the user's manager by querying Work IQ: "Who is my manager?"
Do NOT ask the user — look it up from the org chart.

For <ManagerName>, run all queries:

A) What's been top of mind
workiq ask -q "In the last N days, what has <ManagerName> been focused on based on emails, Teams messages, meetings, and documents? Identify top priorities, recurring themes, and any shifts in direction or emphasis."

B) Decisions and direction set
workiq ask -q "In the last N days, what decisions did <ManagerName> make or communicate? Include org-level direction, priority changes, or expectations set."

C) What they need from me / follow-ups
workiq ask -q "In the last N days, what does <ManagerName> need from me or expect me to follow up on? Include action items, requests, open questions, or commitments I may have missed while away."

D) Suggested 1:1 agenda
workiq ask -q "Create a focused 1:1 agenda for me and <ManagerName> for my first week back, based on the last N days. Provide 4 to 6 discussion bullets. Focus on: catching up on direction, confirming my priorities align, and surfacing anything time-sensitive."

---

## 4. Top Collaborator Analysis

Auto-discover the user's top collaborators from the past 30 days by querying Work IQ:
workiq ask -q "Who are my top collaborators over the past 30 days based on email, Teams messages, meetings, and document co-authoring? Exclude my direct reports and my manager. Return up to 10 people, ranked by interaction frequency. For each, include their name and role/title if available."

Filter out anyone already covered as a direct report or manager.
Present the list to the user and ask them to confirm which collaborators to include in the report.

For each confirmed collaborator <CollabName>, run a high-level summary query:

A) Activity summary
workiq ask -q "In the last N days, summarize my interactions with <CollabName> across email, Teams, meetings, and documents. What were the key topics, decisions, open items, or things I should follow up on? Keep it high-level."

Compile all confirmed collaborators into a single report file.

---

## 5. Cross-Team Synthesis

Identify patterns across directs:

workiq ask -q "Across my team’s activity in the last N days, identify cross-team themes, shared risks, systemic blockers, or emerging organizational patterns. Focus on anything affecting multiple people or the broader roadmap."

---

## 6. Manager Re-Entry Prioritization

Use gathered findings to produce:

- Top priorities for first day back
- Decisions that cannot wait
- Conversations that must happen immediately
- Work that can safely wait

---

## 7. Output Format

Write reports as markdown files to the user's chosen output location.

### Generation order

Generate output files in this order:
1. Per-direct report files (one per direct report)
2. Manager report file
3. Top collaborators report file
4. **Main briefing file (generated LAST)** — this file synthesizes and links to all the above.
   By generating it last, the briefing can incorporate all findings, follow-ups, and action items
   from the individual reports and present a complete, linked overview.

### File structure

Create the following files in the output directory:

1. **`catchup-<FirstName>-<LastName>-YYYY-MM-DD.md`** — One file per direct report.
   Contains: What I Likely Missed, Key Accomplishments, Decisions Made Without Me,
   Risks and Blockers, Needs From Me Now, and Suggested 1:1 Agenda.

2. **`catchup-<FirstName>-<LastName>-Manager-YYYY-MM-DD.md`** — One file for the user's manager.
   Contains: What's Been Top of Mind, Decisions and Direction Set, What They Need From Me / Follow-Ups,
   and Suggested 1:1 Agenda.

3. **`catchup-Top-Collaborators-YYYY-MM-DD.md`** — One file for all top collaborators (non-directs, non-manager).
   Contains a high-level summary per collaborator: key topics, decisions, open items, and follow-ups.
   Collaborators are auto-discovered from the past 30 days and confirmed by the user before querying.

4. **`catchup-briefing-YYYY-MM-DD.md`** — The full briefing (main report, generated last).
   Title the report using the user's name: `# <UserName>'s Return-to-Work Briefing`
   Auto-discover the user's name from Work IQ or the org chart.
   Contains: TL;DR, Team Snapshot, Cross-Team Themes, My Manager (with link), Per Direct Report
   (with links), Top Collaborators (with link and per-person summaries), Key Follow-Ups
   (consolidated from all reports), Meetings Missed, Meeting Recordings to Watch,
   Important Documents, Emerging Risks, First Day Back Plan, Conversations to Have,
   Quick Messages, Open Loops, Guardrails.
   Does NOT duplicate full per-direct sections (those are in individual files).

Use the date the briefing covers through (end date), not the generation date, for YYYY-MM-DD.

Do NOT paste sensitive content verbatim unless asked.

Be concise and high-signal.

---

# MAIN REPORT STRUCTURE (catchup-briefing-YYYY-MM-DD.md)

## TL;DR (max 5 bullets)
Most important things to know immediately.

---

## Team Snapshot
- Overall momentum
- Major changes while away
- Immediate attention areas

---

## What I Likely Missed — Cross-Team
Ranked by impact:
HIGH / MEDIUM / LOW

---

## My Manager
Link to the manager report file:
- **[ManagerName](catchup-FirstName-LastName-Manager-YYYY-MM-DD.md)** — one-line summary of what's top of mind

---

## Per Direct Report

For each person, include a one-line summary and a link to their individual report file:
- **[Name](catchup-FirstName-LastName-YYYY-MM-DD.md)** — one-line summary of top issue

---

## Top Collaborators

Link to the collaborator report file and list each collaborator with a one-line summary:
- **[Top Collaborators](catchup-Top-Collaborators-YYYY-MM-DD.md)**
  - **CollaboratorName** — one-line summary of key topic or follow-up
  - *(repeat for each confirmed collaborator)*

---

## Key Follow-Ups (Consolidated)

Pull the most important follow-up items from ALL reports (directs, manager, and collaborators)
into a single prioritized list. For each item, include:
- **Source:** Who it relates to (direct, manager, or collaborator name)
- **Action:** What needs to happen
- **Urgency:** HIGH / MEDIUM / LOW

This section gives the user a single place to see everything they need to act on.

---

## Meetings I Missed — Key Takeaways
Decisions, actions, context needed.

---

## Meeting Recordings to Watch
Ranked by importance. For each, include: title, date, why it matters, and link to recording.
Prioritize recordings where decisions were made, the user was mentioned, or key team topics were discussed.

---

## Important Documents or Changes
What changed and why it matters.

---

## Emerging Risks Requiring Attention
Rank by severity.

---

## My First Day Back — Recommended Plan
Ordered actions.

---

## Conversations I Should Have Immediately
List people and purpose.

---

## Quick Messages I Can Send Now
Draft short messages if appropriate.

---

## Open Loops Waiting On Me
Unresolved commitments or expectations.

---

## Guardrails

Prioritize signal over completeness.

**Date verification:** WorkIQ may return results outside the requested time window. Before including any item in the briefing, critically assess whether it actually falls within the specified date range. If a source explicitly states it is from outside the window, or if the user flags an item as suspect, remove it. When in doubt, mark the item as "unverified date" rather than presenting it as confirmed. Do not inflate a direct's report with older activity.

If data is noisy:
- tighten timeframe
- filter by project
- filter by direct

Never assume intent.  
Report observed patterns only.

---

# PER-DIRECT REPORT STRUCTURE (catchup-FirstName-LastName-YYYY-MM-DD.md)

## [Name] — [Title]
**Period:** [date range]

### What I Likely Missed That Matters Most
Top 3–5 items ranked:
HIGH / MEDIUM / LOW

### Key Accomplishments
Major progress or outcomes.

### Decisions Made Without Me
Include scope and impact.

### Risks and Blockers
Include severity.

### Needs From Me Now
Approvals, priorities, support.

### Suggested 1:1 Agenda
Bullet list (6–10 items, including 2+ coaching/support questions).

---

# MANAGER REPORT STRUCTURE (catchup-FirstName-LastName-Manager-YYYY-MM-DD.md)

## [ManagerName] — My Manager
**Period:** [date range]

### What's Been Top of Mind
Key priorities, recurring themes, and any shifts in direction or emphasis.

### Decisions and Direction Set
Org-level decisions, priority changes, or expectations communicated.

### What They Need From Me / Follow-Ups
Action items, requests, open questions, or commitments I may have missed.

### Suggested 1:1 Agenda
Bullet list (4–6 items focused on catching up on direction, confirming priority alignment, and surfacing time-sensitive items).

---

# TOP COLLABORATOR REPORT STRUCTURE (catchup-Top-Collaborators-YYYY-MM-DD.md)

## Top Collaborators — High-Level Summary
**Period:** [date range]

For each collaborator, include:

### [CollaboratorName] — [Title/Role if known]
- **Key topics:** What we've been working on together
- **Decisions or outcomes:** Anything decided or concluded
- **Open items:** Things I should follow up on
- **Action needed:** Any immediate follow-up required from me