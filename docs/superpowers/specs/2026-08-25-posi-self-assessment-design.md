# POSI-based self-assessment tool — design

Date: 2026-08-25

## Goal

Replace the current SPII values/principles self-assessment (Openness,
Autonomy, Sustainability, Usability) with a self-assessment based on the
[Principles of Open Scholarly Infrastructure (POSI) v2.0](https://openscholarlyinfrastructure.org/).
The assessment flow becomes:

1. Enter the infrastructure's name.
2. Classify the infrastructure against an "activities supported" taxonomy.
3. Score the infrastructure against each POSI principle (compliant / making
   progress / not compliant), with an optional free-text explanation per
   principle.

Output stays downloadable and reloadable, as today, plus a new embeddable
"doughnut" badge image.

Everything remains in the single standalone `index.html` file — no build
step, no framework, no server. Styling continues to come from the vendored
Oat CSS framework, layered with hand-copied color/font tokens from the
[SURF Design System](https://surfnet.github.io/DesignSystem/).

## Out of scope

- The old SPII values/principles content is removed, not kept as a second
  mode (confirmed with user).
- No backend/server component. The "embeddable doughnut" is a downloadable
  static image (SVG/PNG), not a live-rendering embed served from a URL.
- No import of SURF's actual component packages (React/Angular) or their
  compiled stylesheet from a CDN — only their published token *values* are
  hand-copied in as CSS custom properties, keeping the file dependency-free
  aside from the pre-existing jsPDF/html2canvas CDN scripts.
- Conditional/branching logic that changes which POSI principles are shown
  based on the activity taxonomy — the taxonomy is descriptive metadata
  only; every infrastructure answers the same 20 principles.

## Architecture

Move from the current "hand-authored HTML, JS scrapes the DOM" pattern to a
data-driven pattern: POSI content and the activity taxonomy are defined once
as JS data structures, the DOM is rendered from that data at load time, and
a single in-memory `state` object is the source of truth for everything
(rendering, scoring, JSON export/reload, PDF, badge). This trades the
current ~13 repetitive `<details>` blocks for ~20 without repeating markup
by hand, and keeps export/import trivially in sync with what's on screen
since both read/write the same `state` object instead of the DOM.

### Data model

```js
const ACTIVITIES = [
  "Manage & Organise",
  { group: "Research Lifecycle",
    items: ["Discover", "Hypothesise", "Plan", "Collect", "Process", "Analyse", "Report"] },
  "Document & Preserve",
  "Review & Curate",
  "Publish & Communicate",
  "Evaluate & Monitor",
];

const POSI = [
  { section: "Governance", principles: [
    { id: "coverage", title: "Coverage across the scholarly enterprise", text: "…" },
    { id: "stakeholder-governed", title: "Stakeholder governed", text: "…" },
    { id: "non-discriminatory", title: "Non-discriminatory participation or membership", text: "…" },
    { id: "transparent-governance", title: "Transparent governance", text: "…" },
    { id: "cannot-lobby", title: "Cannot lobby", text: "…" },
    { id: "living-will", title: "Living will", text: "…" },
    { id: "regular-review", title: "Regular review of purpose and community value", text: "…" },
  ]},
  { section: "Sustainability", principles: [
    { id: "transparent-operations", title: "Transparent operations", text: "…" },
    { id: "time-limited-funds", title: "Time-limited funds are used only for time-limited activities", text: "…" },
    { id: "generate-surplus", title: "Goal to generate surplus", text: "…" },
    { id: "financial-reserves", title: "Establish and maintain financial reserves guided by policy", text: "…" },
    { id: "mission-consistent-revenue", title: "Mission-consistent revenue generation", text: "…" },
    { id: "revenue-not-data", title: "Revenue generated from services, not data", text: "…" },
    { id: "volunteer-labour", title: "Volunteer labour", text: "…" },
    { id: "transition-planning", title: "Transition planning", text: "…" },
  ]},
  { section: "Insurance", principles: [
    { id: "open-source", title: "Open source", text: "…" },
    { id: "open-secure-data", title: "Ensure open and secure data accessibility within legal and ethical constraints", text: "…" },
    { id: "available-preserved", title: "Available and preserved", text: "…" },
    { id: "patent-non-assertion", title: "Patent non-assertion", text: "…" },
    { id: "interoperability-standards", title: "Prioritise interoperability and open standards to ensure continuity and resilience", text: "…" },
  ]},
];
```

`text` fields will be transcribed verbatim from
openscholarlyinfrastructure.org during implementation (they are quoted
above only where already captured), each principle linking back to the
source page for attribution. This is the complete v2.0 principle set: 7 +
8 + 5 = 20 principles across 3 sections.

Runtime state (single source of truth, in memory):

```js
let state = {
  name: "",
  activities: [],   // selected labels, e.g. ["Manage & Organise", "Research Lifecycle > Collect"]
  posi: {           // keyed by principle id
    "coverage": { status: null, notes: "" },   // status: "compliant" | "progress" | "non-compliant" | null
    // … one entry per principle id
  },
};
```

`render()` rebuilds the whole page from `state`. Every mutation (checkbox
toggle, status card click, textarea edit, JSON load) updates `state` then
calls `render()` (or a narrower `update_summary()` where a full re-render
isn't needed, e.g. on textarea `input` events).

## Page structure

1. **Header** — `<h1>` with the infrastructure name `<input>`, same
   placement as today.
2. **Classify** — one `<section>` rendered from `ACTIVITIES`: a flat list of
   checkboxes, with the Research Lifecycle group rendered as a labelled
   subgroup of checkboxes. Multi-select, purely descriptive — writes to
   `state.activities`.
3. **POSI principles** — one `<section>` per POSI section (Governance,
   Sustainability, Insurance), each containing one `<details>` per
   principle:
   - `<summary>` with the principle title.
   - The principle's description text (quoted, with a link to the POSI
     source page).
   - Three clickable status cards — **Compliant**, **Making progress**,
     **Not compliant** — replacing the old 4 level-N cards. Clicking a card
     sets `state.posi[id].status` and highlights the card (colored border:
     `--compliant` / `--warning` / `--destructive`).
   - A `<textarea>` for the optional free-text explanation, bound to
     `state.posi[id].notes`.
4. **Summary** — near the top (where the values/overall-score row is
   today): overall `% compliant` plus counts, e.g. "12/20 compliant (60%),
   5 making progress, 3 not compliant". Recomputed on every change from
   `state.posi`, counting only principles with a non-null status (same
   "only count what's been scored" behavior as today).
5. **Actions row** — Download JSON, Load JSON (new `<input type="file">`),
   Download PDF, Download badge (SVG), Download badge (PNG).
6. **Doughnut preview** — the live `<svg>` badge, shown on-page above or
   near the actions row so users see what they're about to download.

## Styling

SURF Design System token values (fetched from
`packages/tokens/src/tokens.json` and `tokens.dark.json` in
github.com/SURFnet/DesignSystem) are hand-copied as CSS custom properties
in the existing `<style>` block — no new `<link>`/`<script>` dependency:

```css
:root {
  --primary: rgba(6,75,203,1);
  --warning: rgba(202,138,4,1);
  --destructive: rgba(185,28,28,1);
  --compliant: rgba(13,148,136,1);   /* chart-2 teal, used as the "success" color */
  --font-sans: "Source Sans 3", sans-serif;
}
@media (prefers-color-scheme: dark) {
  :root {
    --primary: rgba(5,62,170,1);
    --warning: rgba(250,204,21,1);
    --destructive: rgba(248,113,113,1);
    --compliant: rgba(16,185,129,1);
  }
}
```

There is no dedicated "success" token in the SURF palette, so the
"compliant" status reuses `chart-2` (teal), which is one of the five
official chart colors — satisfying "make sure to use the chart colors"
while keeping semantic (green-ish = good) meaning. `--warning` and
`--destructive` are used as-is for "making progress" and "not compliant".
Status cards, the summary counts, and the doughnut segments all read from
these same variables.

No external font is loaded (keeps the page standalone/offline-capable);
`--font-sans` is set as a preferred name with a system `sans-serif`
fallback, same approach browsers already take for unavailable fonts.

## JSON export / reload

`download_json()` serializes `state` directly:

```json
{
  "name": "Example Repository",
  "activities": ["Manage & Organise", "Research Lifecycle > Collect"],
  "posi": {
    "coverage": { "status": "compliant", "notes": "..." },
    "...": { "status": null, "notes": "" }
  }
}
```

A new `<input type="file" accept="application/json">` (labelled "Load
JSON") triggers `load_json(file)`: parse the file, replace `state`
(defaulting any missing principle ids to `{status: null, notes: ""}` so an
older/partial JSON file doesn't crash the reload), then call `render()`.
Because rendering, scoring, and the badge all read from `state`, reload is
just "parse → assign → render" — no separate DOM-patching logic to keep in
sync.

## PDF export

Unchanged mechanism: `jsPDF` + `doc.html(document.body, …)`, still loaded
from the existing CDN `<script>` tags. The `beforeprint` listener that
collapses all `<details>` elements continues to apply. `@media print`
rules continue to hide interactive-only chrome (buttons, the file input);
notes text remains visible in print since it's part of the assessment
record.

## Doughnut badge

Three concentric rings — inner = Governance (7 segments), middle =
Sustainability (8), outer = Insurance (5) — each ring divided into
equal-width arcs (one per principle in that section), colored
`--compliant` / `--warning` / `--destructive` per that principle's status,
or a neutral grey for unscored principles. Center text: infrastructure
name (small) and overall `%` compliant (large), matching the summary
figure.

Implementation: an `<svg>` with one `<circle>` per ring, arcs drawn via
`stroke-dasharray`/`stroke-dashoffset` per segment (standard SVG
donut-chart technique — no charting library). The SVG is rendered live
on-page as the preview shown in the Actions section, and is also the
export source:

- **Download SVG** — serialize the live `<svg>` node with
  `XMLSerializer`, download as a self-contained `.svg` file (no external
  refs — safe to embed directly on another site as `<img src="badge.svg">`
  or inline).
- **Download PNG** — draw the same SVG into an offscreen `<canvas>` via an
  `Image`/data-URL round-trip, then `canvas.toDataURL()` for download.

Both are static, point-in-time exports — re-downloading after further
edits produces an updated file; there's no live/dynamic embed since there's
no backend to serve one.

## File-level impact

- `index.html` — full rewrite of `<main>` content and the inline scripts,
  per the above. `README.md` gets a one-line update reflecting the POSI
  basis instead of "SPII project" (still a single standalone HTML file, no
  build step).
- `CLAUDE.md` — update the "Content structure" and "Scoring flow" sections
  to describe the data-driven architecture (state object, render(),
  `ACTIVITIES`/`POSI` data, principle ids) instead of the current
  DOM-scraping/`level-1..4` convention, since that convention no longer
  applies.

## Testing

No test suite (matches project convention). Manual verification in a
browser after implementation:

1. Open `index.html` directly (`file://`) — confirm it loads standalone,
   no console errors, no network requests except the pre-existing jsPDF/
   html2canvas CDN scripts.
2. Fill in name, select several activities (including Research Lifecycle
   sub-items), score a mix of principles across all 3 sections with some
   notes text, leave some unscored.
3. Confirm the summary counts and doughnut update live and match what was
   clicked.
4. Download JSON, reload the page, load that JSON back in — confirm name,
   activities, statuses, and notes are restored exactly, and the doughnut/
   summary recompute correctly.
5. Download the badge as SVG and as PNG — open each standalone (outside
   the page) to confirm they render correctly with no missing references.
6. Download the PDF — confirm it reflects entered data, collapses details
   as expected, and doesn't include the buttons/file input.
7. Toggle OS dark mode and confirm the dark-mode token values apply.
