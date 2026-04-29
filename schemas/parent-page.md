# Schema — Parent Notion Page

The PM's Notion parent page (set in `config.md` as `DEFAULT_NOTION_PARENT_PAGE_ID`) follows a fixed structure. Every PM's parent page looks identical in shape and order. This is enforced by `writers/notion.md` on every Mode 1 run and on every monthly archival, and re-confirmed by `preflight.md` Step 6 on every routine fire.

See also:
- `schemas/run-log-database.md` — schema for the inline database on the `Run Log` sub-page
- `schemas/run-log-detail-page.md` — schema for the per-fire detail pages nested under `Run Log`
- `connector-failure-notify.md` — Tier 4 fallback that writes to the `Incidents` sub-page
- `preflight.md` — Step 6 creates `Run Log` and `Incidents` sub-pages if missing and re-asserts order

## Top-down layout

```
[Page header — controlled by Notion, just shows the page title]

┌─────────────────────────────────────────────────────────────┐
│ HEADER CALLOUT (static, updated daily by Mode 1)            │
│                                                              │
│ "PM Task Assignment skill writes here every morning at      │
│  [PM's morning run time] IST. Approved items execute at     │
│  [PM's execution run time] IST. Last run: [timestamp]."     │
└─────────────────────────────────────────────────────────────┘

📅 [Today's dated page] — created/refreshed daily by Mode 1
📅 [Yesterday's dated page]
📅 [Day before's dated page]
...
📅 [Older dated pages from this month]

▸ [Month YYYY] (toggle, e.g., "April 2026")    ← created on 1st of next month
   📅 [30 dated pages from that month]

▸ [Earlier Month YYYY] (toggle)
   📅 [dated pages]

... older months ...

📒 Run Log         ← NEW: routine-fire log + linked detail pages (auto-written)
🚨 Incidents       ← NEW: append-only connector-failure log (Tier 4 fallback)
⚙️ Preferences (always last)
```

## Static elements (always present)

### 1. Header callout — at the very top

A Notion callout block. Updated by Mode 1 every morning (the timestamp at minimum). Content:

```
PM Task Assignment skill writes here every morning at [TIME] IST.
Approved items execute at [TIME] IST.
Last morning run: [ISO timestamp]
Last execution run: [ISO timestamp]
```

If the skill detected connector failures or other issues at the last run, the header includes a brief warning line:

```
⚠️ Last run had issues — see today's page for details.
```

The header is the only static, persistent block on the parent. Everything else under it is either dated pages, the operational sub-pages (`Run Log`, `Incidents`), or the `Preferences` page.

### 2. Run Log sub-page — second-to-last static block

- **Title:** exactly `Run Log` (no emoji, no suffix — the writer matches by exact title).
- **Contents:**
  1. A one-paragraph header callout at the top of the page reading approximately:
     > "This page is auto-written by the PM Task Assignment skill on every routine fire. Do not edit rows manually — manual edits will be overwritten or ignored. To investigate a specific fire, click into its row to open the detail page."
  2. An inline Notion database below the callout. Schema is in `schemas/run-log-database.md`.
- **Sort:** the inline database default view sorts by `Started` descending (newest fire at the top).
- **Detail pages:** each row in the inline database opens a child page whose schema is in `schemas/run-log-detail-page.md`. Detail pages are nested as children of THIS sub-page (not of the parent), which keeps the parent tidy — the parent only ever has the top-level children listed in the order rules below.
- **Lifecycle:** created on first preflight if missing (per `preflight.md` Step 6). Once it exists, the writer (`writers/run-log.md`) appends a row on every fire.
- **Position:** below all month toggles and immediately above `Incidents`.

### 3. Incidents sub-page — between Run Log and Preferences

- **Title:** exactly `Incidents`.
- **Contents:** an append-only inline Notion database. No header callout required — the title is self-explanatory. Columns:

  | Column              | Type                       | Notes                                                                                       |
  | ------------------- | -------------------------- | ------------------------------------------------------------------------------------------- |
  | `Timestamp`         | Date (with time)           | When the incident occurred (ISO, local TZ).                                                 |
  | `Mode`              | Select                     | One of: `Mode 1`, `Mode 2`, `Monthly Archival`.                                             |
  | `MCP`               | Select                     | One of: `Orbit`, `Gmail`, `Slack`, `Fathom`, `Notion`.                                      |
  | `Step`              | Text                       | Which step in the mode failed (free text, e.g., "fetch overdue tasks").                     |
  | `Error`             | Text                       | The error message captured from the connector.                                              |
  | `Retry on next fire`| Checkbox                   | If checked, the next fire of the same mode will retry the failed step before normal flow.   |
  | `Resolved`          | Checkbox                   | Set manually by the PM (or by the skill once a later fire succeeds on the same MCP/step).   |
  | `Run Log link`      | URL (or relation)          | Points at the corresponding row's detail page in `Run Log`.                                 |

- **Sort:** by `Timestamp` descending (most recent incident first).
- **Lifecycle:** created on first preflight if missing (per `preflight.md` Step 6), OR on first incident if it doesn't exist yet (the connector-failure path in `connector-failure-notify.md` Tier 4 will create-or-append).
- **Read on every fire:** preflight loads unresolved incidents and surfaces them in the new run-log entry's "Pre-existing unresolved incidents" field, so each fire is aware of standing failures.
- **Escalation:** 3+ consecutive unresolved incidents on the same `MCP` trigger a Slack escalation to the backup person — already documented in `connector-failure-notify.md`.
- **Position:** below `Run Log` and immediately above `Preferences`.

### 4. Preferences sub-page — always last

The `Preferences` sub-page is the very last child of the parent. Position is enforced by:

- `writers/notion.md` Step 1 of the Mode 1 flow: confirm Preferences is at the bottom; move it if not.
- `preflight.md` Step 6: re-confirm Preferences is last after creating/positioning `Run Log` and `Incidents`.
- Monthly archival: re-confirm position after moving prior-month pages into a toggle.

## Dynamic elements (created and managed by the skill)

### Dated sub-pages — top of parent

Each Mode 1 run creates a new dated sub-page titled with the current date in `DD Month YYYY` format (e.g., `25 April 2026`). New pages go at the TOP of the parent (above the previous day's).

Layout of each dated page is defined in this file (parent-page.md) by reference, but the actual structure is in:
- The dated page itself contains the inline `Morning Queue` database — schema in `schemas/morning-queue-database.md`
- Each row in that database opens to a row detail page — schema in `schemas/row-detail-page.md`

### Month toggles — middle of parent

On the 1st of each month, `monthly-archival.md` runs and:

1. Identifies all dated pages from the previous month
2. Creates a toggle block titled `[Month YYYY]` (e.g., `April 2026`)
3. Moves all those dated pages inside the toggle, preserving newest-on-top order within the toggle
4. Positions the toggle below the current month's dated pages and above earlier-month toggles
5. Re-confirms `Preferences` is still last

After several months of use, the parent looks like:

```
HEADER CALLOUT
📅 25 April 2026
📅 24 April 2026
📅 23 April 2026
...
📅 1 April 2026
▸ March 2026
▸ February 2026
▸ January 2026
📒 Run Log
🚨 Incidents
⚙️ Preferences
```

## What the parent must NOT contain

- No other content blocks at the parent level (no extra databases, no extra sub-pages beyond the seven enforced ones, no orphan blocks)
- No PM-added arbitrary content at the parent level (PMs can add content INSIDE Preferences or inside dated pages, but not at the parent level itself)
- No orphan dated pages outside the toggle structure once their month has been archived
- No `Run Log` detail pages at the parent level — those must be nested under the `Run Log` sub-page

The skill enforces this on every routine fire by:

1. Verifying the header callout exists (creating or updating it if needed)
2. Verifying today's dated page is at the top (creating or moving it if needed)
3. Verifying `Run Log` exists and sits below the month toggles (creating it if missing — see `preflight.md` Step 6)
4. Verifying `Incidents` exists and sits below `Run Log` (creating it if missing — see `preflight.md` Step 6)
5. Verifying `Preferences` is at the bottom
6. Logging (but not removing) any unexpected content at the parent level so the PM can investigate

## Visual hierarchy and order — strict rules

The parent has **seven** ordered sections (was five before the routines refactor). The order of children is:

| #  | Section                                  | Created/refreshed by                              | Notes                                                  |
| -- | ---------------------------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| 1  | Header callout                           | Mode 1 daily; preflight on every fire             | Always first.                                          |
| 2  | Today's dated page                       | Mode 1 (or current run if missing)                | Newest at top.                                         |
| 3  | This month's earlier dated pages         | Prior Mode 1 runs                                 | Newest first.                                          |
| 4  | Last month's toggle (if exists)          | `monthly-archival.md` on the 1st                  | Below current month's dated pages.                     |
| 5  | Earlier months' toggles                  | Prior monthly archivals                           | Newest first.                                          |
| 6  | **`Run Log` sub-page (NEW)**             | `preflight.md` Step 6 (creates if missing)        | Below all month toggles, above `Incidents`.            |
| 7  | **`Incidents` sub-page (NEW)**           | `preflight.md` Step 6 OR Tier 4 of failure path   | Between `Run Log` and `Preferences`.                   |
| 8  | `Preferences`                            | PM-curated; position enforced                     | Always last.                                           |

(The numbered list above is for reference; the seven *sections* are 1–8 minus the toggle/dated-page subdivisions, since 2–5 are all "dated content" variants. The hard count of distinct top-level *named* children is: header callout + today + older-this-month + month toggles + Run Log + Incidents + Preferences.)

Mode 1's Notion writer enforces this order. `preflight.md` Step 6 creates `Run Log` and `Incidents` in the correct slot if either is missing on a routine fire. Monthly archival re-establishes the full order after moving prior-month pages into a toggle. If a PM manually rearranges, the next routine fire silently re-sorts.

## Why the structure is fixed

- **Predictability:** every PM's parent looks the same. Onboarding new PMs is faster because the layout is recognizable.
- **Skill reliability:** the skill always knows where to find things. It doesn't have to guess.
- **Audit-friendly:** anyone (PM, leadership, fellow PM) can open a parent page and immediately understand what they're looking at.
- **Archival is clean:** monthly toggles keep the page from growing unbounded.

## What this schema does NOT include

- Per-PM customization of parent layout (none allowed in v1)
- Multiple parents per PM (one parent per PM, hardcoded in config)
- Cross-PM sharing of parent pages (each PM has their own)
- A "drafts" section or "ideas" section at the parent level (everything goes through dated pages)

If a PM wants additional content at the parent level, they should add it INSIDE the Preferences page under a custom heading. The skill won't touch their custom content there.
