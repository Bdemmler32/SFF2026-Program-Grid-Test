# SFF2026 Technical Program — Concurrent Grid

A self-contained, drop-in schedule widget for the SFF2026 Technical Program. One HTML file reads a live-hosted Excel workbook, builds a searchable/filterable schedule, and renders it two ways: a chronological list, and a Room × Time grid for spotting concurrent sessions.

No build step, no server-side code, no dependencies beyond two CDN scripts. Designed to be dropped into a page on tms.org the same way it runs on GitHub Pages.

---

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire widget — markup, styles, and logic in one file |
| `SFF2026-Program.xlsx` | The data source. Must live in the **same folder** as `index.html` |

The widget fetches the workbook relative to its own location (`SFF2026-Program.xlsx`, no leading slash), so this works unmodified on GitHub Pages root, in a tms.org subfolder, or anywhere else — as long as the two files stay together. If the workbook ever needs to live at a different path or domain, that's a one-line change: the `DATA_URL` constant near the top of the `<script>` block.

---

## Data source

### `Abstracts` sheet (required)

One row per talk or poster. Header row is row 1; data starts row 2. Columns used:

| Column | Used for |
|---|---|
| `Topic Area` | Filter tree + color coding (see below) |
| `Symposium Name` | Not currently displayed (constant across all rows) |
| `Session Name` | Shown in the event modal |
| `Date` | Day grouping |
| `StartTime` | Start of the time slot |
| `SesOrder` | **Internal only** — never rendered |
| `Duration` | Minutes; added to `StartTime` to get the end time |
| `RoomName` | Room filter + grid column |
| `Abstract Number` | Sort key within a shared time slot (e.g. ordering posters); not displayed |
| `Abstract Type` | Oral / StudentOral / Poster / StudentPoster / Invited — badges + filter |
| `Abstract Title` | Card/modal title |
| `Speaker's ID` | **Internal only** — never rendered |
| `Speaker's Name` | Card/modal |
| `Speaker's Company` | Card/modal |
| `All Authors and Affiliations` | Modal only |
| `Abstract` | Modal only (full text) |
| `Academic Status` | **Internal only** — never rendered |

**The three "internal only" columns are read from the sheet but intentionally never displayed anywhere in the UI** — not in cards, not in the modal, not in search results. They're there for backstage use only.

### `Admin` sheet (required)

One row, one relevant column: `Revision Date` (e.g. `07/01/2026`). This drives the "Current as of" stamp at the top of the widget. If the sheet or column is missing, the widget silently falls back to today's date rather than erroring.

---

## How the data flows

1. **Fetch** — `fetchProgramData()` pulls the workbook with a cache-busting query param (`?v=<timestamp>`) so browsers don't serve a stale copy after the sheet is updated.
2. **Parse** — [SheetJS](https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js) reads the workbook. `cellDates: true` means Excel date/time cells arrive as native JS `Date` objects.
3. **Normalize** — `rowToEvent()` converts each raw row into a clean event object:
   - `startMinutes` / `endMinutes`: minutes-since-midnight integers, computed as `StartTime + Duration`. See below for more detail — this came up before.
   - `dateKey`: a `YYYY-MM-DD` string used for grouping/filtering by day (computed with local-timezone getters, not `toISOString()`, to avoid off-by-one-day bugs).
   - `broadTopic`: the 6-category bucket derived from `Topic Area` (see **Topic coloring**, below).
4. **Group into blocks** — `buildBlocks()` groups events sharing the same `dateKey + startMinutes + room` into a single "block." For almost every row this is a block of one. The one exception is the poster session: 34 posters share an identical day/time/room, so they collapse into a single block with 34 `items`. This is the mechanism behind the "📌 Poster Session — N of 34" cards you see throughout the UI, and the reason clicking one opens a list instead of a single detail view.
5. **Render** — everything downstream (List view, Concurrent Grid, filters, search, modal) reads from `events` (flat) and `blocks` (grouped).

### Where end times are calculated

Two places, both derived — the sheet has no explicit "end time" column:

- **Per event**, in `rowToEvent()`: `endMinutes: startMinutes + duration`. Straight `StartTime + Duration` math.
- **Per block**, in `buildBlocks()`: `b.endMinutes = Math.max(b.endMinutes, e.endMinutes)`. Blocks share a start time by definition, so this just carries forward the latest end time among a block's items (in practice always the same 20-minute span, since every talk in this dataset is 20 minutes).

Both are stored as raw minute integers and only converted to display strings ("4:00 PM") at render time, via `minutesToLabel()`.

---

## Views

### Schedule List (default)

Chronological cards grouped by day. Each card shows time, title, room, speaker, and type/topic badges. Click a card to open the detail modal. Sortable by Time, Room, Speaker (A–Z), or Topic Area via the **Sort by** control inside the Filters panel.

### Concurrent Grid

Built to answer "what else is happening at the same time." Since every talk in this dataset has an explicit start time and 20-minute duration with no overlaps within a room, the grid is an *exact* representation of concurrency, not an approximation.

Two layouts, toggled via the Stacked/Grid slider (Stacked is the default):

- **Stacked** — time-slot sections, each listing the rooms active at that moment as cards (Title → Speaker → Room). Works well at any screen width.
- **Grid** — a real Room × Time table. Rooms as sticky columns, time as sticky rows, so headers stay visible while scrolling in either direction. On desktop, if the table is wider than the viewport, a shadow scrollbar pins to the bottom of the screen while the grid is in view (see **Sticky bottom scrollbar**, below) so you don't have to scroll all the way down to find the real one.

Because "All Days" is ambiguous in a Room × Time grid, switching into Concurrent Grid disables that option and auto-selects a real day. Switching back to List view re-enables it.

---

## Filters

Everything lives inside one collapsible **Filters** panel (closed by default, to keep the page compact). Reset Filters sits in the panel's header, to the left of the collapse chevron, and clicking it won't collapse the panel.

- **Sort by** — List view only; hidden in Concurrent Grid.
- **Day** — single-select.
- **Room** — multi-select chips.
- **Topic Area** — a nested checklist tree, not flat chips (see below).
- **Presentation Type** — multi-select chips (Oral, Student Oral, Poster, Student Poster, Invited).
- **Color and Icon Key** — a legend, not a filter; explains the topic colors and the student/invited/poster icons.

### Topic Area tree

`Topic Area` values in the sheet look like `"Materials: Polymers"` — a broad category and a specific sub-area. The filter reflects that structure:

- Six broad categories (`Materials`, `Process Development`, `Applications`, `Modeling`, `Special Session`, `Other`), each collapsed by default with a checkbox, a color swatch, and a running count.
- Clicking the checkbox selects/deselects every sub-topic in that category at once, and shows a dash (**partial** state) when only some are selected.
- Clicking anywhere else on the row expands it to reveal individual sub-topics as their own checkbox rows.
- Categories with only one sub-topic (e.g. `Process Development`, which has no colon-separated sub-areas in the source data) skip the expand affordance entirely — the row *is* the checkbox.
- Selected topics also surface as removable chips above the tree, and the "Topic Area" dropdown itself shows a live count badge ("Any" / "N selected") so the active filter state is visible even collapsed.

Filtering itself always operates on the exact `Topic Area` string, never just the broad category — so "select all of Materials" and "select only Polymers" both work precisely, even though they render through the same broad-category checkbox mechanism.

### `broadTopic()` — how the 6 categories are derived

```js
function broadTopic(topic) {
  if (topic.startsWith('Special Session')) return 'Special Session';
  if (topic.startsWith('Materials')) return 'Materials';
  if (topic.startsWith('Process Development')) return 'Process Development';
  if (topic.startsWith('Applications')) return 'Applications';
  if (topic.startsWith('Modeling')) return 'Modeling';
  return 'Other';
}
```

This is a `startsWith` prefix match, not a lookup table — if a future `Topic Area` value doesn't start with one of these five strings, it silently lands in `Other` rather than erroring. Worth checking after any workbook update that introduces new topic wording.

### Sub-topic chip labels

Long `Topic Area` values (e.g. `"Special Session: Wire-Fed AM/Wire-Fed Directed Energy Deposition"`) get shortened for display via `subtopicLabel()`: it strips the `"Broad: "` prefix and any trailing parenthetical explainer (e.g. `"Applications: General (Focus on end-use parts...)"` → `"General"`), but never truncates with an ellipsis. Long labels wrap onto a second line instead — deliberately, since truncated text hidden behind a hover tooltip doesn't work well on mobile or for accessibility. The full, untruncated string is always what's stored in the filter state and shown in the modal.

---

## Topic coloring

Each broad category has a fixed color pair (background + border), applied to event cards, grid mini-cards, and the legend:

| Category | Swatch |
|---|---|
| Materials | soft red |
| Process Development | soft blue |
| Applications | soft green |
| Modeling | soft purple |
| Special Session | soft orange |
| Other | soft tan |

Defined in the `TOPIC_COLORS` constant near the top of the script.

---

## Icons and badges

Font Awesome (loaded from cdnjs, same pattern as the xlsx/pptx CDN scripts) replaced all emoji in v0.02:

- 🎓 → `fa-graduation-cap` — Student Oral / Student Poster
- ⭐ → `fa-star` — Invited
- 📌 → `fa-thumbtack` — Poster Session
- 📍 → `fa-location-dot` — Room
- 🎤 → `fa-microphone-lines` — Speaker
- Plus list/grid/search/close/chevron/arrow icons throughout the UI

---

## Modal

Clicking any event card or grid cell opens a detail modal with title, badges, day/time/room/session/topic, speaker + company, full author list, and the complete abstract text. **The three internal-only columns never appear here or anywhere else.**

Poster sessions open a different modal first — a scrollable list of all posters in that block — since there's no single "the" talk to show. Clicking an individual poster drills into its own full detail view, with a "← Back to poster list" link to return.

---

## Sticky bottom scrollbar

Desktop-only affordance for the Grid layout. When the Room × Time table is wider than the viewport, a second scrollbar element is pinned to the bottom of the *browser window* (not the table) while the grid is anywhere in view, so you can scroll horizontally without hunting for the real scrollbar at the bottom of a tall table. It:

- Only appears when the table actually overflows horizontally.
- Hides itself once the table's real scrollbar has scrolled into view, to avoid showing two scrollbars at once.
- Stays in sync with the real scrollbar in both directions.
- Falls back gracefully (via a small `requestAnimationFrame` polyfill) in any embed context where `requestAnimationFrame` isn't available.

---

## Accessibility

- Base text size is 16px throughout (per EAA guidance), with size differences between primary/secondary/tertiary text handled through font-weight, color, and icons rather than shrinking type.
- Filter dropdowns are native `<details>`/`<summary>` elements — keyboard-operable and screen-reader-friendly with no custom JS needed for open/close.
- The modal closes on Escape, backdrop click, or the close button, and traps focus visually within its bounds.

---

## Deploying / testing

1. Put `index.html` and `SFF2026-Program.xlsx` (exact filename, same case) in the same folder.
2. **GitHub Pages**: put both at the repo root (or same subfolder), enable Pages, done — the relative `DATA_URL` resolves correctly either way.
3. **tms.org**: copy both files into the same folder there. No code changes needed as long as they stay together. If the workbook ends up hosted at a different path, update the `DATA_URL` constant (documented inline in the script).

---

## Known quirks / things to watch

- `broadTopic()` is a prefix match against five hardcoded strings — new/renamed topic areas in future workbook updates could silently fall into "Other" (see above).
- The fetch has no offline fallback or cached copy — if `SFF2026-Program.xlsx` is unreachable, the widget shows the error state and nothing else.
- Poster-block grouping assumes posters sharing identical day/time/room truly are meant to be presented as one session. If a future workbook has two *different* poster sessions at the same time in the same room (unlikely, but possible), they'd merge into one block.
