# Routine Entrypoints

This file holds the three exact prompts to paste into Claude Routines when deploying the PM Task Assignment skill for a new PM. Each routine is a cron-fired headless agent that loads the skill from a public GitHub repo at fire time and executes a single mode end-to-end. No mode performs any work outside its own scope.

> Three routines, three modes. Routine 1 collects, Routine 2 executes, Routine 3 archives. Nothing else.

---

## Pre-deploy Checklist

Before creating any routine, the operator must have:

- [ ] **PM's Notion parent page ID** — the 32-character UUID (or the share-link short ID) of the PM's parent page that holds Preferences + dated queue pages + Run Log database.
- [ ] **PM's Preferences page URL** — the public Notion URL of the Preferences page on that parent.
- [ ] **MCP authentication confirmed** for the PM's Claude account — all 5 MCPs connected and authenticated:
  - [ ] Orbit
  - [ ] Gmail
  - [ ] Slack
  - [ ] Fathom
  - [ ] Notion
- [ ] **Repo URL** of the public GitHub repo holding the skill — referred to below as `<REPO_URL>` (raw-content base, e.g. `https://raw.githubusercontent.com/<org>/<repo>/main`).
- [ ] **Routine TZ** — confirm the Routines runner's timezone. If it is not IST, convert the cron lines below from IST to the runner's TZ.

If any item is missing, do not create routines. Stop and resolve first.

---

## Routine 1 — Mode 1 (Morning Collection)

Purpose: pull overnight signals from Orbit / Gmail / Slack / Fathom, write a dated queue page on the Notion parent, await PM approval (PM acts in their own time, not in this routine).

- **Cron (default):** `30 9 * * *` (09:30 IST daily)
- **Override:** if Preferences specifies a different morning run time, use that. Adjust for routine-runner TZ.
- **Connectors required:** Orbit, Gmail, Slack, Fathom, Notion.

### Prompt template

```
Load and follow these files in order:
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/preflight.md
3. <REPO_URL>/modes/mode-1-morning-collection.md

Per-PM injected values:
NOTION_PARENT_PAGE_ID=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Execute Mode 1 (Morning Collection) end-to-end:
- Run preflight against the parent page and Preferences page.
- Collect signals from Orbit, Gmail, Slack, Fathom per Preferences.
- Synthesize and write today's dated queue page on the parent.
- At the end, write a Run Log entry (writers/run-log.md) with the run summary.

Do not run Mode 2. Do not run Monthly Archival. Do not perform any execution actions.
Exit when the queue page is written and the Run Log row + detail page are created.
```

---

## Routine 2 — Mode 2 (Execution)

Purpose: read PM-approved rows from today's dated queue, execute the corresponding Orbit / Slack / Gmail actions, Slack the PM a summary.

- **Cron (default):** `45 10 * * *` (10:45 IST daily)
- **Override:** if Preferences specifies a different execution run time, use that. Adjust for routine-runner TZ.
- **Connectors required:** Orbit, Gmail, Slack, Fathom, Notion. (Fathom included for parity / late-arriving meeting context lookups; not required for execution itself.)

### Prompt template

```
Load and follow these files in order:
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/preflight.md
3. <REPO_URL>/modes/mode-2-execution.md

Per-PM injected values:
NOTION_PARENT_PAGE_ID=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Execute Mode 2 (Execution) end-to-end:
- Run preflight against the parent page and Preferences page.
- Read today's dated queue page; select rows the PM has approved.
- Execute each approved row's action via Orbit / Slack / Gmail per row's payload.
- Send the PM a Slack summary of what was executed / failed / skipped.
- At the end, write a Run Log entry (writers/run-log.md) with the run summary.

Do not run Mode 1. Do not run Monthly Archival. Do not collect new signals.
Exit when execution and the Run Log row + detail page are complete.
```

---

## Routine 3 — Monthly Archival

Purpose: on the 1st of each month, move the previous month's dated queue pages on the Notion parent into a named-month toggle (e.g. `March 2026`).

- **Cron (default):** `0 6 1 * *` (06:00 IST on the 1st of each month)
- **Adjust for routine-runner TZ if not IST.**
- **Connectors required:** Notion. (No collectors, no executors needed.)

### Prompt template

```
Load and follow these files in order:
1. <REPO_URL>/SKILL.md
2. <REPO_URL>/preflight.md
3. <REPO_URL>/modes/monthly-archival.md

Per-PM injected values:
NOTION_PARENT_PAGE_ID=<INJECTED_VALUE>
PREFERENCES_PAGE_URL=<INJECTED_VALUE>

Execute Monthly Archival end-to-end:
- Run preflight against the parent page (Preferences not strictly required, but verify if reachable).
- Identify all dated queue pages on the parent that belong to the previous calendar month.
- Move them into a toggle block named "<Month YYYY>" (e.g. "March 2026") on the parent.
- Create the toggle if it does not exist.
- At the end, write a Run Log entry (writers/run-log.md) with archival summary.

Do not run Mode 1. Do not run Mode 2. Do not collect or execute anything.
Exit when archival and the Run Log row + detail page are complete.
```

---

## Validation Step

After all 3 routines are created in Claude Routines:

1. Open a regular (non-routine) Claude session with the same MCPs authenticated.
2. Run the manual `validate setup` command per `manual-overrides.md`, pointing at the same Preferences page URL.
3. Confirm preflight passes — parent page reachable, Preferences page parsed cleanly, all 5 MCPs respond, Run Log database present (or that preflight Step 6 created it on first run).
4. If `validate setup` reports errors, fix before relying on the routines.

Optional: trigger Routine 1 manually once via the Claude Routines UI ("run now") and inspect the resulting queue page + Run Log row to confirm shape.

---

## Editing the Schedule Later

A PM may want to change their morning or execution run time after deployment.

- **PM action:** edit the run-time field in their Preferences page on Notion.
- **Operator action (also required):** edit the cron line for the corresponding routine in the Claude Routines UI.

The skill, while running inside a routine, **cannot** modify the routine's cron schedule from the inside. Updating Preferences alone is not enough. Both edits are needed for the schedule to actually shift.

If the operator forgets to update the cron, Mode 1 / Mode 2 will continue firing at the old time; the skill will detect the mismatch on its next preflight (Preferences-time vs. actual fire-time delta) and emit a warning into the Run Log detail page, but it will still execute the run.
