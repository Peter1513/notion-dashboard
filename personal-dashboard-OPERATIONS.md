# Personal Dashboard — Agent Operating Rules

**NORMATIVE — v1.2 — 2026-09-02**

Load this document before operating on the dashboard. Follow it literally.

---

## 0. Status and precedence

Three documents, distinct roles:

| Document | Role | Answers |
|---|---|---|
| `README.md` | operational | how to deploy, boot, operate, verify, troubleshoot |
| **this document** | **normative** | **what you may and may not do** |
| `personal-dashboard-DESIGN.md` | explanatory | why it was decided this way |

- This document **governs**. README says how; DESIGN says why; neither defines rules. Never infer a rule from either. If they disagree with this document, this document wins and the discrepancy is a bug to report.
- A direct user instruction overrides this document. When you depart from a rule here, say which rule and why.
- If the live schema does not match §2, **stop and report**. Do not improvise a schema, do not create missing properties, do not guess a mapping.
- Do not modify this document as part of ordinary operation. See §7.

---

## 1. Scope

**In scope** — Notion databases in workspace `Peter Wang's Space` (`754f2ec5-7392-818d-8cfa-000384801235`):

- `Goals`
- `Tasks`
- `Log`

**Out of scope** — everything else in the workspace: other pages, other databases, workspace settings, connections, sharing, trash.

Touching anything out of scope requires asking the user first, every time.

---

## 2. Schema (authoritative)

### Goals

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Key | text | Unique slug, human-assigned, e.g. `g-etf-2026q4`. Join key that survives CSV export. |
| Area | select | See shared options below |
| Horizon | select | `week` / `quarter` / `year` / `ongoing` |
| Status | select | `active` / `paused` / `done` / `dropped` |
| Metric | text | How it is measured |
| Target | text | What counts as done |
| NextReview | date | |

### Tasks

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Due | date | Reminder only. Google Calendar is the calendar of record. |
| Area | select | |
| Status | select | `todo` / `doing` / `blocked` / `done` |
| GoalKey | text | Agents write this |
| Goal | relation → Goals | Human-maintained in the UI. Agents never write it. |
| Blocker | text | |
| OpID | text | See §4 |
| Source | select | See shared options below |

### Log

| Property | Type | Notes |
|---|---|---|
| Name | title | |
| Date | date | |
| Area | select | |
| GoalKey | text | Agents write this |
| Goal | relation → Goals | Human-maintained in the UI. Agents never write it. |
| Delta | text | What changed |
| Evidence | url | Link only. Never upload files (5 MB cap on this plan). |
| OpID | text | See §4 |
| Source | select | |

### Shared select options

```text
Area    : quanta | grad-school | etf | hardware | skills | fitness | career
Source  : manual | claude | codex | grok | script
Horizon : week | quarter | year | ongoing
Status  : (Goals) active | paused | done | dropped
          (Tasks) todo | doing | blocked | done
```

### Schema rules

- Never write a select value that is not listed above. If the value you need does not exist, write the closest listed value and note the gap in your reply to the user.
- Never add, rename, retype, or remove a property. Never add a select option.
- Never create a new database.
- Schema changes are human-approved and require a version bump of this document.

**Why `GoalKey` (text) exists alongside `Goal` (relation)**: CSV export turns relations into plain text URLs and cannot be re-imported to rebuild them. `GoalKey` is what survives. Writing the relation instead of the key silently breaks the backup path.

---

## 3. Reads

Read through saved views only.

| View | Source | Filter | Sort |
|---|---|---|---|
| `Agent: Log 14d` | Log | Date within the last 14 days | Date desc |
| `Agent: Open Tasks` | Tasks | Status is not `done` | Due asc |
| `Agent: Active Goals` | Goals | Status is `active` | — |
| `Agent: OpID Lookup` | Log | none | Date desc |

All four views currently exist. `Agent: Log 14d` still has the known hardcoded-date defect described below.

**Rules**

- Use `query_data_sources` with `mode: "view"`.
- **Do not run SQL.** Single-data-source SQL is metered on this plan and will silently exhaust; view mode has no tool-specific quota.
- Never `query_multiple_data_sources`. It requires Business + Notion AI and will fail.
- If you need a cross-database answer, read each view separately and join in your own reasoning.

**Known defect**: `Agent: Log 14d` is filtered on a hardcoded date, not a rolling window. It drifts. If results look stale, say so rather than concluding the Log is empty.

---

## 4. Write protocol

Paste this block into agent system prompts verbatim.

```text
Notion write protocol — personal dashboard

SCOPE
- Databases: Goals, Tasks, Log. Nothing else in the workspace.

WRITE
- Create rows, or update a single property of an existing row. Nothing else.
- Do not write page bodies. notion-update-page's update_content is a targeted
  search-and-replace (fails safely if old_str is absent), but replace_content
  on the same tool replaces the whole page and has a history of deleting child
  content. Stay off the tool entirely.
- NEVER use erase_content. It is irreversible via API.
- Log is append-only. Do not modify existing Log rows.
- Goals rows are human-owned. You may append to Log or add a comment to
  propose a change, but must not write Goals.Status or Goals.Target.
- Write GoalKey (text). Do not write the Goal relation.
- Set OpID and Source on every row you create.
  OpID format: {agent}-{YYYYMMDDTHHMMSS}-{slug}
  Check for an existing row with the same OpID before creating.
- Deleting, archiving, or moving anything requires asking the user first.

READ
- Use saved views: query_data_sources with mode "view".
- Do not run SQL. Single-source SQL is metered on this plan; view mode is not.

RATE LIMITS
- 180 req/min (3 req/s) per user across all MCP tool calls.
- notion-search separately capped at 30 req/min.
- A workspace-wide limit shared across all connections also applies
  (value not published by Notion).
- Do not run parallel searches.
```

### OpID

```text
Format : {agent}-{YYYYMMDDTHHMMSS}-{slug}
Example: claude-20260902T1041-p2-validation
```

`Source` must match the `{agent}` prefix. This pair is the only audit trail; without it a bad write cannot be attributed or reverted.

**This is not a lock.** Query-then-create is a TOCTOU race. It is sufficient because a single human drives the agents in sequence. If writes ever become concurrent or unattended, this scheme is inadequate and the design must change first.

### Enforcement reality

Every prohibition above is a **soft constraint** over the hosted MCP path: `update_page`, `erase_content`, and delete are all `available` and nothing blocks them. The only hard constraint is the scheduled path, whose token omits `Update content`.

Treat this as meaning: an agent that violates this protocol will succeed, and the damage will be found later or not at all.

---

## 5. Recovery envelope

Read this as the reason the write rules are strict, not as a task list.

| Layer | Mechanism | Window | Who |
|---|---|---|---|
| L1 | Notion page history | 7 days | Self-serve (Free plan cap) |
| L2 | Notion support snapshot | ~30 days | Support ticket, vendor-dependent |
| L3 | Weekly CSV export to git | unlimited | **Human / scheduled script only** |
| L4 | DESIGN + this document | structure | Version-controlled in the repo |

**Agent rules**

- Agents never run exports, never delete, never archive, never empty trash.
- Agents never restore from history. Restores are human-driven.
- If you suspect you wrote something wrong, **say so immediately in your reply**. Do not attempt a compensating write. A wrong row plus a wrong correction is harder to untangle than one wrong row.

**Consequence to internalize**: a bad write that goes unnoticed for more than 7 days is unrecoverable without vendor assistance. L3 covers data; L4 covers structure. Neither is automatic. This is why writes are row-scoped, append-only, and attributed.

---

## 6. Pre-write checklist

Run through this before every write:

1. Is the target one of `Goals` / `Tasks` / `Log`? If not — stop, ask.
2. Is this a row create, or a single-property update? If not — stop, ask.
3. Does every select value I am writing appear in §2? If not — use the closest listed value and flag it.
4. Am I writing `GoalKey` rather than `Goal`?
5. Have I set `OpID` and `Source`?
6. Have I checked for an existing row with this `OpID`?
7. Is this a Goals row I am modifying? If yes — stop. Propose via Log or a comment instead.

---

## 7. Change control

- Owner: Peter.
- Agents may **propose** changes to this document, in their reply to the user. Agents do not edit it.
- Any schema, view, or protocol change requires a version bump here **before** the change is applied in Notion. Notion follows this document, not the other way round.
- The authoritative copy is the one in the git repo. The Notion page is a convenience mirror and may lag.

| Version | Date | Change |
|---|---|---|
| v1.2 | 2026-09-02 | Recorded P3 view deployment: `Agent: Open Tasks`, `Agent: Active Goals`, and `Agent: OpID Lookup` now exist; all four normative views are live. |
| v1.1 | 2026-09-02 | Promoted `Log.Goal` (`relation → Goals`) from planned to authoritative schema. Agents still write `GoalKey` only; `Goal` remains human-maintained. |
| v1.0 | 2026-09-02 | Split out of DESIGN v3.0 (§5 schema, §6 views, §7 protocol, §9 backup). Added §0 precedence, §1 scope, §6 checklist, §7 change control. |
