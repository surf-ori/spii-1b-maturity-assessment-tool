# Multi-framework machinery generalization — design

Date: 2026-08-26

## Goal

Generalize `index.html`'s rendering machinery so it genuinely supports more
than one assessment framework, as a prerequisite for adding a second
framework (SPII v0.0.1 — a separate follow-up plan) without another
structural rewrite. This plan touches only the *machinery*: no new
framework content is added here. POSI remains the only framework with real
content, and its on-screen appearance must not change as a result of this
plan (aside from one pre-existing bug fix, noted below).

## Background: latent bugs this plan fixes

`render_framework_section(framework)` is already called once per
`FRAMEWORKS` entry, but the summary section it renders
(`render_framework_summary()`) produces non-unique DOM ids
(`#summary-heading`, `#summary-counts`), and `update_summary()` is hardcoded
to always compute `compute_summary(state, "posi")` and write to those two
ids via `document.getElementById(...)` (which returns only the *first*
matching element in the document). With only one framework this is
invisible. The moment a second framework section renders, its summary
placeholder would silently never update — a bug that has existed since the
prior plan but never surfaced because only POSI existed. This plan fixes it
as part of making summaries framework-scoped.

A second, more serious latent bug: `render_principle(principle)`,
`set_status(principleId, status)`, and the notes-textarea input listener
are all hardcoded to read/write `state.frameworks.posi[...]` directly,
regardless of which framework's section they were rendered under. This
isn't just a display issue — it's the actual scoring data path. Rendering
a second framework's principles through the current `render_principle`
would either throw (if the principle id doesn't exist in POSI's slice) or,
worse, silently write that principle's status into the *wrong* framework's
state if ids ever collided. This plan threads a `frameworkId` through
`render_framework_content_section` → `render_principle` (stored as
`details.dataset.frameworkId`, alongside the existing
`details.dataset.principleId`) and through `set_status`, so each
principle's status/notes are read from and written to the correct
framework's slice of `state.frameworks`.

## Out of scope

- SPII's actual content (principles, subprinciples, criteria text) — a
  separate, follow-up plan/spec, built on top of this one.
- Persisting `activeFrameworkId` in the exported/reloaded JSON — it's
  view-state, not assessment data, and is intentionally NOT part of
  `build_export_object`/`merge_loaded_state`.
- Any framework beyond generalizing the *shape* — this plan does not add,
  remove, or rename POSI's principles/sections.

## Data model additions

Two new **optional** fields on a principle object, alongside the existing
`id`/`title`/`text`:

```js
{
  id: "...", title: "...", text: "...",
  criteria: {                    // optional
    compliant: "Short rubric describing what 'compliant' looks like.",
    progress: "Short rubric describing what 'making progress' looks like.",
    "non-compliant": "Short rubric describing what 'not compliant' looks like.",
  },
  improvementAction: "One or two sentences on how to move up the scale.",  // optional
}
```

POSI's principles will not define `criteria`/`improvementAction` — their
absence must render exactly as today (status cards showing only their
label, no extra text, no "To improve:" note).

One new **optional** field on a framework object:

```js
{ id: "...", title: "...", shortName: "...", description: "...", aboutHtml: "...", sections: [...], draft: true }
```

`draft` is omitted (falsy) for POSI.

## Rendering changes

**Status cards** (`render_principle`): when `principle.criteria` is
present, each status card additionally shows its
`criteria[option.status]` text (e.g. as a small paragraph under the
label). When absent, cards render exactly as they do today — this must be
verified explicitly for POSI.

**Improvement note**: when `principle.improvementAction` is present, a
short "To improve:" line renders below the notes `<textarea>`. Absent for
POSI's principles.

**Framework summary**: `render_framework_summary(framework)` (gains a
parameter) produces ids `#summary-heading-<framework.id>` /
`#summary-counts-<framework.id>`. `update_summary()` iterates
`FRAMEWORKS`, calling `compute_summary(state, framework.id)` for each and
writing to that framework's scoped ids — not just `"posi"`.

**Draft label**: `render_framework_section(framework)` shows a small
"Draft — work in progress" label next to the framework's `<h2>` title when
`framework.draft` is true, and `render_sidebar_nav()` shows an equivalent
small marker next to that framework's nav entry. The old global
`<span id="watermark">` fixed diagonal overlay and its CSS are removed
entirely — replaced by this per-framework mechanism.

## Active framework & badge generalization

- New module-level state: `let activeFrameworkId = FRAMEWORKS[0].id;`
  (defaults to the first framework — POSI, until a second is added).
- Clicking a framework's nav link, or any of its section sub-links, sets
  `activeFrameworkId` to that framework's id and calls `update_badge()`.
- `build_badge_svg_markup(state)`, `build_badge_hovercard_html(state)`, and
  their internal `compute_summary(state, "posi")` calls all read
  `activeFrameworkId` instead of the literal string `"posi"`.
- The badge's title text (currently the hardcoded "POSI Compliance")
  becomes `<framework.shortName> Compliance`.
- `RING_CONFIG` (currently a hardcoded 3-entry array with explicit radii
  for Governance/Sustainability/Insurance) is replaced by a function that
  computes N rings from `activeFramework.sections.length` — e.g. starting
  radius + a fixed increment per ring index — so it scales correctly
  whether the active framework has 3 sections (POSI) or a different count
  (a future framework), with no per-framework-name branching.

## File-level impact

- `index.html` only — `<script id="app-logic">` (new optional principle/
  framework fields — no change to POSI's actual data, since the fields are
  additive and optional), `<script id="app-render">` (all the rendering/
  state changes above), `<style>` (remove `#watermark` rule, add a small
  draft-label style), and `<body>` (remove the static
  `<span id="watermark">...</span>`).
- `CLAUDE.md` — architecture description updated for the generalized
  summary/badge/draft-label mechanism and the new optional principle
  fields.

## Testing

No committed test suite (project convention, unchanged). Manual
verification in a browser after implementation:

1. With only POSI present: confirm the page looks and behaves *identically*
   to before this plan — no draft label anywhere, status cards show only
   their label (no extra criteria text), no "To improve:" note, summary
   updates correctly, badge shows "POSI Compliance" and POSI's 3-ring
   layout unchanged.
2. Manually add a second, minimal test framework entry to `FRAMEWORKS`
   temporarily (2-3 sections, at least one principle with `criteria`/
   `improvementAction` set, `draft: true`) to verify: both frameworks' own
   summaries update independently (the bug this plan fixes); clicking each
   framework's nav link switches the badge's ring count, title, and data to
   match; the draft label appears only on the test framework's section and
   nav entry; a principle with `criteria` shows the extra rubric text per
   card and the "To improve:" note; a principle without them renders
   exactly like POSI's. Remove the temporary test entry before finishing
   (or leave removal as the very last verification step) — it must not
   ship.
3. JSON export/reload still round-trips correctly and does NOT include
   `activeFrameworkId`.
