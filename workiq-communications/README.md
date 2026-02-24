# WorkIQ Communications Skills

A collection of [GitHub Copilot Skills](https://docs.github.com/en/copilot/copilot-extensions/copilot-skills) for generating return-to-work briefings using [WorkIQ](https://github.com/microsoft/work-iq-mcp) (Microsoft 365 Copilot). These skills extend GitHub Copilot CLI with capabilities for scanning your email, Teams messages, meetings, meeting recordings, and documents to build comprehensive catch-up briefings when you return from time away.

> **⚠️ Prototype Notice:** All skills in this category are experimental prototypes. They are under active development, may change without notice, and are not intended for production use.

## Prerequisites

- **WorkIQ MCP server** must be connected and functional — [github.com/microsoft/work-iq-mcp](https://github.com/microsoft/work-iq-mcp)
- The `workiq-ask_work_iq` tool must be available in your Copilot session

## Which Skill Should I Use?

```
                    ┌──────────────────────────┐
                    │  Returning from time off? │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────┴─────────────┐
                    │  Do you manage a team?    │
                    └────┬──────────────┬──────┘
                         │              │
                        YES             NO
                         │              │
                         ▼              ▼
            ┌────────────────┐  ┌────────────────┐
            │ catchup-manager│  │  catchup-ic    │
            │                │  │                │
            │ Deep dives per │  │ Your own work  │
            │ direct report, │  │ items, manager │
            │ manager, and   │  │ analysis, and  │
            │ collaborators  │  │ collaborators  │
            └────────────────┘  └────────────────┘
```

## Skills

| Skill | Description |
|-------|-------------|
| [`catchup-ic`](./catchup-ic/) | For individual contributors — scans your work items, deadlines, manager activity, and top collaborators to produce a briefing with consolidated follow-ups and a first-day-back plan |
| [`catchup-manager`](./catchup-manager/) | For managers — auto-discovers direct reports and generates per-person deep dives with 1:1 agendas, manager analysis, top collaborator summaries, and a consolidated briefing with follow-ups |

## Skill Details

### [`catchup-manager`](./catchup-manager/)

**Generate a comprehensive return-to-work briefing for managers with direct reports.**

This skill auto-discovers your org context and builds a multi-file briefing by querying WorkIQ across the time you were away. It:

- **Auto-discovers your org** — direct reports, manager, and top collaborators from the org chart and activity data
- **Deep dives per direct** — activity, decisions, risks, blockers, what they need from you, and a suggested 1:1 agenda
- **Analyzes your manager** — what's been top of mind, decisions made, and what they need from you
- **Discovers top collaborators** — finds your most active cross-team collaborators and summarizes interactions
- **Consolidates follow-ups** — pulls all action items from every report into a single prioritized list
- **Generates markdown reports** — one file per direct, one for your manager, one for collaborators, and a main briefing that links them all

#### Output Files

| File | Contents |
|------|----------|
| `catchup-<Name>-YYYY-MM-DD.md` | Per-direct report — what you missed, accomplishments, decisions, risks, needs, 1:1 agenda |
| `catchup-<Name>-Manager-YYYY-MM-DD.md` | Manager report — priorities, direction, what they need from you, 1:1 agenda |
| `catchup-Top-Collaborators-YYYY-MM-DD.md` | All confirmed collaborators — key topics, decisions, open items per person |
| `catchup-briefing-YYYY-MM-DD.md` | Main briefing — TL;DR, team snapshot, links to all reports, consolidated follow-ups, first-day plan |

#### Quick Start

```
Catch me up — I was out starting 2/16
```

```
Run catchup-manager, I've been away for 5 days
```

---

### [`catchup-ic`](./catchup-ic/)

**Generate a comprehensive return-to-work briefing for individual contributors.**

This skill auto-discovers your context and builds a briefing focused on your own work, your manager's activity, and your top collaborators. It:

- **Tracks your work** — work items, PRs, tasks, and deadlines that are waiting on you
- **Analyzes your manager** — what's been top of mind, decisions made, and what they need from you
- **Discovers top collaborators** — finds your most active collaborators and summarizes interactions
- **Consolidates follow-ups** — pulls all action items into a single prioritized list
- **Generates markdown reports** — one for your manager, one for collaborators, and a main briefing that links them all

#### Output Files

| File | Contents |
|------|----------|
| `catchup-<Name>-Manager-YYYY-MM-DD.md` | Manager report — priorities, direction, what they need from you, 1:1 agenda |
| `catchup-Top-Collaborators-YYYY-MM-DD.md` | All confirmed collaborators — key topics, decisions, open items per person |
| `catchup-briefing-YYYY-MM-DD.md` | Main briefing — TL;DR, your work items, links to reports, consolidated follow-ups, first-day plan |

#### Quick Start

```
Catch me up — I was out starting 3/10
```

```
Run catchup-ic, I've been away for a week
```

## Key Features

Both skills share these capabilities:

- **Fully automatic org discovery** — no hardcoded names or paths; works for anyone with WorkIQ access
- **Remembers your save location** — stores your preferred output path and offers it as default on future runs
- **Date verification guardrails** — WorkIQ may return results outside the requested window; the skills verify dates before including items
- **Meeting recording filtering** — only surfaces recordings where decisions were made or you were mentioned
- **Consolidated follow-ups** — a single prioritized list across all reports so nothing falls through the cracks
- **Week-based output folders** — reports are organized into `Week of YYYY-MM-DD` subfolders

## Known Limitations

- **WorkIQ reliability** — WorkIQ may time out or become temporarily unavailable during long runs. The skills handle partial data gracefully and can resume.
- **Date filtering** — WorkIQ sometimes returns results outside the requested date range. The skills include guardrails but some manual verification may be needed.
- **Collaborator discovery** — Top collaborator queries depend on interaction volume. People you collaborate with infrequently may not appear.
- **Meeting recordings** — Recording links depend on what's indexed in Microsoft 365. Not all recordings may be discoverable.
