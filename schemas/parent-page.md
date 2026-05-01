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

📂 [Current Year]                              ← e.g., "2026"
   📂 [Current Month]                          ← e.g., "April"
      📅 [Today's dated page]                   ← created/refreshed daily by Mode 1
      📅 [Yesterday's dated page]
      📅 [Day before's dated page]
      ...
      📅 [Older dated pages from this month]
   📂 [Previous Month, same year]              ← e.g., "March"
      📅 [dated pages]
   📂 [Earlier months in the year]
      📅 [dated pages]

📂 [Previous Year]                              ← e.g., "2025"
   📂 [December]
      📅 [dated pages]
   📂 [Earlier months]
   ...

... older years ...

📒 Run Log         ← routine-fire log + linked detail pages (auto-written)
🚨 Incidents       ← append-only connector-failure log (Tier 4 fallback)
⚙️ Preferences (always last)
```

**Hierarchy rule (non-negotiable):** every dated page lives at depth 3 under the parent — `Parent → Year → Month → Date`. There are no flat dated pages directly under the parent. The hierarchy is enforced from creation time, not lazily by archival.

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
- **Position:** below all Year sub-pages and immediately above `Incidents`.

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
- Monthly archival: re-confirm position as part of the structure-verification pass.

## Dynamic elements (created and managed by the skill)

### Year sub-page — top of parent

A Year page (e.g., `2026`) is the outermost dated container. The current Year page sits at the TOP of the parent, above earlier Year pages. Title is the bare 4-digit year — no prefix, no suffix.

**Lifecycle:** created on demand by Mode 1 (or any writer that needs to place a dated page) when no Year page for the current year exists. Idempotent — never created twice.

### Month sub-page — under Year

A Month page (e.g., `April`) lives inside its parent Year page. Title is the spelled-out month name only (e.g., `April`, not `04`, `Apr`, or `April 2026`). The year context is implicit from the Year parent.

**Order under the Year:** newest month at the top, descending. So inside `2026` you see `April → March → February → January` top-to-bottom.

**Lifecycle:** created on demand when no Month page for the current month exists under the current Year page. Idempotent.

### Dated sub-page — under Month

Each Mode 1 run creates (or refreshes) a dated sub-page titled with the date in `DD Month YYYY` format (e.g., `25 April 2026`). The dated page is the child of the Month page, which is the child of the Year page, which is the child of the parent.

New dated pages go at the TOP of their Month page's children (newest at top within the month).

Layout of each dated page is defined in this file (parent-page.md) by reference, but the actual structure is in:
- The dated page itself contains the inline `Morning Queue` database — schema in `schemas/morning-queue-database.md`
- Each row in that database opens to a row detail page — schema in `schemas/row-detail-page.md`

### Resolving "where does today's dated page go?"

On every Mode 1 fire (and any manual write that creates a dated page):

1. Compute current `YYYY`, `Month` (spelled out), and `DD Month YYYY` title.
2. Look for a child of the parent titled exactly `YYYY`. If absent, create it at the top of the parent.
3. Inside that Year page, look for a child titled exactly `Month`. If absent, create it at the top of the Year page.
4. Inside that Month page, create or refresh the `DD Month YYYY` page at the top.

This sequence is the single source of truth for placement. Placement is correct from creation — the skill does NOT lazily flatten dated pages and archive them later. Monthly archival (`modes/monthly-archival.md`) only verifies nothing has drifted out of place.

After several months of use, the parent looks like:

```
HEADER CALLOUT
📂 2026
   📂 April
      📅 25 April 2026
      📅 24 April 2026
      ...
      📅 1 April 2026
   📂 March
      📅 31 March 2026
      ...
   📂 February
   📂 January
📂 2025
   📂 December
   📂 November
   ...
📒 Run Log
🚨 Incidents
⚙️ Preferences
```

## What the parent must NOT contain

- No other content blocks at the parent level (no extra databases, no orphan blocks)
- No PM-added arbitrary content at the parent level (PMs can add content INSIDE Preferences or inside dated pages, but not at the parent level itself)
- **No flat dated pages directly under the parent** — every dated page lives at `Year → Month → Date` depth
- **No flat Month pages directly under the parent** — every Month page lives inside a Year page
- **No `[Month YYYY]` toggle blocks at the parent level** (legacy structure superseded by Year/Month sub-pages)
- No `Run Log` detail pages at the parent level — those must be nested under the `Run Log` sub-page

The skill enforces this on every routine fire by:

1. Verifying the header callout exists (creating or updating it if needed)
2. Verifying the current Year page exists at the top of the parent (creating it if missing)
3. Verifying the current Month page exists inside the Year page (creating it if missing)
4. Verifying today's dated page is at the top of the Month page (creating or moving it if needed)
5. Verifying `Run Log` exists and sits below all Year pages (creating it if missing — see `preflight.md` Step 6)
6. Verifying `Incidents` exists and sits below `Run Log` (creating it if missing — see `preflight.md` Step 6)
7. Verifying `Preferences` is at the bottom
8. Logging (but not removing) any unexpected content at the parent level so the PM can investigate
9. Logging (but not auto-moving) any flat dated page found at the parent level — surfaces it in the run-log so the PM can confirm before relocation

## Visual hierarchy and order — strict rules

The parent has these ordered top-level children:

| #  | Section                          | Created/refreshed by                              | Notes                                                  |
| -- | -------------------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| 1  | Header callout                   | Mode 1 daily; preflight on every fire             | Always first.                                          |
| 2  | Current Year sub-page            | Mode 1 / writers/notion.md on demand              | Newest year at top. Contains Month sub-pages.          |
| 3  | Earlier Year sub-pages           | Prior Mode 1 runs in those years                  | Descending year order.                                 |
| 4  | `Run Log` sub-page               | `preflight.md` Step 6 (creates if missing)        | Below all Year sub-pages, above `Incidents`.           |
| 5  | `Incidents` sub-page             | `preflight.md` Step 6 OR Tier 4 of failure path   | Between `Run Log` and `Preferences`.                   |
| 6  | `Preferences`                    | PM-curated; position enforced                     | Always last.                                           |

**Hierarchy nesting:** Year sub-pages contain Month sub-pages (descending), Month sub-pages contain Date sub-pages (descending). The parent itself only ever has the children listed in the table above — nothing else.

Mode 1's Notion writer enforces this order. `preflight.md` Step 6 creates `Run Log` and `Incidents` in the correct slot if either is missing on a routine fire. Monthly archival verifies (rather than reshuffles) the structure since placement is correct from creation. If a PM manually rearranges, the next routine fire silently re-sorts top-level children; flat dated/month pages found at the parent level are flagged in the run-log for PM review rather than auto-moved.

## Why the structure is fixed

- **Predictability:** every PM's parent looks the same. Onboarding new PMs is faster because the layout is recognizable.
- **Skill reliability:** the skill always knows where to find things. It doesn't have to guess.
- **Audit-friendly:** anyone (PM, leadership, fellow PM) can open a parent page and immediately understand what they're looking at.
- **Archival is clean:** the Year/Month nesting keeps the parent's top-level child count bounded (one entry per active year), no matter how many dated pages accumulate.

## What this schema does NOT include

- Per-PM customization of parent layout (none allowed in v1)
- Multiple parents per PM (one parent per PM, hardcoded in config)
- Cross-PM sharing of parent pages (each PM has their own)
- A "drafts" section or "ideas" section at the parent level (everything goes through dated pages)

If a PM wants additional content at the parent level, they should add it INSIDE the Preferences page under a custom heading. The skill won't touch their custom content there.
