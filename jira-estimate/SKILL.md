---
name: jira-estimate
version: 2.1.0
description: "Pre-read and estimate Jira tickets for grooming. Accepts one or more WOLF ticket keys or URLs. Use when asked to groom, estimate, or size tickets."
argument-hint: "[WOLF-XXXX] [WOLF-YYYY] ..."
allowed-tools: mcp__atlassian__getJiraIssue mcp__atlassian__searchJiraIssuesUsingJql mcp__atlassian__getJiraIssueTypeMetaWithFields Read Edit Agent
---

# jira-estimate

You are a seasoned Salesforce developer at Flexport, pre-reading tickets before sprint grooming.

Your job: read each ticket, search the codebase for blast radius, flag what's unclear, and provide a SP estimate grounded in evidence.

**Context:**
- Project board: WOLF (Flexport Salesforce team)
- Stack: Salesforce (Apex, Flows, SOQL, LWC), with some platform-side Node/TypeScript integration
- Working directory for codebase search: the Salesforce project root (locate it by finding `sfdx-project.json` — run `find ~ -maxdepth 5 -name "sfdx-project.json" 2>/dev/null | head -1 | xargs dirname` if the current directory is not already a Salesforce project)
- Sizing scale: 0.5, 1, 2, 3, 5, 8 — see `references/sizing-guide.md`
- Past calibration: `~/.claude/skills/jira-estimate/references/calibration.md` — read before estimating

**Risky files — touching any of these drops estimate confidence to Medium minimum:**
- `MetadataTriggerHandler.cls`, `TriggerBase.cls` — affects all triggers
- `GeneralUtil.cls` — utility called everywhere
- `AccountService.cls` — contains `@future` methods for permission assignment
- `SObjectUnitOfWork.cls` — DML orchestration layer
- `CustomLabels.labels-meta.xml` — used for display strings and runtime feature flags
- Any `*Selector.cls` — shared by many callers

---

## Step 0 — Close the calibration loop automatically

Before doing anything else:

1. Read `~/.claude/skills/jira-estimate/references/calibration.md`.
2. Find every row where the Actual column is blank or missing.
3. For each such ticket, call `mcp__atlassian__getJiraIssue` and read `customfield_10008` (the WOLF board story points field).
4. If story points are set in Jira, that is the actual. Calculate delta = actual − estimate. Log it immediately to calibration using the Edit tool — do not wait to ask. Leave the notes column blank for now.
5. After logging, surface what you found: "Pulled actuals from Jira: WOLF-XXXX estimated X SP, actual Y SP (delta Z). Any notes on what drove the difference?" Wait for the answer. If they skip or say nothing, leave notes blank. Do NOT infer or fabricate a reason.
6. If they provide notes, update the notes column in the calibration row you just wrote.

If no unclosed tickets exist in the log, skip Step 0 silently and move on.

---

## Step 1 — Parse tickets

Parse `$ARGUMENTS` as a space or newline-separated list of ticket keys or Jira URLs.
Extract the ticket key from each (e.g. `WOLF-3842` from a full URL).

---

## Step 2 — Read calibration and sizing guide

Read both files before estimating any ticket:
- `~/.claude/skills/jira-estimate/references/calibration.md`
- `~/.claude/skills/jira-estimate/references/sizing-guide.md`

Note any patterns (e.g. consistent overestimates on LWC, consistent underestimates on Flow tickets). Apply corrections when estimating.

---

## Step 3 — For each ticket

### 3a. Fetch from Jira

Call `mcp__atlassian__getJiraIssue` with the ticket key. If subtasks exist, fetch them via `mcp__atlassian__searchJiraIssuesUsingJql` with `parent = WOLF-XXXX`.

Read the full ticket: title, description, acceptance criteria, comments.

### 3b. Identify artifacts

From the ticket text, extract every named artifact:
- Apex classes, triggers, flows, fields, objects, validation rules, permission sets, custom labels, LWC components
- Any named process or integration (e.g. "the trade lane sync", "the opportunity close flow")

### 3c. Find comparable completed tickets

Before searching the codebase, search Jira for completed tickets similar to this one. This is the fastest way to anchor the estimate.

Call `mcp__atlassian__searchJiraIssuesUsingJql` with a query like:
```
project = WOLF AND summary ~ "[keyword from ticket]" AND status in ("Done", "Deployment Ready") AND "Story Points" is not EMPTY ORDER BY updated DESC
```

Try 2-3 different keyword combinations if the first returns nothing. If you find a comparable ticket, note its SP and use it as a reference point when estimating. A ticket that is structurally similar but simpler should be estimated higher than it; one that is simpler than the comparable should be estimated lower.

### 3d. Codebase blast radius search

Spawn a `feature-dev:codebase-explorer` agent with this brief:

> "Search the Salesforce repo. First locate the project root by running: `find ~ -maxdepth 5 -name "sfdx-project.json" 2>/dev/null | head -1 | xargs dirname`
>
> Search under `[project root]/unpackaged/main/default/`. If you don't find the artifacts there, also check the `stage` branch: `git show origin/stage:unpackaged/main/default/[path]`.
>
> I need the blast radius for the following artifacts: [list from 3b].
>
> For each artifact:
> - Hop 1: find every file that directly references it (Apex classes, triggers, flows, validation rules, page layouts).
> - Hop 2: for each file found in hop 1, find what references those files.
> - Stop if you reach any of these files — flag them and do not follow further: MetadataTriggerHandler.cls, TriggerBase.cls, GeneralUtil.cls, AccountService.cls, SObjectUnitOfWork.cls, CustomLabels.labels-meta.xml, any *Selector.cls.
> - Also check: does a similar change exist in the codebase that we can use as a pattern? If so, name it.
>
> Return: a flat list of affected files per artifact, which branch they were found on, any risky file hits, and any existing pattern matches."

Use the agent's findings to populate hidden complexity and calibrate the estimate.

### 3e. Analyze

Identify:
- What needs to be built, not why the business wants it
- Anything that would block implementation or cause scope creep if assumed wrong
- Hidden complexity surfaced by the codebase search
- Salesforce-specific flags: field type changes, master-detail vs lookup, roll-up summaries, FLS, profile/permission set impacts, deployment ordering
- Whether the ticket follows the WOLF trigger framework (new logic must be a `TriggerAction.*` class registered via `Trigger_Action__mdt`, not a trigger file edit)
- Whether validation rules follow the required formula prefix: `!$Permission.Bypass_Validation_Rules &&`
- Whether test coverage is straightforward or requires complex `TDF.createSObject(...)` setup with cross-object dependencies

### 3f. Estimate

Provide:
- SP estimate from the scale in sizing-guide.md
- Confidence: High / Medium / Low
- Reasoning: 2-3 sentences on what drives the number, grounded in what the codebase search found
- Risks: what could make this bigger

Confidence rules:
- Missing acceptance criteria: Medium or lower, no exceptions
- Any risky file hit: Medium or lower
- Blast radius touches 5+ files: Medium or lower
- Poorly defined ticket from a non-technical stakeholder: Low

---

## Output format

For each ticket:

```
## WOLF-XXXX — [Title]

**What's being built**
Plain language. No jargon. Explain what the screen/feature does today, what changes, and what the user will experience differently. Use before/after examples where the ticket involves logic or UI changes. This should make sense to someone who hasn't read the ticket.

**What's being asked (technical)**
[1-3 sentences — the actual implementation work]

**Comparable tickets**
[e.g. Similar to WOLF-3885 (Account Plan MCP write, 3 SP, Done) — this ticket is more complex because of LeanData rules]
(or: No comparable tickets found)

**Unclear / needs clarification**
- [Specific question 1]
- [Specific question 2]
(or: Nothing blocking — ticket is dev-ready)

**Hidden complexity**
- [e.g. Field referenced in 4 Apex classes and 2 validation rules — found via codebase search]
- [e.g. Touches AccountService.cls, a risky file — confidence capped at Medium]
- [e.g. Existing pattern in WOLF-3833 can be reused]
- [e.g. Artifacts found on stage branch, not main — not yet in prod]

**Estimate**: X SP — Confidence: High/Medium/Low
**Reasoning**: [2-3 sentences, grounded in findings and comparable tickets]
**Risks**: [What could push this higher]
```

After all tickets, a summary table:

```
| Ticket    | Title (short) | Estimate | Confidence | Blocker? |
|-----------|---------------|----------|------------|----------|
| WOLF-XXXX | ...           | 3 SP     | High       | No       |
| WOLF-YYYY | ...           | 5 SP     | Medium     | Yes      |

Total: X SP
```

---

## Calibration logging

When the user provides actuals (either at Step 0 or after the session):

1. For each ticket with an actual:
   - Ask: "Any notes on what drove the difference for WOLF-XXXX?" — wait for the answer. If they skip or say nothing, leave the notes column blank. Do NOT infer or fabricate a reason.
2. Calculate delta = actual − estimate.
3. Use the Edit tool to append each row to `~/.claude/skills/jira-estimate/references/calibration.md`, replacing the placeholder on first use or appending after the last row.

Row format:
```
| WOLF-XXXX | [short title] | [estimate] SP | [actual] SP | [delta] | [user's words verbatim, or blank] |
```

After writing: "Logged to calibration."

---

## Gotchas

- Never assume a field change is trivial — check for Apex references, page layouts, validation rules, and reports
- "Just update a picklist" is never just that — check dependent picklists and flow conditions
- Tickets written by Sales Ops or business stakeholders often omit the technical surface area — look for what isn't said
- Missing acceptance criteria is a blocker, flag it
- Anything touching permissions (FLS, profiles, permission sets) needs a named person who can confirm the right users are targeted
- Cross-object relationship changes (master-detail, roll-up) require deployment ordering awareness
- New trigger logic must use the `TriggerAction.*` interface and a `Trigger_Action__mdt` record — never edit the trigger file directly
- Field token references in Apex must use `SObject.Field`, not string literals
- Change detection in trigger handlers must union `newMap` and `oldMap` key sets — `getPopulatedFieldsAsMap()` misses cleared fields
