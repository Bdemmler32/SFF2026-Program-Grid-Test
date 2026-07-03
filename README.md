# SFF2026 Technical Program — Concurrent Schedule

A self-contained, drop-in schedule widget for the SFF2026 Technical Program. One HTML file reads a live-hosted Excel workbook, builds a searchable/filterable schedule, and renders it two ways: a Room × Time **Concurrent Schedule** (the default view) for spotting what's running at the same time, and a chronological **Schedule List**.

No build step, no server-side code, no dependencies beyond two CDN scripts (SheetJS for parsing, Font Awesome for icons). Designed to be dropped into a page on tms.org the same way it runs on GitHub Pages.

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
| `Topic Area` | Filter tree + color coding |
| `Symposium Name` | Not currently displayed (constant across all rows) |
| `Session Name` | Shown as "Session: …" on cards, and in the modal |
| `Date` | Day grouping |
| `StartTime` | Start of the time slot |
| `SesOrder` | **Internal only** — never rendered |
| `Duration` | Minutes; added to `StartTime` to get the end time |
| `RoomName` | Room filter + grid column |
| `Abstract Number` | Sort key within a shared time slot (e.g. ordering posters); not displayed |
| `Abstract Type` | Oral / StudentOral / Poster / StudentPoster / Invited — icon + label, filter |
| `Abstract Title` | Card/modal title |
| `Speaker's ID` | **Internal only** — never rendered |
| `Speaker's Name` | Card/modal |
| `Speaker's Company` | Modal only |
| `All Authors and Affiliations` | Modal only |
| `Abstract` | Modal only (full text) |
| `Academic Status` | **Internal only** — never rendered |

**The three "internal only" columns are read from the sheet but intentionally never displayed anywhere in the UI** — not in cards, not in the modal, not in search results.

### `Admin` sheet (required)

One row, one relevant column: `Revision Date` (e.g. `07/01/2026`). This drives the "Current as of" stamp at the top of the widget (bold italic, 17px). If the sheet or column is missing, the widget silently falls back to today's date rather than erroring.

---

## How the data flows

1. **Fetch** — `fetchProgramData()` pulls the workbook with a cache-busting query param (`?v=<timestamp>`).
2. **Parse** — SheetJS reads the workbook with `cellDates: true`, so Excel date/time cells arrive as native JS `Date` objects.
3. **Normalize** — `rowToEvent()` converts each raw row into a clean event object: `startMinutes`/`endMinutes` (minutes-since-midnight, computed as `StartTime + Duration`), `dateKey` (`YYYY-MM-DD`, using local-timezone getters to avoid off-by-one-day bugs), and `broadTopic` (see **Topic coloring**, below).
4. **Group into blocks** — `buildBlocks()` groups events sharing the same `dateKey + startMinutes + room` into a single "block." Almost every row is a block of one; the exception is the poster session, where 34 posters share an identical day/time/room and collapse into one block with 34 `items`. That's the mechanism behind every "📌 Poster Session — N of 34" card, and why clicking one opens a list instead of a single detail view.
5. **Render** — everything downstream reads from `events` (flat) and `blocks` (grouped).

---

## Views

### Concurrent Schedule (default view on load)

Built to answer "what else is happening at the same time." Since every talk has an explicit start time and 20-minute duration with no overlaps within a room, this is an *exact* representation of concurrency, not an approximation. Because "All Days" is ambiguous in a Room × Time layout, this view auto-selects a real day and disables the "All Days" filter option; switching to Schedule List re-enables it.

Two layouts, toggled via the Grid/Stacked slider in the toolbar (**Grid is the default**, toggle is left-aligned):

- **Grid** — a real Room × Time table. Rooms as sticky columns, time as sticky rows, so headers stay visible while scrolling. On desktop, if the table is wider than the viewport, a shadow scrollbar pins to the bottom of the screen while the grid is in view, so you don't have to scroll down to find the real one.
- **Stacked** — time-slot sections, each listing the rooms active at that moment. Cards here are the *exact same component* as Schedule List cards (see below), just without the time row, since the time is already the section header.

### Schedule List

Chronological cards grouped by day, one column, stacked top to bottom (including when "All Days" is selected — days stack in sequence rather than side by side). Sortable by Time, Room, Speaker (A–Z, by **last name** — see note below), or Topic Area via the **Sort by** control inside the Filters & Key panel.

### Card anatomy (List and Stacked both use this)

Top to bottom:
1. Time range (List only — Stacked omits it, since the section header already shows the time)
2. `Session: <name>` — muted, sits above the title
3. Title — bold, the visual anchor of the card
4. Meta line — icon + text, no pill background, in this order: **Presentation Type → Room → Speaker**

**Topic Area does not appear on cards at all.** The card's background/border tint (see below) already carries the topic color, so a bold pill on top of it would compete with the title for attention. Topic Area only shows — as a colored pill — inside the detail modal, where it's sitting on plain white with nothing to compete against.

### Grid mini-cards (Concurrent Schedule → Grid)

Compact version of the same idea: title, then speaker with a user icon for consistency with the rest of the app.

---

## Filters & Key

Everything lives inside one collapsible **Filters & Key** panel (closed by default). Its open/closed state, and each individual dropdown's state, is remembered in `localStorage` per visitor — so returning visitors see the panel arranged the way they last left it. (This works once the file is actually hosted on GitHub Pages/tms.org; it won't necessarily persist inside a sandboxed file-preview pane, which is a different, more locked-down environment — not a concern for real deployment.)

Reset Filters sits in the panel's header, right-aligned next to the collapse chevron; clicking it won't collapse the panel.

- **Sort by** — List/Stacked only; hidden in Grid layout (which is always time-ordered by definition).
- **Day** — single-select, its own dropdown.
- **Room** — multi-select chips.
- **Topic Area** — a nested checklist tree (see below), not flat chips.
- **Presentation Type** — multi-select chips (Oral, Student Oral, Poster, Student Poster, Invited).
- **Color and Icon Key** — a legend, not a filter. Rendered as two separate rows: topic-color swatches on the first line, then all icon meanings (Room, Speaker, Oral, Student, Invited, Poster) on their own line below — kept apart deliberately so the icon row doesn't wrap awkwardly off the end of the color swatches.

### Topic Area tree

`Topic Area` values in the sheet look like `"Materials: Polymers"` — a broad category and a specific sub-area. The filter reflects that structure as a checklist:

- Six broad categories (`Materials`, `Process Development`, `Applications`, `Modeling`, `Special Session`, `Other`), each collapsed by default with a checkbox, a color swatch, and a running count.
- Clicking the checkbox selects/deselects every sub-topic in that category at once, showing a dash (**partial** state) when only some are selected.
- Clicking anywhere else on the row expands it to reveal individual sub-topics as their own checkbox rows.
- Categories with only one sub-topic (e.g. `Process Development`) skip the expand affordance — the row *is* the checkbox.
- Selected topics surface as removable chips above the tree, and the "Topic Area" dropdown label shows a live count badge — but **only when something is selected**. With nothing selected, no badge shows at all (no "Any" placeholder pill).

Filtering always operates on the exact `Topic Area` string, never just the broad category — "select all of Materials" and "select only Polymers" both work precisely through the same checkbox mechanism.

Long sub-topic labels (e.g. `"Wire-Fed AM/Wire-Fed Directed Energy Deposition"`) get the `"Broad: "` prefix and any trailing parenthetical explainer stripped via `subtopicLabel()`, but are never truncated with an ellipsis — they wrap onto a second line instead, since truncated text behind a hover tooltip is a poor mobile/accessibility pattern.

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

A `startsWith` prefix match, not a lookup table — a future `Topic Area` value that doesn't start with one of these five strings silently lands in `Other` rather than erroring. Worth checking after any workbook update that introduces new topic wording.

---

## Topic coloring

Each broad category has a fixed `{ bg, border, text }` color set in the `TOPIC_COLORS` constant:

| Category | Swatch |
|---|---|
| Materials | soft red |
| Process Development | soft blue |
| Applications | soft green |
| Modeling | soft purple |
| Special Session | soft orange |
| Other | soft tan |

- **Cards** (List/Stacked/Grid) use `bg` for the card background and `border` for the card border — the pale tint.
- **The modal's Topic Area pill** uses `border` as its background and `text` as its label color — the more saturated tone — so it stays visibly distinct against the modal's plain white background rather than blending in.

---

## Icons

Every icon is Font Awesome (loaded from cdnjs). `TYPE_ICONS` maps each presentation type to one:

| Type | Icon |
|---|---|
| Oral | chalkboard-presenter |
| Student Oral / Student Poster | graduation cap |
| Poster | thumbtack |
| Invited | star |

Plus: location-pin for Room, person for Speaker, chevrons for every collapsible dropdown, list/table-cells for the view toggle, magnifying-glass for search, arrow-left for the poster-detail back link, and clock for Jump to Now.

Icon spacing uses `margin-right` on the icon element rather than relying on a literal whitespace character in the template string or a fixed icon `width` — an earlier version constrained icon width to 16px, which caused wider glyphs (graduation-cap, chalkboard-user) to visually overflow into the gap before the following text.

---

## Modal

Clicking any event card or grid cell opens a detail modal with title, badges (type + colored topic pill, each with matching icons), then a meta grid, then speaker/company, full author list, and the complete abstract text. **The three internal-only columns never appear here or anywhere else.**

The meta grid pairs only **Day + Time** as a true 2-column row, since both are reliably short, fixed-format strings. **Room, Session, and Topic Area each get their own full-width row.** This is deliberate: CSS Grid sizes an entire row to its tallest cell, and Session/Topic Area values can run 2–3 lines long — pairing either of them next to a one-word Room value used to stretch the Room cell and leave visible dead space beneath it. Giving each of those three fields its own row avoids that regardless of how long any individual value gets. On mobile, everything collapses to one column anyway.

Poster sessions open a different modal first — a scrollable list of all posters in that block, since there's no single "the" talk to show. Clicking an individual poster drills into its own full detail view, with a "← Back to poster list" link to return.

Opening any modal locks background scroll (the page behind it freezes in place via a `position: fixed` technique that preserves and restores the exact scroll offset) so the page can't be scrolled behind the overlay.

---

## Jump to Now

A button next to search, always visible. On click:
- Reads the visitor's local device clock.
- If today matches one of the three program dates, selects that day and scrolls to the nearest time slot/card to the current time (works across Schedule List, Stacked, and Grid).
- If today isn't a program day, shows a brief inline message instead of doing nothing silently.

This relies on the **visitor's device clock/timezone**, not a hardcoded conference timezone — accurate for someone checking the page from the conference itself, less meaningful for remote browsing outside the event dates. It's manual-only by design (no auto-scroll on page load), so a first-time visitor always lands at the top of the page rather than being scrolled somewhere unexpected.

---

## The "Day filter" link

The helper text shown above the Concurrent Schedule grid/stacked view ("Use the Day filter above to switch days…") contains a real link on "Day filter." Clicking it opens the Filters & Key panel and the Day dropdown specifically (even if both were collapsed), and scrolls the Day dropdown into view.

---

## Sticky bottom scrollbar

Desktop-only, Grid layout only. When the Room × Time table is wider than the viewport, a second scrollbar element pins to the bottom of the *browser window* while the grid is anywhere in view, so horizontal scrolling doesn't require hunting for the real scrollbar at the bottom of a tall table. It only appears when the table actually overflows, hides itself once the real scrollbar is visible (to avoid showing two at once), stays in sync with the real one in both directions, and falls back gracefully if `requestAnimationFrame` or `scrollIntoView` aren't available in a given embed context.

---

## Accessibility

- Base text size is 16px throughout, with hierarchy (title vs. session line vs. meta text) handled through font-weight and color rather than shrinking type below that floor.
- All filter dropdowns, plus the outer Filters & Key panel, are native `<details>`/`<summary>` elements — keyboard-operable and screen-reader-friendly with no custom JS needed for open/close (their state is what gets persisted to `localStorage`).
- The modal closes on Escape, backdrop click, or the close button.

---

## Deploying / testing

1. Put `index.html` and `SFF2026-Program.xlsx` (exact filename, same case) in the same folder.
2. **GitHub Pages**: put both at the repo root (or same subfolder), enable Pages — the relative `DATA_URL` resolves correctly either way.
3. **tms.org**: copy both files into the same folder there. No code changes needed as long as they stay together. If the workbook ends up hosted at a different path, update the `DATA_URL` constant (documented inline in the script).

---

## Known quirks / things to watch

- `broadTopic()` is a prefix match against five hardcoded strings — new/renamed topic areas in future workbook updates could silently fall into "Other."
- The fetch has no offline fallback — if `SFF2026-Program.xlsx` is unreachable, the widget shows the error state and nothing else.
- Poster-block grouping assumes posters sharing identical day/time/room are meant to be one session. Two genuinely different poster sessions at the same time in the same room (unlikely but possible in a future workbook) would merge into one block.
- Sort by Speaker (A–Z) extracts the surname as the last whitespace-separated token in `Speaker's Name`. Works for the vast majority of "First Last" names; won't be perfect for suffixes like "Jr." or multi-word surnames, since the source data doesn't have separate first/last-name columns to sort on directly.
- Jump to Now uses the visitor's own device clock — see the **Jump to Now** section above.
