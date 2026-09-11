# Dashboard consistency fixes — design

**Date:** 2026-09-11  
**Status:** Approved design  
**Target branch:** `beta`  
**Baseline:** `022fa3d7309dd1f949ccd435ad779c0fdde28630`

## 1. Purpose

Resolve the internal and cross-file inconsistencies identified in:

- `README.md`
- `personal-dashboard-OPERATIONS.md`
- `personal-dashboard-DESIGN.md`

The change also updates Live Notion where the approved rules require new schema properties or saved views.

## 2. Scope

### Documentation

Update all three project documents so that permissions, read paths, audit behavior, deployment status, history, and examples agree.

### Live Notion

Modify only the existing `Goals` and `Tasks` databases under `Dashboard`:

- Add `Goals.OpID` as text.
- Add `Goals.Source` as select.
- Create `Agent: All Goals`.
- Create `Agent: All Tasks`.

No database, row, property, or existing view is deleted, archived, or moved.

### Non-scope

- P4 deployment.
- P5 scheduler or CSV export implementation.
- Changes to `main`.
- Backfilling `OpID` or `Source` on legacy Goals.
- Changing existing Task or Log rows.

## 3. Delivery sequence

Use two documentation commits around the Live Notion migration.

### Commit 1: target rules

- Bump OPERATIONS to v1.9.
- Bump DESIGN to v3.9.
- Update README and mark the migration as pending.
- Define the target schema, permissions, read paths, audit behavior, and failure behavior.

After this commit, apply the Notion schema and view changes.

### Commit 2: verified state

Only after every verification passes:

- Bump OPERATIONS to v1.10.
- Bump DESIGN to v3.10.
- Update README with current observed results.
- Record that the Live Notion migration was completed and verified.

This sequence preserves the existing rule that documentation is committed before schema or view changes.

## 4. Agent permissions

### Goals

Agents may create a Goal. Before creation, the agent must verify that both `Key` and `OpID` are unique through `Agent: All Goals`.

Every Agent-created Goal must include:

- `Name`
- `Key`
- `Area`
- `Horizon`
- `Status`
- `Metric`
- `Target`
- `NextReview`
- `OpID`
- `Source`

If any value is missing or ambiguous, the agent must ask the user before creating the Goal.

After creation, the Goal is read-only to agents. Agents must propose later changes through a Log row or a comment. They must not modify any Goal property.

Existing Goals are legacy records. Their new `OpID` and `Source` fields remain empty because their original creator cannot be established reliably.

### Tasks

Agents may create Tasks under the existing creation protocol.

For an existing Task, an agent may update only:

- `Status`
- `Due`
- `Blocker`

Each operation changes exactly one property. Agents must not modify `Name`, `Area`, `GoalKey`, `Goal`, `OpID`, or `Source`.

Property updates must use a property-only operation. Page body or content mutation parameters must not be used.

### Log

Log remains append-only. Agents never modify existing Log rows.

## 5. Audit behavior

A Task's `OpID` and `Source` identify its creator and remain unchanged after creation.

After a Task property update succeeds, the agent appends a new Log row containing:

- The Task name or stable identifying information.
- The changed property.
- The old value.
- The new value.
- A new Log OpID.
- Source matching the executing agent.

Notion does not provide a transaction covering both operations. The required order is:

1. Update the Task property.
2. Append the audit Log row.

If the Task update succeeds and the Log append fails, the agent reports:

```text
PARTIAL FAILURE
Task update: succeeded
Audit Log append: failed
No compensating write attempted
```

The agent must not revert the Task or attempt another corrective write automatically.

## 6. Saved views and read paths

| View | Source | Filter | Purpose |
|---|---|---|---|
| `Agent: Open Tasks` | Tasks | Status is not `done` | Routine open-task reading |
| `Agent: Active Goals` | Goals | Status is `active` | Routine active-goal reading |
| `Agent: All Tasks` | Tasks | none | Complete Task lookup and OpID deduplication |
| `Agent: All Goals` | Goals | none | Goal Key/OpID deduplication, GoalKey resolution, and Area inheritance |
| `Agent: Log YYYY-MM` | Log | Calendar-month bounds | Monthly Log reading |
| `Agent: OpID Lookup` | Log | none | Log OpID deduplication |

Agents continue to read through saved views only.

When creating a Task, the agent uses `Agent: All Tasks` to check OpID across every status, including `done`. It then uses `Agent: All Goals` to resolve GoalKey and copy the Goal's Area, including when the Goal is paused, done, or dropped.

## 7. P4 wording

OPERATIONS retains a description of the planned restricted scheduler token, but it must state that P4 is deferred and unverified.

The rules must make these facts explicit:

- All current hosted MCP prohibitions are soft constraints.
- A future P4 token with Update content disabled would create a hard capability constraint only for that scheduled path.
- Agents must not treat that hard constraint as currently active.

P5 recovery information about a future weekly CSV export remains in OPERATIONS.

## 8. Documentation corrections

The document updates also:

- Correct the OpID example to include seconds, such as `codex-20260911T143025-update-task-status`.
- Replace DESIGN's stale reference to “§3 access mode B” with the current interactive/scheduled-path terminology.
- Remove README wording that points to a nonexistent Bash/curl example.
- Distinguish the earlier Status property conversion from the later option and view-filter repair.
- Update README observed results from the stale 2026-09-03 summary while preserving historical evidence as history.
- Make the Goals creation rule, post-creation immutability, Task update whitelist, and audit requirements consistent across all three files.

## 9. Live Notion migration

After Commit 1:

1. Confirm Live Notion still matches the pre-migration schema expected by OPERATIONS v1.8.
2. Add `Goals.OpID` with type text.
3. Add `Goals.Source` with type select and exactly these options:
   - `manual`
   - `claude`
   - `codex`
   - `grok`
   - `script`
4. Create `Agent: All Goals` with no filter.
5. Create `Agent: All Tasks` with no filter.
6. Do not populate the new fields on existing Goals.

If any mutation fails, stop. Report which steps succeeded, failed, and were not attempted. Do not remove successful changes and do not create the completion commit.

## 10. Verification

Before Commit 1, re-fetch `beta`. If its HEAD differs from the approved baseline, re-evaluate the diff before writing.

After the Notion migration, verify:

- `Goals.OpID` exists and is text.
- `Goals.Source` exists and is select.
- `Goals.Source` contains exactly the approved Source options.
- Existing Goals retain empty `OpID` and `Source`.
- `Agent: All Goals` exists with no filter and can return Goals of every present status.
- `Agent: All Tasks` exists with no filter and can return Tasks including `done`.
- Existing Agent views and their filters remain unchanged.
- All three documents agree on schema, permissions, reads, audit behavior, P4 status, and project progress.
- OPERATIONS and DESIGN versions and changelogs match the intended phase.
- Every active OpID format and example includes seconds.

Commit 2 is allowed only when all applicable checks pass.

## 11. Issue coverage

| Finding | Resolution |
|---|---|
| Goals modification rules conflict | Permit complete creation; prohibit all post-creation Agent edits |
| Goals lack required audit fields | Add `OpID` and `Source`; grandfather legacy rows |
| Single-property updates conflict with avoiding the update tool | Permit property-only updates and prohibit content mutation |
| Undeployed P4 described as active protection | Retain it as explicitly deferred and conditional |
| OpID example omits seconds | Correct the example |
| DESIGN references nonexistent access mode B | Replace the stale reference |
| README references a nonexistent shell example | Remove the reference |
| Completed Tasks cannot be checked for duplicate OpID | Add `Agent: All Tasks` |
| Non-active Goals cannot supply inherited Area | Add `Agent: All Goals` |
| Task updates lack per-change attribution | Append an audit Log after each allowed update |
| Status migration timeline is ambiguous | Separate property conversion from later repair and verification |
