# Information architecture & sidebar redesign — design

Date: 2026-08-26

## Goal

Restructure the self-assessment tool's information architecture and visual
design:

- A grouped sidebar navigation (About / OS Infrastructure Description /
  assessment framework with expandable sub-sections), replacing today's flat
  link list.
- A new "OS Infrastructure Description" section (name, description, URL,
  assessment date) with "Classify" nested as its sub-heading.
- A new "About this tool" section referencing the SPII project's background
  and goals.
- The data model generalized to a list of assessment frameworks (POSI being
  the first), so a second framework can be added later without another
  restructure.
- A redesigned "Assessment badge" (was "Certification badge"): compact SVG
  (title, %, name, date) plus a hover/focus-revealed detail panel
  ("hovercard").
- Export/import (JSON download/reload, PDF download) moved into a collapsed
  accordion in the sidebar footer, below the badge.
- The activities checklist redrawn as two boxed, multi-select columns.
- Inline SVG status icons (Font-Awesome-styled, not the actual library) on
  the three status cards.

This builds on the already-merged POSI self-assessment implementation
(`docs/superpowers/specs/2026-08-25-posi-self-assessment-design.md`) and the
basic Oat sidebar someone else already added (PR #3, commit `3cf38d9`). It
does **not** implement the separate, still-unstarted
`docs/superpowers/plans/2026-08-26-surf-design-layout.md` (Oat token/spacing
reconciliation) — that plan's concerns (color-token naming, `pt`-vs-`space-*`
units, `.container`/`.row` usage) are complementary and can be picked up
independently; this spec only touches structure/content/data, reusing
whatever CSS conventions are already in the file.

## Out of scope

- Font Awesome (or any icon library) as a CDN dependency — icons are
  hand-authored inline SVG, visually styled like Font Awesome's glyphs, with
  no new runtime dependency (confirmed with user, reversing an initial
  ambiguous request).
- A collapsible icon-only "rail" sidebar mode — the sidebar keeps its
  current full-width layout with the existing mobile show/hide toggle;
  "collapsible ... grouped menu, submenu, and trigger" is satisfied by
  nested expand/collapse groups, not an icon-rail mode (confirmed with
  user — the icon-rail mode is a Curve/React component this project can't
  adopt without a build step).
- Backward compatibility with the old (already-shipped but never-used-by-
  anyone) flat `state.posi` JSON export shape — confirmed with user that no
  migration logic is needed; `merge_loaded_state` only needs to handle the
  new nested shape.
- Implementing a second assessment framework's actual content — only the
  data model is generalized to support one; POSI remains the only framework
  with real content.
- The separate Oat-token-reconciliation plan (see Goal, above).

## Information architecture

### Sidebar (grouped nav, no icon-rail mode)

```
Open Science infrastructure Self-assessment Tool   ← sidebar heading (h2)
──────────────────────────────────────────
▸ About
▸ OS Infrastructure Description
▾ POSI v2.0 Assessment                              ← <details>/<summary>, expanded by default
    Governance
    Sustainability
    Insurance
──────────────────────────────────────────
[Assessment badge — SVG preview + Download badge (SVG)/(PNG) buttons]
▸ Export / import results                           ← <details>/<summary>, collapsed by default
    [Download results as JSON] [Load JSON: ...] [Download report]
```

- "About" and "OS Infrastructure Description" are plain `<a href="#...">`
  links (single level, no sub-items).
- The framework entry ("POSI v2.0 Assessment") is a nested `<details open>`
  inside the nav, its `<summary>` linking to the framework section and its
  children being `<a href="#...">` links to each of the framework's
  sections (Governance/Sustainability/Insurance) — this is the "grouped
  menu, submenu, trigger" pattern, built from the same `<details>`/
  `<summary>` convention already used throughout the page (no new
  interaction pattern to learn).
- Clicking any nav link still closes the mobile sidebar overlay (existing
  `sidebar-nav` click handler behavior, unchanged).

### Main content flow (top to bottom)

1. `<h1>` "Open Science Infrastructure Assessment Tool"
2. Brief one-line description paragraph
3. `<details>` "About this tool" (see Content, below)
4. `<section>` "OS Infrastructure Description" (`<h2>`)
   - Fields: infrastructure name, description (textarea), URL, assessment
     date
   - `<h3>` "Classify" sub-heading, containing the two-column activities
     picker
5. `<section>` per framework (currently just POSI) (`<h2>` = framework
   title)
   - Framework description paragraph
   - `<details>` "About POSI" (see Content, below)
   - One `<section>` per framework section (Governance/Sustainability/
     Insurance), now `<h3>` (demoted from today's `<h2>`, since they're
     nested under the framework's `<h2>`), each with its principles as
     today

## Data model

```js
const FRAMEWORKS = [
  {
    id: "posi",
    title: "POSI v2.0 Assessment",
    shortName: "POSI",
    description: "A community-maintained framework covering governance, sustainability, and insurance against lock-in for scholarly infrastructure.",
    aboutHtml: "<p>...</p>", // see Content section for full text
    sections: [ /* today's POSI array: Governance/Sustainability/Insurance, unchanged */ ],
  },
];

const ACTIVITIES = {
  primary: {
    title: "Primary research life cycle activities",
    items: ["Discover", "Hypothesise", "Plan", "Collect", "Process", "Analyse", "Report"],
  },
  related: {
    title: "Research related activities",
    items: ["Manage & Organise", "Document & Preserve", "Review & Curate", "Publish & Communicate", "Evaluate & Monitor"],
  },
};

function today_iso_date() {
  const d = new Date();
  return d.getFullYear() + "-" + String(d.getMonth() + 1).padStart(2, "0") + "-" + String(d.getDate()).padStart(2, "0");
}

function initial_state() {
  const frameworks = {};
  FRAMEWORKS.forEach((framework) => {
    const posiLike = {};
    framework.sections.forEach((section) => {
      section.principles.forEach((principle) => {
        posiLike[principle.id] = { status: null, notes: "" };
      });
    });
    frameworks[framework.id] = posiLike;
  });
  return {
    name: "", description: "", url: "", assessedOn: today_iso_date(),
    activities: [],
    frameworks,
  };
}
```

- `compute_summary(state, frameworkId)` — now takes a framework id, reads
  `state.frameworks[frameworkId]` instead of the old flat `state.posi`.
- `build_export_object(state)` — serializes the full new shape (`name`,
  `description`, `url`, `assessedOn`, `activities`, `frameworks`).
- `merge_loaded_state(loaded)` — validates/defaults the new shape only
  (`frameworks.posi.<id>.status`/`.notes`, `activities` array, `name`/
  `description`/`url` strings). If `loaded.assessedOn` is missing/invalid,
  default it to `today_iso_date()` (per user: "if no date was entered
  before ... pre-fill the date field with the date of today, but keep it
  editable" — this covers a hand-edited or partial JSON file, not an old
  schema, since no old schema needs supporting).
- Checkbox `value` strings for `ACTIVITIES.primary` items keep the
  `"Primary research life cycle activities > <item>"` group-prefix
  convention (same pattern as today's export shape); `ACTIVITIES.related`
  items use their bare label, matching today's ungrouped items.
- The badge's `RING_CONFIG`/`build_badge_svg_markup` read
  `state.frameworks.posi` (hardcoded to POSI for now, since it's the only
  framework with content — generalizing the badge to an arbitrary framework
  is deferred until a second framework actually exists).

## Components

### Assessment badge + hovercard

SVG layout (compact — only what fits legibly at badge size):
- Top: small text "POSI Compliance"
- Center: large `%` only (the "n/20 scored" line from the prior
  implementation is removed from the visual — that detail moves to the
  hovercard)
- Bottom: two small lines — infrastructure name, then assessment date

Hovercard: a plain `<div class="hovercard">` inside a `position: relative`
wrapper around the badge preview, shown via CSS (`:hover`, `:focus-within`)
— no JS needed beyond keeping its text content in sync in `update_badge()`.
Content: infrastructure name, "Assessed `<date>`", overall `%`, and a
compliant/progress/not-compliant count per framework section (Governance/
Sustainability/Insurance), e.g.:

```
Example Repository
Assessed 2026-08-26
Overall: 72%
Governance: 5✓ 1◐ 1✗
Sustainability: 6✓ 2◐ 0✗
Insurance: 3✓ 1◐ 1✗
```

### Two-column Classify boxes

`render_classify_section()` renders `ACTIVITIES.primary` and
`ACTIVITIES.related` as two side-by-side boxes (Oat `.card`, matching the
site's existing card styling — reusing the class already used for principle
text/status cards, not inventing a new box style), each with its own
heading (`ACTIVITIES.primary.title` / `ACTIVITIES.related.title`) and a
checkbox list of its items. Columns stack to one column on narrow/mobile
widths (matches the existing responsive pattern used for status cards).

### Status icons

Three small inline SVG icons (viewBox ~20×20, single-color `currentColor`
stroke/fill so they inherit each status's color), visually similar to Font
Awesome's `circle-check` / `clock` / `circle-xmark`, defined once as
template strings and inserted into each status card next to its label:
- Compliant → check-circle
- Making progress → clock
- Not compliant → x-circle

### Export/import accordion

A `<details>` titled "Export / import results" in `#sidebar-footer`, below
the badge section, collapsed by default. Contains the existing "Download
results as JSON" button, "Load JSON" file input, and "Download report"
(PDF) button — moved here from the main content's `#actions` section (which
is removed; nothing in main content triggers exports/imports anymore).

## Content

**"About this tool"** (`<details>`, top of main content):

> This tool lets Open Science infrastructure providers, projects, funders,
> and programme teams reflect on and self-assess how open science values
> and principles are put into practice in an infrastructure's design,
> governance, and operation.
>
> It grew out of the SPII project's exploration of a shared self-assessment
> approach for open science infrastructures. Rather than building a bespoke
> framework from scratch, we chose to begin with an existing,
> community-governed framework — the [Principles of Open Scholarly
> Infrastructure (POSI)](https://openscholarlyinfrastructure.org/) — so
> infrastructures can assess themselves against criteria the wider
> community already recognises, and so results can be compared and shared.
> The tool is designed to support additional assessment frameworks over
> time, not just POSI.
>
> The underlying values — quality and integrity, collective benefit, and
> equity, fairness, diversity and inclusion — draw on UNESCO's Open Science
> Recommendation and the Dutch National Plan Open Science (NPOS).

**"About POSI"** (`<details>`, inside the POSI framework section, replaces
today's intro paragraph):

> The Principles of Open Scholarly Infrastructure (POSI) is a
> community-maintained framework (v2.0, released October 2025) covering
> three areas: **Governance** (who's accountable and how),
> **Sustainability** (financial and operational resilience), and
> **Insurance** (protection against lock-in or failure — open source, open
> data, patent non-assertion). Read the full principles at
> [openscholarlyinfrastructure.org](https://openscholarlyinfrastructure.org/).
>
> Cite as: POSI Adopters (2025), *The Principles of Open Scholarly
> Infrastructure*, <https://doi.org/10.14454/G8WV-VM65>.

Both are approved drafts (confirmed with user, with one correction already
applied: "SPII chose" → "we chose to begin with").

## File-level impact

- `index.html` — the only file touched: `<script id="app-logic">` (data
  model), `<script id="app-render">` (rendering/interaction), `<style>`
  (two-column boxes, hovercard, icon sizing — reusing existing card/details
  conventions rather than introducing a new visual language).
- `CLAUDE.md` — architecture description updated for `FRAMEWORKS`, the new
  state shape, the grouped sidebar nav, and the badge/hovercard/accordion
  changes (same pattern as the prior spec's Task 6).

## Testing

No committed test suite (project convention, unchanged). Manual
verification in a browser after implementation:

1. Sidebar: About/OS Infrastructure Description links scroll to the right
   section; the POSI group expands/collapses and its sub-links scroll to
   Governance/Sustainability/Insurance; the Export/import accordion is
   collapsed by default and reveals its three controls when opened.
2. OS Infrastructure Description: fill name/description/URL, confirm the
   assessment date defaults to today and is editable.
3. Classify: confirm the two boxes render side by side on desktop, stack on
   narrow widths, and both groups' checkboxes toggle independently and
   persist through JSON export/reload.
4. Status icons render correctly for all three states, colored via the
   existing status tokens, and remain legible in dark mode.
5. Badge: compact SVG shows title/%/name/date only; hovering or
   keyboard-focusing it reveals the hovercard with the full per-section
   breakdown; both update live as principles are scored.
6. JSON export/reload round-trip preserves name/description/url/
   assessedOn/activities/frameworks exactly.
7. PDF export and the browser's native print preview still work per the
   existing force-open/chrome-hiding behavior, now against the
   restructured markup (re-verify the relevant selectors still match).
8. Dark mode: hovercard, icons, and the two classify boxes all remain
   legible.
