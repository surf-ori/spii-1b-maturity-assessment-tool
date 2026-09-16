# IA & Sidebar Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure `index.html`'s information architecture: a grouped sidebar nav (About / OS Infrastructure Description / expandable POSI Assessment with sub-links), a new "OS Infrastructure Description" section (name/description/URL/date + a two-column "Classify" sub-section), a generalized `FRAMEWORKS` data model (POSI as the first entry), a redesigned "Assessment badge" (compact SVG + hover/focus "hovercard" with full detail), export/import moved into a collapsed sidebar-footer accordion, and inline SVG status icons.

**Architecture:** Single standalone `index.html` file, no build step, no framework (unchanged). The existing `state`-driven `render()` pattern is extended, not replaced: new render functions for the new sections, a generalized `FRAMEWORKS`/`state.frameworks` shape (POSI is `frameworks.posi`), and a `compute_summary(state, frameworkId)` signature change. No new runtime dependency (icons are hand-authored inline SVG, not a library; the hovercard is a plain CSS-shown `<div>`, not a new component).

**Tech Stack:** Vanilla HTML/CSS/JS, Oat CSS (`oat.min.css`, vendored), jsPDF + html2canvas (CDN, pre-existing), no npm/build tooling.

**Spec:** `docs/superpowers/specs/2026-08-26-ia-sidebar-redesign-design.md`

## Global Constraints

- Everything lives in one file, `index.html` — no build step, no package manager, no server, no JS framework.
- No new runtime dependency: icons are hand-authored inline SVG visually styled like Font Awesome, not the actual library or any CDN. The hovercard is plain CSS (`:hover`/`:focus-within`), no new component library.
- No collapsible icon-only "rail" sidebar mode — the sidebar keeps its current full-width layout and existing mobile show/hide toggle; "grouped menu, submenu, trigger" is satisfied by nested `<details>`/`<summary>` groups.
- No backward compatibility needed for the old flat `state.posi` JSON export shape — `merge_loaded_state` only needs to handle the new nested `state.frameworks` shape (confirmed: no one has used the tool yet).
- Only one framework (`posi`) ships real content in this pass; the data model is generalized to a `FRAMEWORKS` list, but the badge/summary/hovercard may stay hardcoded to `"posi"` rather than iterate all frameworks generically.
- Classify column titles: left = "Primary research life cycle activities", right = "Research related activities" (exact copy, confirmed with user).
- Badge SVG shows only: title "POSI Compliance", the `%` in the center, infrastructure name, and assessment date — no "n/20 scored" line (that detail moves to the hovercard).
- Content for "About this tool" and "About POSI" is given verbatim in Task 2 and Task 3 below (already approved by the user, including the "we chose to begin with" correction) — do not paraphrase or shorten it.

---

### Task 1: Data model migration (`FRAMEWORKS`, `ACTIVITIES`, assessment date)

**Files:**
- Modify: `index.html` (`<script id="app-logic">` and several small call-site fixes in `<script id="app-render">`)
- Test: none committed — throwaway Node script in the scratchpad, per existing project convention

**Interfaces:**
- Consumes: nothing new (builds on Task 1 of the prior plan: `POSI` array, `ring_segment_offsets`, `escape_xml` — unchanged, do not touch `POSI`'s content).
- Produces:
  - `FRAMEWORKS: Array<{id: string, title: string, shortName: string, description: string, aboutHtml: string, sections: typeof POSI}>` (one entry, `id: "posi"`)
  - `ACTIVITIES: {primary: {title: string, items: string[]}, related: {title: string, items: string[]}}` (replaces the old array shape)
  - `ABOUT_TOOL_HTML: string`
  - `today_iso_date(): string` (returns `"YYYY-MM-DD"`)
  - `initial_state(): {name: string, description: string, url: string, assessedOn: string, activities: string[], frameworks: {[frameworkId: string]: {[principleId: string]: {status: string|null, notes: string}}}}`
  - `compute_summary(state, frameworkId: string): {compliant, progress, nonCompliant, scoredCount, totalCount, percentCompliant}` (frameworkId is now REQUIRED — this changes the call signature from the previous plan)
  - `build_export_object(state)`, `merge_loaded_state(loaded)` — updated for the new shape

This task's goal: migrate the data model and fix every existing call site that breaks as a result, so the page keeps working exactly as it does today (no visual changes yet — those are Tasks 2-5).

- [ ] **Step 1: Write the throwaway verification script**

Create a file `verify-logic-2.mjs` anywhere outside the repo (e.g. your scratchpad, or `/tmp`) with:

```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

// Run from the repository root so this relative path resolves.
const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
assert.ok(match, 'app-logic script block not found');
const context = vm.createContext({});
vm.runInContext(match[1], context);

// FRAMEWORKS shape
assert.equal(context.FRAMEWORKS.length, 1);
assert.equal(context.FRAMEWORKS[0].id, 'posi');
assert.equal(context.FRAMEWORKS[0].sections.length, 3);
assert.equal(typeof context.FRAMEWORKS[0].aboutHtml, 'string');
assert.ok(context.FRAMEWORKS[0].aboutHtml.length > 0);

// ACTIVITIES shape
assert.equal(context.ACTIVITIES.primary.title, 'Primary research life cycle activities');
assert.deepEqual(context.ACTIVITIES.primary.items, ['Discover', 'Hypothesise', 'Plan', 'Collect', 'Process', 'Analyse', 'Report']);
assert.equal(context.ACTIVITIES.related.title, 'Research related activities');
assert.deepEqual(context.ACTIVITIES.related.items, ['Manage & Organise', 'Document & Preserve', 'Review & Curate', 'Publish & Communicate', 'Evaluate & Monitor']);

// today_iso_date
const date = context.today_iso_date();
assert.match(date, /^\d{4}-\d{2}-\d{2}$/);

// initial_state
const state = context.initial_state();
assert.equal(state.name, '');
assert.equal(state.description, '');
assert.equal(state.url, '');
assert.equal(state.assessedOn, date);
assert.deepEqual(state.activities, []);
assert.equal(Object.keys(state.frameworks).length, 1);
assert.equal(Object.keys(state.frameworks.posi).length, 20);

// compute_summary requires a frameworkId now
state.frameworks.posi['coverage'].status = 'compliant';
state.frameworks.posi['stakeholder-governed'].status = 'compliant';
state.frameworks.posi['cannot-lobby'].status = 'progress';
state.frameworks.posi['living-will'].status = 'non-compliant';
const summary = context.compute_summary(state, 'posi');
assert.equal(summary.compliant, 2);
assert.equal(summary.progress, 1);
assert.equal(summary.nonCompliant, 1);
assert.equal(summary.scoredCount, 4);
assert.equal(summary.totalCount, 20);
assert.equal(summary.percentCompliant, 50);

// build_export_object / merge_loaded_state round trip
const exported = context.build_export_object(state);
assert.equal(exported.frameworks.posi['coverage'].status, 'compliant');
assert.equal(exported.assessedOn, date);

const merged = context.merge_loaded_state({
  name: 'Test Repo',
  description: 'A test infrastructure',
  url: 'https://example.org',
  assessedOn: '2026-01-15',
  activities: ['Primary research life cycle activities > Discover'],
  frameworks: { posi: { coverage: { status: 'compliant', notes: 'good' }, unknown_id: { status: 'compliant' } } },
});
assert.equal(merged.name, 'Test Repo');
assert.equal(merged.assessedOn, '2026-01-15');
assert.deepEqual(merged.activities, ['Primary research life cycle activities > Discover']);
assert.equal(merged.frameworks.posi['coverage'].status, 'compliant');
assert.equal(merged.frameworks.posi['coverage'].notes, 'good');
assert.equal(merged.frameworks.posi['stakeholder-governed'].status, null);
assert.equal('unknown_id' in merged.frameworks.posi, false);

// missing/invalid assessedOn defaults to today
const mergedNoDate = context.merge_loaded_state({ name: 'X' });
assert.equal(mergedNoDate.assessedOn, date);

const mergedBad = context.merge_loaded_state(null);
assert.equal(mergedBad.name, '');
assert.equal(Object.keys(mergedBad.frameworks.posi).length, 20);

console.log('All data-model checks passed.');
```

- [ ] **Step 2: Run it to verify it fails**

Run (from the repository root): `node /path/to/verify-logic-2.mjs`
Expected: FAIL — `context.FRAMEWORKS` is undefined (assertion throws), since `index.html` doesn't have `FRAMEWORKS`/the new `ACTIVITIES` shape yet.

- [ ] **Step 3: Replace `ACTIVITIES` and add `FRAMEWORKS`/`ABOUT_TOOL_HTML`/`today_iso_date`**

In `<script id="app-logic">`, replace the existing `ACTIVITIES` array:

```js
            const ACTIVITIES = [
                "Manage & Organise",
                { group: "Research Lifecycle", items: ["Discover", "Hypothesise", "Plan", "Collect", "Process", "Analyse", "Report"] },
                "Document & Preserve",
                "Review & Curate",
                "Publish & Communicate",
                "Evaluate & Monitor",
            ];
```

with:

```js
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

            const ABOUT_TOOL_HTML =
                "<p>This tool lets Open Science infrastructure providers, projects, funders, and programme teams reflect on and self-assess how open science values and principles are put into practice in an infrastructure’s design, governance, and operation.</p>" +
                "<p>It grew out of the SPII project’s exploration of a shared self-assessment approach for open science infrastructures. Rather than building a bespoke framework from scratch, we chose to begin with an existing, community-governed framework — the <a href=\"https://openscholarlyinfrastructure.org/\">Principles of Open Scholarly Infrastructure (POSI)</a> — so infrastructures can assess themselves against criteria the wider community already recognises, and so results can be compared and shared. The tool is designed to support additional assessment frameworks over time, not just POSI.</p>" +
                "<p>The underlying values — quality and integrity, collective benefit, and equity, fairness, diversity and inclusion — draw on UNESCO’s Open Science Recommendation and the Dutch National Plan Open Science (NPOS).</p>";

            function today_iso_date() {
                const d = new Date();
                return d.getFullYear() + "-" + String(d.getMonth() + 1).padStart(2, "0") + "-" + String(d.getDate()).padStart(2, "0");
            }
```

Then, immediately after the closing `];` of the existing `POSI` array (leave `POSI`'s content completely untouched — do not edit any principle text), add:

```js

            const FRAMEWORKS = [
                {
                    id: "posi",
                    title: "POSI v2.0 Assessment",
                    shortName: "POSI",
                    description: "A community-maintained framework covering governance, sustainability, and insurance against lock-in for scholarly infrastructure.",
                    aboutHtml:
                        "<p>The Principles of Open Scholarly Infrastructure (POSI) is a community-maintained framework (v2.0, released October 2025) covering three areas: <strong>Governance</strong> (who’s accountable and how), <strong>Sustainability</strong> (financial and operational resilience), and <strong>Insurance</strong> (protection against lock-in or failure — open source, open data, patent non-assertion). Read the full principles at <a href=\"https://openscholarlyinfrastructure.org/\">openscholarlyinfrastructure.org</a>.</p>" +
                        "<p>Cite as: POSI Adopters (2025), <em>The Principles of Open Scholarly Infrastructure</em>, <a href=\"https://doi.org/10.14454/G8WV-VM65\">https://doi.org/10.14454/G8WV-VM65</a>.</p>",
                    sections: POSI,
                },
            ];
```

- [ ] **Step 4: Rewrite `initial_state`, `compute_summary`, `build_export_object`, `merge_loaded_state`**

Replace the existing `initial_state()`:

```js
            function initial_state() {
                const posi = {};
                POSI.forEach((section) => {
                    section.principles.forEach((principle) => {
                        posi[principle.id] = { status: null, notes: "" };
                    });
                });
                return { name: "", activities: [], posi };
            }
```

with:

```js
            function initial_state() {
                const frameworks = {};
                FRAMEWORKS.forEach((framework) => {
                    const entries = {};
                    framework.sections.forEach((section) => {
                        section.principles.forEach((principle) => {
                            entries[principle.id] = { status: null, notes: "" };
                        });
                    });
                    frameworks[framework.id] = entries;
                });
                return {
                    name: "",
                    description: "",
                    url: "",
                    assessedOn: today_iso_date(),
                    activities: [],
                    frameworks,
                };
            }
```

Replace `compute_summary(state)`:

```js
            function compute_summary(state) {
                const entries = Object.values(state.posi);
```

with (only the signature and the first line change; the rest of the function body is identical):

```js
            function compute_summary(state, frameworkId) {
                const entries = Object.values(state.frameworks[frameworkId]);
```

Replace `build_export_object(state)`:

```js
            function build_export_object(state) {
                const posi = {};
                Object.keys(state.posi).forEach((id) => {
                    posi[id] = { status: state.posi[id].status, notes: state.posi[id].notes };
                });
                return { name: state.name, activities: state.activities.slice(), posi };
            }
```

with:

```js
            function build_export_object(state) {
                const frameworks = {};
                Object.keys(state.frameworks).forEach((frameworkId) => {
                    const entries = {};
                    Object.keys(state.frameworks[frameworkId]).forEach((id) => {
                        entries[id] = { status: state.frameworks[frameworkId][id].status, notes: state.frameworks[frameworkId][id].notes };
                    });
                    frameworks[frameworkId] = entries;
                });
                return {
                    name: state.name,
                    description: state.description,
                    url: state.url,
                    assessedOn: state.assessedOn,
                    activities: state.activities.slice(),
                    frameworks,
                };
            }
```

Replace `merge_loaded_state(loaded)`:

```js
            function merge_loaded_state(loaded) {
                const fresh = initial_state();
                if (!loaded || typeof loaded !== "object") return fresh;
                if (typeof loaded.name === "string") fresh.name = loaded.name;
                if (Array.isArray(loaded.activities)) fresh.activities = loaded.activities.slice();
                if (loaded.posi && typeof loaded.posi === "object") {
                    Object.keys(fresh.posi).forEach((id) => {
                        const entry = loaded.posi[id];
                        if (entry && typeof entry === "object") {
                            fresh.posi[id].status = ["compliant", "progress", "non-compliant"].includes(entry.status) ? entry.status : null;
                            fresh.posi[id].notes = typeof entry.notes === "string" ? entry.notes : "";
                        }
                    });
                }
                return fresh;
            }
```

with:

```js
            function merge_loaded_state(loaded) {
                const fresh = initial_state();
                if (!loaded || typeof loaded !== "object") return fresh;
                if (typeof loaded.name === "string") fresh.name = loaded.name;
                if (typeof loaded.description === "string") fresh.description = loaded.description;
                if (typeof loaded.url === "string") fresh.url = loaded.url;
                fresh.assessedOn = typeof loaded.assessedOn === "string" && loaded.assessedOn ? loaded.assessedOn : today_iso_date();
                if (Array.isArray(loaded.activities)) fresh.activities = loaded.activities.slice();
                if (loaded.frameworks && typeof loaded.frameworks === "object") {
                    Object.keys(fresh.frameworks).forEach((frameworkId) => {
                        const loadedFramework = loaded.frameworks[frameworkId];
                        if (!loadedFramework || typeof loadedFramework !== "object") return;
                        Object.keys(fresh.frameworks[frameworkId]).forEach((id) => {
                            const entry = loadedFramework[id];
                            if (entry && typeof entry === "object") {
                                fresh.frameworks[frameworkId][id].status = ["compliant", "progress", "non-compliant"].includes(entry.status) ? entry.status : null;
                                fresh.frameworks[frameworkId][id].notes = typeof entry.notes === "string" ? entry.notes : "";
                            }
                        });
                    });
                }
                return fresh;
            }
```

- [ ] **Step 5: Run the verification script again to confirm it passes**

Run: `node /path/to/verify-logic-2.mjs`
Expected: PASS — prints `All data-model checks passed.`

- [ ] **Step 6: Fix the now-broken call sites in `<script id="app-render">`**

These six edits are mechanical renames only (no visual/behavioral change) — they exist so the page keeps working after this task, before Tasks 2-5 land.

In `update_summary()`, change:
```js
            function update_summary() {
                const summary = compute_summary(state);
```
to:
```js
            function update_summary() {
                const summary = compute_summary(state, "posi");
```

In `set_status()`, change:
```js
            function set_status(principleId, status) {
                state.posi[principleId].status = state.posi[principleId].status === status ? null : status;
                const details = document.querySelector('[data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    const isSelected = card.dataset.status === state.posi[principleId].status;
```
to:
```js
            function set_status(principleId, status) {
                state.frameworks.posi[principleId].status = state.frameworks.posi[principleId].status === status ? null : status;
                const details = document.querySelector('[data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    const isSelected = card.dataset.status === state.frameworks.posi[principleId].status;
```

In `render_principle(principle)`, change the three `state.posi[principle.id]` references — the `aria-pressed` line, the `if (state.posi...)` line, and the notes value line — to `state.frameworks.posi[principle.id]`. The full function should read:
```js
            function render_principle(principle) {
                const details = document.createElement("details");
                details.className = "container";
                details.dataset.principleId = principle.id;

                const summary = document.createElement("summary");
                const h3 = document.createElement("h3");
                h3.textContent = principle.title;
                summary.appendChild(h3);
                details.appendChild(summary);

                const text = document.createElement("p");
                text.textContent = principle.text;
                details.appendChild(text);

                const cards = document.createElement("div");
                cards.className = "status-cards";
                cards.setAttribute("role", "group");
                cards.setAttribute("aria-label", principle.title + " maturity status");
                [
                    { status: "compliant", label: "Compliant" },
                    { status: "progress", label: "Making progress" },
                    { status: "non-compliant", label: "Not compliant" },
                ].forEach((option) => {
                    const card = document.createElement("article");
                    card.className = "card col-3 status-card";
                    card.dataset.status = option.status;
                    card.setAttribute("role", "button");
                    card.setAttribute("tabindex", "0");
                    card.setAttribute("aria-pressed", state.frameworks.posi[principle.id].status === option.status ? "true" : "false");
                    if (state.frameworks.posi[principle.id].status === option.status) {
                        card.classList.add("selected");
                    }
                    const h4 = document.createElement("h4");
                    h4.textContent = option.label;
                    card.appendChild(h4);
                    cards.appendChild(card);
                });
                details.appendChild(cards);

                const notes = document.createElement("textarea");
                notes.className = "notes";
                notes.placeholder = "Explain your reasoning (optional)";
                notes.value = state.frameworks.posi[principle.id].notes;
                details.appendChild(notes);

                return details;
            }
```

Near the bottom of the script, in the one-time `appRoot` listener setup, change:
```js
            appRoot.addEventListener("input", (e) => {
                if (e.target.matches("textarea.notes")) {
                    const details = e.target.closest("[data-principle-id]");
                    state.posi[details.dataset.principleId].notes = e.target.value;
                }
            });
```
to:
```js
            appRoot.addEventListener("input", (e) => {
                if (e.target.matches("textarea.notes")) {
                    const details = e.target.closest("[data-principle-id]");
                    state.frameworks.posi[details.dataset.principleId].notes = e.target.value;
                }
            });
```

In `build_badge_svg_markup(state)`, change:
```js
                    section.principles.forEach((principle, i) => {
                        const status = state.posi[principle.id].status;
```
to:
```js
                    section.principles.forEach((principle, i) => {
                        const status = state.frameworks.posi[principle.id].status;
```
and change:
```js
                const summary = compute_summary(state);
```
to:
```js
                const summary = compute_summary(state, "posi");
```
(This function's visual output is redesigned in Task 4 — this step only fixes the data access so it doesn't throw in the meantime.)

Finally, replace `render_classify_section()` and keep `render_activity_checkbox()` as-is (unchanged, still used) — this is a minimal adaptation to the new `ACTIVITIES` shape using the OLD flat-list visual style; Task 2 replaces this function entirely with the two-column boxed version. Replace:
```js
            function render_classify_section() {
                const section = document.createElement("section");
                section.id = "classify-section";
                const h2 = document.createElement("h2");
                h2.textContent = "Classify";
                section.appendChild(h2);
                const p = document.createElement("p");
                p.textContent = "This infrastructure supports the following activities:";
                section.appendChild(p);

                const list = document.createElement("div");
                ACTIVITIES.forEach((item) => {
                    if (typeof item === "string") {
                        list.appendChild(render_activity_checkbox(item, item));
                    } else {
                        const groupWrap = document.createElement("div");
                        const groupLabel = document.createElement("strong");
                        groupLabel.textContent = item.group;
                        groupWrap.appendChild(groupLabel);
                        const groupList = document.createElement("div");
                        groupList.className = "taxonomy-group";
                        item.items.forEach((sub) => {
                            const value = item.group + " > " + sub;
                            groupList.appendChild(render_activity_checkbox(value, sub));
                        });
                        groupWrap.appendChild(groupList);
                        list.appendChild(groupWrap);
                    }
                });
                section.appendChild(list);
                return section;
            }
```
with:
```js
            function render_classify_section() {
                const section = document.createElement("section");
                section.id = "classify-section";
                const h2 = document.createElement("h2");
                h2.textContent = "Classify";
                section.appendChild(h2);
                const p = document.createElement("p");
                p.textContent = "This infrastructure supports the following activities:";
                section.appendChild(p);

                const list = document.createElement("div");
                [ACTIVITIES.primary, ACTIVITIES.related].forEach((group) => {
                    const groupWrap = document.createElement("div");
                    const groupLabel = document.createElement("strong");
                    groupLabel.textContent = group.title;
                    groupWrap.appendChild(groupLabel);
                    const groupList = document.createElement("div");
                    groupList.className = "taxonomy-group";
                    group.items.forEach((item) => {
                        const value = group.title + " > " + item;
                        groupList.appendChild(render_activity_checkbox(value, item));
                    });
                    groupWrap.appendChild(groupList);
                    list.appendChild(groupWrap);
                });
                section.appendChild(list);
                return section;
            }
```

- [ ] **Step 7: Manually verify in a browser**

Open `index.html` directly. Confirm the page looks and behaves EXACTLY as before this task (no visual change is expected yet): name input works, classify checkboxes work (still one flat list per group, not yet boxed), scoring principles works, notes work, the summary line updates, the badge preview renders (still showing the old 3-line %/scored/name layout — visual redesign is Task 4), JSON download/reload round-trips correctly (open the downloaded file and confirm it has `name`/`description`/`url`/`assessedOn`/`activities`/`frameworks.posi` — `description`/`url` will be empty strings since there's no UI for them yet, that's expected), and no console errors appear.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Migrate data model to FRAMEWORKS/ACTIVITIES groups and assessment date"
```

(Do not commit the scratchpad verification script.)

---

### Task 2: Main content restructure — header, OS Infrastructure Description, framework section

**Files:**
- Modify: `index.html` (`<script id="app-render">`, `<style>`)

**Interfaces:**
- Consumes: `FRAMEWORKS`, `ACTIVITIES`, `ABOUT_TOOL_HTML`, `state` (Task 1).
- Produces:
  - `render_header(): HTMLElement` (rewritten — no longer contains the name input)
  - `render_infra_description_section(): HTMLElement` (new)
  - `render_text_field(id, labelText, value, type, placeholder): HTMLElement`
  - `render_textarea_field(id, labelText, value, placeholder): HTMLElement`
  - `render_activities_column(group): HTMLElement`
  - `render_framework_section(framework): HTMLElement` (new)
  - `render_framework_summary(): HTMLElement` (renamed from `render_summary_section`)
  - `render_framework_content_section(section): HTMLElement` (renamed from `render_posi_section`, heading demoted `h2`→`h3`)
  - `posi_section_id(section)` — unchanged, reused
  - DOM ids later tasks rely on: `#about-section` (the About-this-tool `<details>`), `#infra-description-section`, `#framework-posi`, `#infra-name`, `#infra-description`, `#infra-url`, `#infra-assessed-on`

This task removes `render_classify_section()`/`render_summary_section()`/`render_posi_section()`'s old top-level call sites from `render()` in favor of the new structure. `render_classify_section()` itself (added in Task 1) is deleted, replaced by logic inside `render_infra_description_section()`.

- [ ] **Step 1: Rewrite `render_header()`**

Replace:
```js
            function render_header() {
                const wrap = document.createElement("div");

                const toggleBtn = document.createElement("button");
                toggleBtn.type = "button";
                toggleBtn.id = "sidebar-toggle";
                toggleBtn.setAttribute("data-sidebar-toggle", "");
                toggleBtn.setAttribute("aria-label", "Toggle navigation");
                toggleBtn.setAttribute("aria-expanded", "false");
                toggleBtn.textContent = "☰ Menu";
                wrap.appendChild(toggleBtn);

                const h1 = document.createElement("h1");
                h1.appendChild(document.createTextNode("Open Science infrastructure self-assessment for "));
                const nameInput = document.createElement("input");
                nameInput.type = "text";
                nameInput.id = "infra-name";
                nameInput.placeholder = "name of the infrastructure";
                nameInput.style.display = "inline-block";
                nameInput.style.width = "200pt";
                nameInput.value = state.name;
                h1.appendChild(nameInput);
                wrap.appendChild(h1);

                const intro = document.createElement("p");
                intro.innerHTML = 'Based on the <a href="https://openscholarlyinfrastructure.org/">Principles of Open Scholarly Infrastructure (POSI) v2.0</a>. Cite as: POSI Adopters (2025), The Principles of Open Scholarly Infrastructure, <a href="https://doi.org/10.14454/G8WV-VM65">https://doi.org/10.14454/G8WV-VM65</a>.';
                wrap.appendChild(intro);
                return wrap;
            }
```
with:
```js
            function render_header() {
                const wrap = document.createElement("div");

                const toggleBtn = document.createElement("button");
                toggleBtn.type = "button";
                toggleBtn.id = "sidebar-toggle";
                toggleBtn.setAttribute("data-sidebar-toggle", "");
                toggleBtn.setAttribute("aria-label", "Toggle navigation");
                toggleBtn.setAttribute("aria-expanded", "false");
                toggleBtn.textContent = "☰ Menu";
                wrap.appendChild(toggleBtn);

                const h1 = document.createElement("h1");
                h1.textContent = "Open Science Infrastructure Assessment Tool";
                wrap.appendChild(h1);

                const intro = document.createElement("p");
                intro.textContent = "Self-assess how your Open Science infrastructure puts open science values and principles into practice, against one or more community-recognised assessment frameworks.";
                wrap.appendChild(intro);

                const about = document.createElement("details");
                about.id = "about-section";
                about.innerHTML = "<summary>About this tool</summary>" + ABOUT_TOOL_HTML;
                wrap.appendChild(about);

                return wrap;
            }
```

- [ ] **Step 2: Delete `render_classify_section()`, add `render_infra_description_section()` and its field helpers**

Delete the entire `render_classify_section()` function (added in Task 1) — it is fully replaced by the code below. Keep `render_activity_checkbox(value, labelText)` — it's reused, unchanged.

Add these new functions (place them where `render_classify_section()` used to be):

```js
            function render_infra_description_section() {
                const section = document.createElement("section");
                section.id = "infra-description-section";
                const h2 = document.createElement("h2");
                h2.textContent = "OS Infrastructure Description";
                section.appendChild(h2);

                section.appendChild(render_text_field("infra-name", "Infrastructure name", state.name, "text", "name of the infrastructure"));
                section.appendChild(render_textarea_field("infra-description", "Description", state.description, "brief description of the infrastructure"));
                section.appendChild(render_text_field("infra-url", "Landing page URL", state.url, "url", "https://..."));
                section.appendChild(render_text_field("infra-assessed-on", "Assessment date", state.assessedOn, "date", ""));

                const classifyHeading = document.createElement("h3");
                classifyHeading.textContent = "Classify";
                section.appendChild(classifyHeading);
                const classifyIntro = document.createElement("p");
                classifyIntro.textContent = "This infrastructure supports the following activities:";
                section.appendChild(classifyIntro);

                const columns = document.createElement("div");
                columns.className = "classify-columns";
                columns.appendChild(render_activities_column(ACTIVITIES.primary));
                columns.appendChild(render_activities_column(ACTIVITIES.related));
                section.appendChild(columns);

                return section;
            }

            function render_text_field(id, labelText, value, type, placeholder) {
                const label = document.createElement("label");
                label.className = "field";
                label.htmlFor = id;
                const span = document.createElement("span");
                span.textContent = labelText;
                label.appendChild(span);
                const input = document.createElement("input");
                input.type = type;
                input.id = id;
                input.value = value;
                if (placeholder) input.placeholder = placeholder;
                label.appendChild(input);
                return label;
            }

            function render_textarea_field(id, labelText, value, placeholder) {
                const label = document.createElement("label");
                label.className = "field";
                label.htmlFor = id;
                const span = document.createElement("span");
                span.textContent = labelText;
                label.appendChild(span);
                const textarea = document.createElement("textarea");
                textarea.id = id;
                textarea.value = value;
                if (placeholder) textarea.placeholder = placeholder;
                label.appendChild(textarea);
                return label;
            }

            function render_activities_column(group) {
                const box = document.createElement("div");
                box.className = "card classify-column";
                const h4 = document.createElement("h4");
                h4.textContent = group.title;
                box.appendChild(h4);
                group.items.forEach((item) => {
                    const value = group.title + " > " + item;
                    box.appendChild(render_activity_checkbox(value, item));
                });
                return box;
            }
```

- [ ] **Step 3: Rename `render_summary_section` → `render_framework_summary`, `render_posi_section` → `render_framework_content_section`, add `render_framework_section`**

Replace:
```js
            function render_summary_section() {
                const section = document.createElement("section");
                section.id = "summary-section";
                const heading = document.createElement("strong");
                heading.id = "summary-heading";
                heading.textContent = "Overall: 0% compliant";
                const counts = document.createElement("span");
                counts.id = "summary-counts";
                section.appendChild(heading);
                section.appendChild(counts);
                return section;
            }
```
with:
```js
            function render_framework_summary() {
                const section = document.createElement("section");
                section.id = "summary-section";
                const heading = document.createElement("strong");
                heading.id = "summary-heading";
                heading.textContent = "Overall: 0% compliant";
                const counts = document.createElement("span");
                counts.id = "summary-counts";
                section.appendChild(heading);
                section.appendChild(counts);
                return section;
            }

            function render_framework_section(framework) {
                const section = document.createElement("section");
                section.id = "framework-" + framework.id;
                const h2 = document.createElement("h2");
                h2.textContent = framework.title;
                section.appendChild(h2);

                const desc = document.createElement("p");
                desc.textContent = framework.description;
                section.appendChild(desc);

                const about = document.createElement("details");
                about.innerHTML = "<summary>About " + framework.shortName + "</summary>" + framework.aboutHtml;
                section.appendChild(about);

                section.appendChild(render_framework_summary());

                framework.sections.forEach((frameworkSection) => {
                    section.appendChild(render_framework_content_section(frameworkSection));
                });

                return section;
            }
```

Replace:
```js
            function render_posi_section(section) {
                const el = document.createElement("section");
                el.id = posi_section_id(section);
                const h2 = document.createElement("h2");
                h2.textContent = section.section;
                el.appendChild(h2);
                const sourceNote = document.createElement("p");
                sourceNote.innerHTML = 'Source: <a href="' + section.sourceUrl + '">' + section.sourceUrl + "</a>";
                el.appendChild(sourceNote);
                section.principles.forEach((principle) => {
                    el.appendChild(render_principle(principle));
                });
                return el;
            }
```
with:
```js
            function render_framework_content_section(section) {
                const el = document.createElement("section");
                el.id = posi_section_id(section);
                const h3 = document.createElement("h3");
                h3.textContent = section.section;
                el.appendChild(h3);
                const sourceNote = document.createElement("p");
                sourceNote.innerHTML = 'Source: <a href="' + section.sourceUrl + '">' + section.sourceUrl + "</a>";
                el.appendChild(sourceNote);
                section.principles.forEach((principle) => {
                    el.appendChild(render_principle(principle));
                });
                return el;
            }
```

`posi_section_id(section)` is unchanged — leave it exactly as-is.

- [ ] **Step 4: Update `render()`**

Replace:
```js
            function render() {
                const app = document.getElementById("app");
                app.innerHTML = "";
                app.appendChild(render_header());
                app.appendChild(render_summary_section());
                app.appendChild(render_classify_section());
                POSI.forEach((section) => app.appendChild(render_posi_section(section)));
                app.appendChild(render_actions_section());
                render_sidebar_nav();
                render_sidebar_footer();
                wire_up_events();
                update_summary();
                update_badge();
            }
```
with:
```js
            function render() {
                const app = document.getElementById("app");
                app.innerHTML = "";
                app.appendChild(render_header());
                app.appendChild(render_infra_description_section());
                FRAMEWORKS.forEach((framework) => app.appendChild(render_framework_section(framework)));
                app.appendChild(render_actions_section());
                render_sidebar_nav();
                render_sidebar_footer();
                wire_up_events();
                update_summary();
                update_badge();
            }
```

- [ ] **Step 5: Update `wire_up_events()` for the new fields**

Replace:
```js
            function wire_up_events() {
                const nameInput = document.getElementById("infra-name");
                nameInput.addEventListener("input", (e) => {
                    state.name = e.target.value;
                    update_badge();
                });

                document.getElementById("classify-section").addEventListener("change", (e) => {
                    if (e.target.matches('input[type="checkbox"]')) {
                        toggle_activity(e.target.value, e.target.checked);
                    }
                });

                document.getElementById("sidebar-toggle").addEventListener("click", (e) => {
                    const isOpen = document.getElementById("app-shell").toggleAttribute("data-sidebar-open");
                    e.currentTarget.setAttribute("aria-expanded", String(isOpen));
                });
            }
```
with:
```js
            function wire_up_events() {
                const nameInput = document.getElementById("infra-name");
                nameInput.addEventListener("input", (e) => {
                    state.name = e.target.value;
                    update_badge();
                });

                document.getElementById("infra-description").addEventListener("input", (e) => {
                    state.description = e.target.value;
                });

                document.getElementById("infra-url").addEventListener("input", (e) => {
                    state.url = e.target.value;
                });

                document.getElementById("infra-assessed-on").addEventListener("input", (e) => {
                    state.assessedOn = e.target.value;
                    update_badge();
                });

                document.getElementById("infra-description-section").addEventListener("change", (e) => {
                    if (e.target.matches('input[type="checkbox"]')) {
                        toggle_activity(e.target.value, e.target.checked);
                    }
                });

                document.getElementById("sidebar-toggle").addEventListener("click", (e) => {
                    const isOpen = document.getElementById("app-shell").toggleAttribute("data-sidebar-open");
                    e.currentTarget.setAttribute("aria-expanded", String(isOpen));
                });
            }
```

- [ ] **Step 6: Add CSS for the new fields and columns**

In `<style>`, add (anywhere after the existing `.taxonomy-group` rule is a natural spot, though exact position doesn't matter):

```css
            .classify-columns {
                display: flex;
                gap: 16pt;
                flex-wrap: wrap;
            }

            .classify-column {
                flex: 1 1 240pt;
            }

            .field {
                display: block;
                margin-bottom: 8pt;
            }

            .field span {
                display: block;
                font-weight: bold;
                margin-bottom: 2pt;
            }

            .field input,
            .field textarea {
                width: 100%;
            }
```

- [ ] **Step 7: Manually verify in a browser**

Open `index.html`. Confirm: the `<h1>` reads "Open Science Infrastructure Assessment Tool" with a brief description below it, "About this tool" is a closed-by-default accordion with the SPII/background text; "OS Infrastructure Description" shows name/description/URL/assessment-date fields (date pre-filled to today) all editable, with "Classify" as an `<h3>` sub-heading below them showing two side-by-side boxed columns ("Primary research life cycle activities" / "Research related activities") that stack on a narrow window; the "POSI v2.0 Assessment" section shows its description, a closed "About POSI" accordion, then Governance/Sustainability/Insurance as `<h3>` sub-sections with their principles unchanged; typing in any of the 4 new fields updates `state` (check via browser devtools or by exporting JSON and confirming the values appear); no console errors.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Restructure main content: header, OS Infrastructure Description, framework sections"
```

---

### Task 3: Sidebar nav — grouped, nested structure

**Files:**
- Modify: `index.html` (static markup in `<body>`, and `<script id="app-render">`)

**Interfaces:**
- Consumes: `FRAMEWORKS` (Task 1), `#about-section`/`#infra-description-section`/`#framework-posi` ids and `posi_section_id()` (Task 2).
- Produces: `render_sidebar_nav(): void` (rewritten), `render_nav_link(href, label): HTMLElement` (new helper).

- [ ] **Step 1: Update the sidebar heading**

The sidebar's `<h2>` is static HTML, not generated by any render function. Find, in `<body>`:
```html
        <div data-sidebar-layout id="app-shell">
            <aside data-sidebar>
                <header>
                    <h2>POSI Assessment</h2>
                </header>
```
and change the heading text to:
```html
        <div data-sidebar-layout id="app-shell">
            <aside data-sidebar>
                <header>
                    <h2>Open Science infrastructure Self-assessment Tool</h2>
                </header>
```

- [ ] **Step 2: Rewrite `render_sidebar_nav()`**

Replace:
```js
            function render_sidebar_nav() {
                const nav = document.getElementById("sidebar-nav");
                nav.innerHTML = "";
                const ul = document.createElement("ul");
                const links = [{ href: "#classify-section", label: "Classify" }].concat(
                    POSI.map((section) => ({ href: "#" + posi_section_id(section), label: section.section }))
                );
                links.forEach((link) => {
                    const li = document.createElement("li");
                    const a = document.createElement("a");
                    a.href = link.href;
                    a.textContent = link.label;
                    li.appendChild(a);
                    ul.appendChild(li);
                });
                nav.appendChild(ul);
            }
```
with:
```js
            function render_sidebar_nav() {
                const nav = document.getElementById("sidebar-nav");
                nav.innerHTML = "";
                const ul = document.createElement("ul");

                ul.appendChild(render_nav_link("#about-section", "About"));
                ul.appendChild(render_nav_link("#infra-description-section", "OS Infrastructure Description"));

                FRAMEWORKS.forEach((framework) => {
                    const li = document.createElement("li");
                    const groupDetails = document.createElement("details");
                    groupDetails.open = true;
                    const summary = document.createElement("summary");
                    const a = document.createElement("a");
                    a.href = "#framework-" + framework.id;
                    a.textContent = framework.title;
                    summary.appendChild(a);
                    groupDetails.appendChild(summary);

                    const subUl = document.createElement("ul");
                    framework.sections.forEach((section) => {
                        subUl.appendChild(render_nav_link("#" + posi_section_id(section), section.section));
                    });
                    groupDetails.appendChild(subUl);

                    li.appendChild(groupDetails);
                    ul.appendChild(li);
                });

                nav.appendChild(ul);
            }

            function render_nav_link(href, label) {
                const li = document.createElement("li");
                const a = document.createElement("a");
                a.href = href;
                a.textContent = label;
                li.appendChild(a);
                return li;
            }
```

- [ ] **Step 3: Manually verify in a browser**

Open `index.html`. Confirm the sidebar heading reads "Open Science infrastructure Self-assessment Tool", and the nav below it shows, in order: "About" link, "OS Infrastructure Description" link, then an expanded group "POSI v2.0 Assessment" (a link itself) with three sub-links (Governance/Sustainability/Insurance) nested under it. Clicking each link scrolls/jumps to the right section (About's accordion, the infra description fields, the framework section, or the specific Governance/Sustainability/Insurance sub-section). On mobile width, opening the sidebar and clicking any link still closes the sidebar overlay (existing behavior, unchanged — confirm it still works with the new nested markup).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Restructure sidebar nav into grouped, nested sections"
```

---

### Task 4: Sidebar footer — badge redesign, hovercard, export/import accordion

**Files:**
- Modify: `index.html` (`<script id="app-render">`, `<style>`)

**Interfaces:**
- Consumes: `state`, `FRAMEWORKS`, `compute_summary`, `escape_xml`, `ring_segment_offsets`, `RING_CONFIG`, `STATUS_COLOR_VAR`, `status_color` (all from Task 1 / prior plan, unchanged except call-site fixes already applied in Task 1).
- Produces: `render_sidebar_footer()` (rewritten), `render_export_import_accordion(): HTMLElement` (new), `build_badge_hovercard_html(state): string` (new), `build_badge_svg_markup(state)` (redesigned), `update_badge()` (extended), `download_badge_png()` (canvas size updated). Removes `render_actions_section()` entirely (its buttons move into `render_export_import_accordion()`).

- [ ] **Step 1: Rewrite `render_sidebar_footer()` and add `render_export_import_accordion()`**

Replace:
```js
            function render_sidebar_footer() {
                const footer = document.getElementById("sidebar-footer");
                footer.innerHTML = "";

                const heading = document.createElement("h3");
                heading.textContent = "Certification badge";
                footer.appendChild(heading);

                const preview = document.createElement("div");
                preview.id = "badge-preview";
                footer.appendChild(preview);

                const svgBtn = document.createElement("button");
                svgBtn.textContent = "Download badge (SVG)";
                svgBtn.addEventListener("click", download_badge_svg);
                footer.appendChild(svgBtn);

                const pngBtn = document.createElement("button");
                pngBtn.textContent = "Download badge (PNG)";
                pngBtn.addEventListener("click", download_badge_png);
                footer.appendChild(pngBtn);
            }
```
with:
```js
            function render_sidebar_footer() {
                const footer = document.getElementById("sidebar-footer");
                footer.innerHTML = "";

                const heading = document.createElement("h3");
                heading.textContent = "Assessment badge";
                footer.appendChild(heading);

                const badgeWrap = document.createElement("div");
                badgeWrap.className = "badge-wrap";
                badgeWrap.tabIndex = 0;
                const preview = document.createElement("div");
                preview.id = "badge-preview";
                badgeWrap.appendChild(preview);
                const hovercard = document.createElement("div");
                hovercard.id = "badge-hovercard";
                hovercard.className = "hovercard";
                badgeWrap.appendChild(hovercard);
                footer.appendChild(badgeWrap);

                const svgBtn = document.createElement("button");
                svgBtn.textContent = "Download badge (SVG)";
                svgBtn.addEventListener("click", download_badge_svg);
                footer.appendChild(svgBtn);

                const pngBtn = document.createElement("button");
                pngBtn.textContent = "Download badge (PNG)";
                pngBtn.addEventListener("click", download_badge_png);
                footer.appendChild(pngBtn);

                footer.appendChild(render_export_import_accordion());
            }

            function render_export_import_accordion() {
                const details = document.createElement("details");
                details.id = "export-import-accordion";
                const summary = document.createElement("summary");
                summary.textContent = "Export / import results";
                details.appendChild(summary);

                const downloadJsonBtn = document.createElement("button");
                downloadJsonBtn.textContent = "Download results as JSON";
                downloadJsonBtn.addEventListener("click", download_json);
                details.appendChild(downloadJsonBtn);

                const loadJsonLabel = document.createElement("label");
                loadJsonLabel.textContent = "Load JSON: ";
                const loadJsonInput = document.createElement("input");
                loadJsonInput.type = "file";
                loadJsonInput.id = "load-json-input";
                loadJsonInput.accept = "application/json";
                loadJsonInput.addEventListener("change", (e) => {
                    const file = e.target.files[0];
                    if (file) load_json(file);
                });
                loadJsonLabel.appendChild(loadJsonInput);
                details.appendChild(loadJsonLabel);

                const downloadPdfBtn = document.createElement("button");
                downloadPdfBtn.textContent = "Download report";
                downloadPdfBtn.addEventListener("click", download_pdf);
                details.appendChild(downloadPdfBtn);

                return details;
            }
```

- [ ] **Step 2: Delete `render_actions_section()` and its call in `render()`**

Delete the entire `render_actions_section()` function (its three controls now live in `render_export_import_accordion()`, added in Step 1). In `render()`, remove the line `app.appendChild(render_actions_section());` — the function should now read:

```js
            function render() {
                const app = document.getElementById("app");
                app.innerHTML = "";
                app.appendChild(render_header());
                app.appendChild(render_infra_description_section());
                FRAMEWORKS.forEach((framework) => app.appendChild(render_framework_section(framework)));
                render_sidebar_nav();
                render_sidebar_footer();
                wire_up_events();
                update_summary();
                update_badge();
            }
```

`download_json`, `load_json`, and `download_pdf` themselves are unchanged — only where their buttons live changes.

Also remove the now-dead `#actions` selector from both print-CSS rules (the element no longer exists; the rest of each rule — `#sidebar-footer button`, `[data-sidebar-toggle]`, etc. — still correctly hides the same controls since they still live under `#sidebar-footer`). In `<style>`, change:
```css
            @media print {
                #actions,
                #sidebar-footer button,
```
to:
```css
            @media print {
                #sidebar-footer button,
```
and change:
```css
            body.printing #actions,
            body.printing #sidebar-footer button,
```
to:
```css
            body.printing #sidebar-footer button,
```

- [ ] **Step 3: Redesign `build_badge_svg_markup(state)`**

Replace:
```js
            function build_badge_svg_markup(state) {
                const size = 200;
                const center = size / 2;
                const strokeWidth = 10;
                let circles = "";
                RING_CONFIG.forEach((ring) => {
                    const section = POSI.find((s) => s.section === ring.section);
                    const offsets = ring_segment_offsets(ring.radius, section.principles.length);
                    section.principles.forEach((principle, i) => {
                        const status = state.frameworks.posi[principle.id].status;
                        const color = status_color(status);
                        const off = offsets[i];
                        circles +=
                            '<circle cx="' + center + '" cy="' + center + '" r="' + ring.radius +
                            '" fill="none" stroke="' + color + '" stroke-width="' + strokeWidth +
                            '" stroke-dasharray="' + off.dasharray + '" stroke-dashoffset="' + off.dashoffset +
                            '" transform="rotate(-90 ' + center + " " + center + ')" />';
                    });
                });
                const summary = compute_summary(state, "posi");
                const name = escape_xml(state.name || "Unnamed infrastructure");
                return (
                    '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ' + size + " " + size + '" width="' + size + '" height="' + size + '">' +
                    '<rect width="' + size + '" height="' + size + '" fill="white" />' +
                    circles +
                    '<text x="' + center + '" y="' + (center - 8) + '" text-anchor="middle" font-size="20" font-family="&quot;Source Sans 3&quot;, sans-serif" font-weight="bold">' + summary.percentCompliant + "%</text>" +
                    '<text x="' + center + '" y="' + (center + 6) + '" text-anchor="middle" font-size="7" font-family="&quot;Source Sans 3&quot;, sans-serif">' + summary.scoredCount + "/" + summary.totalCount + " scored</text>" +
                    '<text x="' + center + '" y="' + (center + 16) + '" text-anchor="middle" font-size="7" font-family="&quot;Source Sans 3&quot;, sans-serif">' + name + "</text>" +
                    "</svg>"
                );
            }
```
with:
```js
            function build_badge_svg_markup(state) {
                const width = 200;
                const height = 260;
                const center = width / 2;
                const ringCenterY = 140;
                const strokeWidth = 10;
                let circles = "";
                const framework = FRAMEWORKS.find((f) => f.id === "posi");
                RING_CONFIG.forEach((ring) => {
                    const section = framework.sections.find((s) => s.section === ring.section);
                    const offsets = ring_segment_offsets(ring.radius, section.principles.length);
                    section.principles.forEach((principle, i) => {
                        const status = state.frameworks.posi[principle.id].status;
                        const color = status_color(status);
                        const off = offsets[i];
                        circles +=
                            '<circle cx="' + center + '" cy="' + ringCenterY + '" r="' + ring.radius +
                            '" fill="none" stroke="' + color + '" stroke-width="' + strokeWidth +
                            '" stroke-dasharray="' + off.dasharray + '" stroke-dashoffset="' + off.dashoffset +
                            '" transform="rotate(-90 ' + center + " " + ringCenterY + ')" />';
                    });
                });
                const summary = compute_summary(state, "posi");
                const name = escape_xml(state.name || "Unnamed infrastructure");
                const date = escape_xml(state.assessedOn || "");
                return (
                    '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ' + width + " " + height + '" width="' + width + '" height="' + height + '">' +
                    '<rect width="' + width + '" height="' + height + '" fill="white" />' +
                    '<text x="' + center + '" y="24" text-anchor="middle" font-size="12" font-family="&quot;Source Sans 3&quot;, sans-serif" font-weight="bold">POSI Compliance</text>' +
                    circles +
                    '<text x="' + center + '" y="' + (ringCenterY + 7) + '" text-anchor="middle" font-size="26" font-family="&quot;Source Sans 3&quot;, sans-serif" font-weight="bold">' + summary.percentCompliant + "%</text>" +
                    '<text x="' + center + '" y="228" text-anchor="middle" font-size="11" font-family="&quot;Source Sans 3&quot;, sans-serif">' + name + "</text>" +
                    '<text x="' + center + '" y="244" text-anchor="middle" font-size="10" font-family="&quot;Source Sans 3&quot;, sans-serif">' + date + "</text>" +
                    "</svg>"
                );
            }
```

- [ ] **Step 4: Update `download_badge_png()`'s canvas size to match the taller badge**

Find, inside `download_badge_png()`:
```js
                    canvas.width = 200;
                    canvas.height = 200;
```
Change to:
```js
                    canvas.width = 200;
                    canvas.height = 260;
```

- [ ] **Step 5: Extend `update_badge()` and add `build_badge_hovercard_html(state)`**

Replace:
```js
            function update_badge() {
                const container = document.getElementById("badge-preview");
                if (!container) return;
                container.innerHTML = build_badge_svg_markup(state);
            }
```
with:
```js
            function update_badge() {
                const container = document.getElementById("badge-preview");
                if (!container) return;
                container.innerHTML = build_badge_svg_markup(state);
                const hovercard = document.getElementById("badge-hovercard");
                if (hovercard) hovercard.innerHTML = build_badge_hovercard_html(state);
            }

            function build_badge_hovercard_html(state) {
                const summary = compute_summary(state, "posi");
                const name = state.name || "Unnamed infrastructure";
                let html = "<strong>" + escape_xml(name) + "</strong>";
                html += "<div>Assessed " + escape_xml(state.assessedOn) + "</div>";
                html += "<div>Overall: " + summary.percentCompliant + "%</div>";
                const framework = FRAMEWORKS.find((f) => f.id === "posi");
                framework.sections.forEach((section) => {
                    let compliant = 0;
                    let progress = 0;
                    let nonCompliant = 0;
                    section.principles.forEach((principle) => {
                        const status = state.frameworks.posi[principle.id].status;
                        if (status === "compliant") compliant++;
                        else if (status === "progress") progress++;
                        else if (status === "non-compliant") nonCompliant++;
                    });
                    html += "<div>" + escape_xml(section.section) + ": " + compliant + "✓ " + progress + "◐ " + nonCompliant + "✗</div>";
                });
                return html;
            }
```

- [ ] **Step 6: Add CSS for the hovercard**

In `<style>`, add:

```css
            .badge-wrap {
                position: relative;
                display: inline-block;
            }

            .hovercard {
                position: absolute;
                left: 0;
                top: 100%;
                z-index: 10;
                background: var(--card);
                color: var(--foreground);
                border: 1pt solid var(--border);
                border-radius: 4pt;
                padding: 8pt;
                font-size: 9pt;
                width: 220pt;
                opacity: 0;
                visibility: hidden;
                transition: opacity 0.15s ease;
                pointer-events: none;
            }

            .badge-wrap:hover .hovercard,
            .badge-wrap:focus-within .hovercard {
                opacity: 1;
                visibility: visible;
            }
```

- [ ] **Step 7: Manually verify in a browser**

Score a spread of principles and fill in the name/date. Confirm: the sidebar footer heading reads "Assessment badge"; the badge SVG shows "POSI Compliance" at the top, a large `%` in the center of the rings, and the infrastructure name + assessment date at the bottom (no "n/20 scored" line); hovering (or Tab-focusing) the badge reveals the hovercard with name, assessed date, overall %, and a per-section compliant/progress/not-compliant count; "Export / import results" is a collapsed-by-default accordion in the footer containing working "Download results as JSON", "Load JSON", and "Download report" controls (confirm each still works exactly as before — round-trip a JSON export/reload, and download a PDF); the old `#actions` section no longer appears in the main content. Also re-run the browser-based verification from the original POSI plan's Task 4/5 (print preview, PDF download) to confirm the moved buttons are still correctly hidden by the `#sidebar-footer button` selector in both the `@media print` and `body.printing` rules (they still live under `#sidebar-footer`, just nested one level deeper inside the new accordion).

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Redesign assessment badge with hovercard, move export/import into sidebar accordion"
```

---

### Task 5: Status icons

**Files:**
- Modify: `index.html` (`<script id="app-logic">` for the icon data, `<script id="app-render">` for `render_principle`, `<style>`)

**Interfaces:**
- Consumes: `render_principle(principle)` (Task 1, already migrated to `state.frameworks.posi`).
- Produces: `ICONS: {compliant: string, progress: string, "non-compliant": string}` (new, in `app-logic`).

- [ ] **Step 1: Add `ICONS` to `<script id="app-logic">`**

Add near `ABOUT_TOOL_HTML`/`today_iso_date` (anywhere at the top level of the script is fine):

```js
            const ICONS = {
                compliant: '<svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6"><circle cx="10" cy="10" r="8.5" /><path d="M6 10.5l2.5 2.5L14 7.5" stroke-linecap="round" stroke-linejoin="round" /></svg>',
                progress: '<svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6"><circle cx="10" cy="10" r="8.5" /><path d="M10 5.5v5l3 2" stroke-linecap="round" stroke-linejoin="round" /></svg>',
                "non-compliant": '<svg viewBox="0 0 20 20" width="18" height="18" fill="none" stroke="currentColor" stroke-width="1.6"><circle cx="10" cy="10" r="8.5" /><path d="M7 7l6 6M13 7l-6 6" stroke-linecap="round" /></svg>',
            };
```

- [ ] **Step 2: Insert icons into each status card in `render_principle(principle)`**

Find the `.forEach((option) => { ... })` loop inside `render_principle` (built in Task 1's rewrite) and change:
```js
                    const h4 = document.createElement("h4");
                    h4.textContent = option.label;
                    card.appendChild(h4);
                    cards.appendChild(card);
```
to:
```js
                    const icon = document.createElement("span");
                    icon.className = "status-icon";
                    icon.innerHTML = ICONS[option.status];
                    card.appendChild(icon);

                    const h4 = document.createElement("h4");
                    h4.textContent = option.label;
                    card.appendChild(h4);
                    cards.appendChild(card);
```

- [ ] **Step 3: Add CSS to color the icons per status**

In `<style>`, add:

```css
            .status-icon {
                display: block;
            }

            .status-card[data-status="compliant"] .status-icon { color: var(--compliant); }
            .status-card[data-status="progress"] .status-icon { color: var(--warning); }
            .status-card[data-status="non-compliant"] .status-icon { color: var(--destructive); }
```

- [ ] **Step 4: Manually verify in a browser**

Confirm each status card shows its icon (check-circle / clock / x-circle) above or beside its label, tinted teal/amber/red respectively regardless of whether that card is currently selected, and that the icons remain legible in dark mode (they use `currentColor`/CSS custom properties, so they should follow the existing light/dark token values automatically — toggle OS dark mode to confirm).

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add inline SVG status icons to maturity level cards"
```

---

### Task 6: Update project documentation

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:** none (documentation only).

- [ ] **Step 1: Update `CLAUDE.md`'s Architecture section**

Read the current `## Architecture` section of `CLAUDE.md`. Replace the **Data model** paragraph:

```markdown
**Data model** (`<script id="app-logic">` in `<head>`): `ACTIVITIES` is the "supports the following activities" taxonomy (flat labels plus one grouped entry for the Research Lifecycle sub-stages). `POSI` is the full Principles of Open Scholarly Infrastructure v2.0 content (3 sections — Governance, Sustainability, Insurance — 20 principles total), each principle with a stable `id`, `title`, and verbatim `text` transcribed from openscholarlyinfrastructure.org. `initial_state()` builds a fresh `state` object: `{name, activities, posi: {[id]: {status, notes}}}`, where `status` is `"compliant"`, `"progress"`, `"non-compliant"`, or `null`. `compute_summary(state)`, `build_export_object(state)`, `merge_loaded_state(loaded)`, and `ring_segment_offsets(radius, count)` are pure functions with no DOM access — this is what makes them testable outside a browser (see Testing below).
```

with:

```markdown
**Data model** (`<script id="app-logic">` in `<head>`): `FRAMEWORKS` is a list of assessment frameworks — currently one entry, POSI v2.0 (`id: "posi"`), each with a `title`/`description`/`aboutHtml` and a `sections` array (3 sections — Governance, Sustainability, Insurance — 20 principles total; the raw content lives in the `POSI` constant, referenced by `FRAMEWORKS[0].sections`). Each principle has a stable `id`, `title`, and verbatim `text` transcribed from openscholarlyinfrastructure.org. `ACTIVITIES` is `{primary, related}` — two named, boxed groups for the "Classify" picker ("Primary research life cycle activities" and "Research related activities"). `ICONS` holds the three inline SVG status icons. `initial_state()` builds a fresh `state` object: `{name, description, url, assessedOn, activities, frameworks: {[frameworkId]: {[principleId]: {status, notes}}}}`, where `status` is `"compliant"`, `"progress"`, `"non-compliant"`, or `null`, and `assessedOn` defaults to today's date (`today_iso_date()`). `compute_summary(state, frameworkId)`, `build_export_object(state)`, `merge_loaded_state(loaded)`, and `ring_segment_offsets(radius, count)` are pure functions with no DOM access — this is what makes them testable outside a browser (see Testing below). Only `"posi"` has content today, but the shape supports adding a second framework without another data-model restructure.
```

Replace the **Rendering** paragraph's mention of the old section structure — find:

```markdown
**Rendering** (`<script id="app-render">` in `<body>`): `render()` rebuilds `#app` entirely from the global `state` — name input, activity checkboxes, one `<details data-principle-id="...">` per POSI principle (3 status cards + a notes `<textarea>`), the live summary, the doughnut badge preview, and the actions row. Interaction handlers (`set_status`, `toggle_activity`, the notes/name input listeners) mutate `state` directly and call the narrower `update_summary()`/`update_badge()` rather than a full `render()`, so typing in a textarea doesn't lose focus. `render()` itself is only called on first load and after a JSON file is loaded (`load_json`), since those are the only times the whole DOM tree needs rebuilding from a new `state`.
```

with:

```markdown
**Rendering** (`<script id="app-render">` in `<body>`): `render()` rebuilds `#app` entirely from the global `state` — the header (title + "About this tool" accordion), the "OS Infrastructure Description" section (name/description/URL/assessment-date fields, plus "Classify" as a sub-heading with two boxed activity columns), one section per `FRAMEWORKS` entry (title, description, "About `<framework>`" accordion, framework-level summary, then one `<details data-principle-id="...">` per principle — 3 status cards with icons + a notes `<textarea>`). The grouped sidebar nav (`render_sidebar_nav`) and footer (`render_sidebar_footer`: the assessment badge + hovercard, then a collapsed "Export / import results" accordion holding the JSON/PDF controls) are rebuilt on every `render()` too. Interaction handlers (`set_status`, `toggle_activity`, the notes/field input listeners) mutate `state` directly and call the narrower `update_summary()`/`update_badge()` rather than a full `render()`, so typing in a field doesn't lose focus. `render()` itself is only called on first load and after a JSON file is loaded (`load_json`), since those are the only times the whole DOM tree needs rebuilding from a new `state`.
```

Replace the **Export flows** bullet list's first bullet — find:

```markdown
- `download_json()` / `load_json(file)` serialize/deserialize `state` directly via `build_export_object`/`merge_loaded_state` — no DOM scraping.
```

with:

```markdown
- `download_json()` / `load_json(file)` (buttons live in the sidebar footer's "Export / import results" accordion) serialize/deserialize `state` directly via `build_export_object`/`merge_loaded_state` — no DOM scraping.
```

Replace the badge bullet — find:

```markdown
- `build_badge_svg_markup(state)` builds a self-contained `<svg>` — 3 concentric rings (Governance/Sustainability/Insurance), each ring split into one arc per principle via `ring_segment_offsets`, colored by that principle's status. `download_badge_svg()`/`download_badge_png()` export it as a static image the infrastructure can host on its own site; there is no live/dynamic embed since the tool has no backend.
```

with:

```markdown
- `build_badge_svg_markup(state)` builds a self-contained `<svg>` ("POSI Compliance" title, 3 concentric rings for Governance/Sustainability/Insurance each split into one arc per principle via `ring_segment_offsets`, the overall `%`, infrastructure name, and assessment date) — deliberately compact; the full per-section breakdown lives in `build_badge_hovercard_html(state)`, shown as a `.hovercard` on hover/focus of the badge (plain CSS, no new dependency). `download_badge_svg()`/`download_badge_png()` export the compact badge as a static image the infrastructure can host on its own site; there is no live/dynamic embed since the tool has no backend.
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "Update docs for FRAMEWORKS data model and sidebar/badge redesign"
```
