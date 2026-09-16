# POSI Self-Assessment Tool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current SPII values/principles self-assessment in `index.html` with a POSI (Principles of Open Scholarly Infrastructure v2.0) based self-assessment: name the infrastructure, classify it against an activities taxonomy, score all 20 POSI principles as compliant/making progress/not compliant with optional notes, and export the result as JSON (reloadable), PDF, and a downloadable SVG/PNG "doughnut" certification badge.

**Architecture:** Single standalone `index.html` file, no build step, no framework. POSI content and the activities taxonomy are defined once as JS data (`ACTIVITIES`, `POSI`), a single in-memory `state` object is the source of truth, and the whole page is rendered from that state via `render()`. Styling combines the vendored Oat CSS framework with hand-copied SURF Design System color/font tokens as CSS custom properties.

**Tech Stack:** Vanilla HTML/CSS/JS, Oat CSS (`oat.min.css`, vendored), jsPDF + html2canvas (CDN, pre-existing), no npm/build tooling.

**Spec:** `docs/superpowers/specs/2026-08-25-posi-self-assessment-design.md`

## Global Constraints

- Everything lives in one file, `index.html` — no build step, no package manager, no server, no JS framework.
- No new runtime dependency: SURF Design System token *values* are hand-copied as CSS custom properties, not imported from a package or CDN stylesheet.
- The activities taxonomy is descriptive metadata only — every infrastructure is scored against the same 20 POSI principles regardless of taxonomy selection.
- The "embeddable" badge is a static downloadable SVG/PNG image — there is no backend, so there is no live/dynamic embed.
- POSI principle text must be verbatim from `https://openscholarlyinfrastructure.org/` (fetched directly from the live page for this plan), each principle section linking back to its source for attribution.
- Maturity scale is exactly three values: `compliant`, `progress` (label: "Making progress"), `non-compliant` (label: "Not compliant"), plus `null` for unscored.
- No committed test suite (matches project convention, confirmed in the spec) — pure-logic verification in Task 1 uses a throwaway Node script run from the scratchpad, not committed to the repo.

---

### Task 1: POSI data model and pure logic functions

**Files:**
- Modify: `index.html` (`<head>`, before `</head>`)
- Test: none committed — throwaway Node script in the session scratchpad, deleted/ignored after use

**Interfaces:**
- Consumes: nothing (foundational).
- Produces (all defined inside `<script id="app-logic">`, no DOM access, order-independent globals shared with later `<script>` blocks in the same document):
  - `ACTIVITIES: Array<string | {group: string, items: string[]}>`
  - `POSI: Array<{section: string, sourceUrl: string, principles: Array<{id: string, title: string, text: string}>}>`
  - `initial_state(): {name: string, activities: string[], posi: {[id: string]: {status: 'compliant'|'progress'|'non-compliant'|null, notes: string}}}`
  - `compute_summary(state): {compliant: number, progress: number, nonCompliant: number, scoredCount: number, totalCount: number, percentCompliant: number}`
  - `build_export_object(state): {name: string, activities: string[], posi: {[id: string]: {status: string|null, notes: string}}}`
  - `merge_loaded_state(loaded: any): state` (same shape as `initial_state()`, defaulting/ignoring anything malformed)
  - `ring_segment_offsets(radius: number, count: number): Array<{dasharray: string, dashoffset: number}>`

- [ ] **Step 1: Write the throwaway verification script**

Create a file named `verify-logic.mjs` anywhere convenient outside the repo (e.g. your scratchpad directory, or `/tmp/verify-logic.mjs`) with:

```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

// Reads index.html relative to the current working directory — run this
// script from the repository root (see Step 2/5 run commands below).
const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
assert.ok(match, 'app-logic script block not found');
const context = vm.createContext({});
vm.runInContext(match[1], context);

// initial_state
const state = context.initial_state();
assert.equal(Object.keys(state.posi).length, 20, 'expected 20 POSI principles');
assert.deepEqual(state.activities, []);
assert.equal(state.name, '');

// POSI content shape
assert.equal(context.POSI.length, 3);
assert.equal(context.POSI[0].section, 'Governance');
assert.equal(context.POSI[0].principles.length, 7);
assert.equal(context.POSI[1].section, 'Sustainability');
assert.equal(context.POSI[1].principles.length, 8);
assert.equal(context.POSI[2].section, 'Insurance');
assert.equal(context.POSI[2].principles.length, 5);

// compute_summary
state.posi['coverage'].status = 'compliant';
state.posi['stakeholder-governed'].status = 'compliant';
state.posi['cannot-lobby'].status = 'progress';
state.posi['living-will'].status = 'non-compliant';
const summary = context.compute_summary(state);
assert.equal(summary.compliant, 2);
assert.equal(summary.progress, 1);
assert.equal(summary.nonCompliant, 1);
assert.equal(summary.scoredCount, 4);
assert.equal(summary.totalCount, 20);
assert.equal(summary.percentCompliant, 50);

// all-unscored edge case
const fresh = context.initial_state();
const freshSummary = context.compute_summary(fresh);
assert.equal(freshSummary.percentCompliant, 0);
assert.equal(freshSummary.scoredCount, 0);

// build_export_object
const exported = context.build_export_object(state);
assert.equal(exported.posi['coverage'].status, 'compliant');
assert.equal(exported.name, '');

// merge_loaded_state: valid partial input
const merged = context.merge_loaded_state({
  name: 'Test Repo',
  activities: ['Manage & Organise'],
  posi: {
    coverage: { status: 'compliant', notes: 'good' },
    unknown_id: { status: 'compliant' },
  },
});
assert.equal(merged.name, 'Test Repo');
assert.deepEqual(merged.activities, ['Manage & Organise']);
assert.equal(merged.posi['coverage'].status, 'compliant');
assert.equal(merged.posi['coverage'].notes, 'good');
assert.equal(merged.posi['stakeholder-governed'].status, null);
assert.equal(Object.keys(merged.posi).length, 20);
assert.equal('unknown_id' in merged.posi, false);

// merge_loaded_state: garbage input doesn't throw
const mergedBad = context.merge_loaded_state(null);
assert.equal(mergedBad.name, '');
assert.equal(Object.keys(mergedBad.posi).length, 20);
const mergedBad2 = context.merge_loaded_state({ posi: 'not an object', activities: 'not an array' });
assert.deepEqual(mergedBad2.activities, []);

// ring_segment_offsets
const offsets = context.ring_segment_offsets(10, 4);
assert.equal(offsets.length, 4);
const circumference = 2 * Math.PI * 10;
const segLen = circumference / 4;
assert.ok(Math.abs(offsets[0].dashoffset - 0) < 1e-9);
assert.ok(Math.abs(offsets[1].dashoffset - -segLen) < 1e-9);
assert.ok(Math.abs(offsets[2].dashoffset - -2 * segLen) < 1e-9);
assert.equal(offsets[0].dasharray, segLen + ' ' + (circumference - segLen));

console.log('All pure-logic checks passed.');
```

- [ ] **Step 2: Run it to verify it fails**

Run (from the repository root, so the relative `index.html` read in Step 1 resolves correctly): `node /path/to/verify-logic.mjs`
Expected: FAIL — `app-logic script block not found` (the assertion throws), since `index.html` has no `<script id="app-logic">` yet.

- [ ] **Step 3: Add `<meta charset="UTF-8">`**

In `index.html`, the `<head>` currently starts:

```html
<head>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
```

Add a charset declaration as the very first line of `<head>` (required because the POSI text below uses real Unicode punctuation — `–`, `—`, `’`, `“`, `”` — not HTML entities):

```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
```

- [ ] **Step 4: Add the `app-logic` script block**

Insert this new `<script id="app-logic">` block right before the closing `</head>` tag (after the existing `<script src="https://html2canvas.hertzen.com/dist/html2canvas.min.js"></script>` line, before the current inline `<script>` block — that inline block and everything in `<body>` will be replaced in Task 2, so exact placement relative to it doesn't matter as long as this new block is inside `<head>`):

```html
        <script id="app-logic">
            const ACTIVITIES = [
                "Manage & Organise",
                { group: "Research Lifecycle", items: ["Discover", "Hypothesise", "Plan", "Collect", "Process", "Analyse", "Report"] },
                "Document & Preserve",
                "Review & Curate",
                "Publish & Communicate",
                "Evaluate & Monitor",
            ];

            const POSI = [
                {
                    section: "Governance",
                    sourceUrl: "https://openscholarlyinfrastructure.org/#governance",
                    principles: [
                        { id: "coverage", title: "Coverage across the scholarly enterprise", text: "Research transcends disciplines, geography, institutions, and stakeholders. Organisations and the infrastructure they run need to reflect this." },
                        { id: "stakeholder-governed", title: "Stakeholder governed", text: "A board-governed organisation drawn from the stakeholder community builds confidence that the organisation will make decisions driven by community consensus and a balance of interests." },
                        { id: "non-discriminatory", title: "Non-discriminatory participation or membership", text: "We see the best option to be an “opt-in” approach with principles of non-discrimination and inclusivity, where any relevant group may express an interest and should be welcome. Representation in governance must reflect the character of the community or membership." },
                        { id: "transparent-governance", title: "Transparent governance", text: "To foster trust, the processes and policies for governing the organisation and selecting representatives to governance groups should be transparent (within the constraints of privacy laws)." },
                        { id: "cannot-lobby", title: "Cannot lobby", text: "Infrastructure organisations should not lobby for regulatory change to cement their own positions or narrow self-interest. However, an infrastructure organisation’s role is to support its community, and this can include advocating for policy changes." },
                        { id: "living-will", title: "Living will", text: "To build trust, organisations should establish and communicate clear commitments regarding their long-term stewardship responsibilities, including the principles by which assets, data, resources, services, and staff would be responsibly transferred to a successor or the organisation or service wound down. The commitments should address future governance, with defined criteria for acceptable successor organisations. This should include continued alignment with POSI and any legal or structural constraints." },
                        { id: "regular-review", title: "Regular review of purpose and community value", text: "Organisations and services should regularly review their relevance, effectiveness, and the level of community support to determine whether their continued operation is necessary. If no longer needed, they should take responsible steps to transition or wind down operations in consultation with the community and in alignment with their living will." },
                    ],
                },
                {
                    section: "Sustainability",
                    sourceUrl: "https://openscholarlyinfrastructure.org/#sustainability",
                    principles: [
                        { id: "transparent-operations", title: "Transparent operations", text: "To enable organisational accountability and openness, the operating policies and procedures, detailed financials, sustainability models, fees, strategic and product roadmaps, organisational charts, and other appropriate operational information should be made openly available (within the constraints of privacy laws). Information should be available for investigation and reuse by the community." },
                        { id: "time-limited-funds", title: "Time-limited funds are used only for time-limited activities", text: "Operations should be supported by sustainable revenue sources, whereas time-limited funds are used only for time-limited activities. Depending on grants to fund ongoing and/or long-term operations fully makes organisations fragile and distracts from maintaining core infrastructure." },
                        { id: "generate-surplus", title: "Goal to generate surplus", text: "It is not enough to merely survive; organisations and services have to be able to adapt and change. Organisations and services that define long-term sustainability based only on recovering costs risk becoming brittle and stagnant. To weather economic, social and technological volatility, organisations and services need financial resources beyond immediate operating costs." },
                        { id: "financial-reserves", title: "Establish and maintain financial reserves guided by policy", text: "Organisations and services should have a clear policy on maintaining financial reserves, including the purpose, minimum and maximum level, and governance of these funds. The actual level of reserves should be determined and periodically reviewed by the governing body, ensuring that resources are available to support Living Will implementation, including an orderly wind-down, transition to a successor, or response to major unforeseen events. A financial reserve policy might include how funds will be held, under what circumstances they will be used, and how much would be necessary for an adequate wind-down or transfer of assets, given the complexity of the organisation’s infrastructure." },
                        { id: "mission-consistent-revenue", title: "Mission-consistent revenue generation", text: "Revenue sources should be evaluated against the infrastructure’s mission and not run counter to the aims of the organisation or service." },
                        { id: "revenue-not-data", title: "Revenue generated from services, not data", text: "Data related to the running of the scholarly infrastructure should be community property. Appropriate revenue sources might include value-added services, consulting, API Service Level Agreements, or membership fees." },
                        { id: "volunteer-labour", title: "Volunteer labour", text: "Organisations that rely on volunteers and their labour should recognise this as a valuable resource for the organisation’s long-term viability, and factor it into sustainability planning and risk management." },
                        { id: "transition-planning", title: "Transition planning", text: "Organisations that are heavily dependent on a limited number of individuals should take steps to reduce their dependence on these individuals, including via transition and succession planning, so that the organisation is not at risk of collapse in the event of their departure." },
                    ],
                },
                {
                    section: "Insurance",
                    sourceUrl: "https://openscholarlyinfrastructure.org/#insurance",
                    principles: [
                        { id: "open-source", title: "Open source", text: "All software and non-physical assets required to run the infrastructure should be available under an open-source licence. This does not include other software that may be involved with running the organisation." },
                        { id: "open-secure-data", title: "Ensure open and secure data accessibility within legal and ethical constraints", text: "To support potential forking or replication, infrastructure should aim to make all relevant data openly available, following best practices such as applying a CC0 waiver where appropriate. This must be balanced with compliance with privacy, data protection, and security requirements. Organisations should have a clear policy outlining how private or sensitive data will be handled—particularly in the event of a transfer to another organisation—to ensure continuity, legal compliance, and responsible stewardship." },
                        { id: "available-preserved", title: "Available and preserved", text: "It is not enough that content, data, and software be “open” if there is no practical way to obtain them. These resources should be made easily available with clear public documentation about where they are and how to access them, as well as an open licence where possible. It is not enough that “open” resources are available. In line with the Living Will, it is essential to deposit content, data, and software with at least one trusted third-party digital archive." },
                        { id: "patent-non-assertion", title: "Patent non-assertion", text: "The organisation should commit to a patent non-assertion policy or covenant. The organisation may obtain patents to protect its own operations, but not use them to prevent the community from replicating the infrastructure." },
                        { id: "interoperability-standards", title: "Prioritise interoperability and open standards to ensure continuity and resilience", text: "Infrastructures should adopt and support widely accepted open standards—both formal and de facto—to ensure that systems, data, and services can be replicated, migrated, or integrated with minimal disruption without the use of proprietary extensions or software. Where relevant, organisations should document dependencies on standards." },
                    ],
                },
            ];

            function initial_state() {
                const posi = {};
                POSI.forEach((section) => {
                    section.principles.forEach((principle) => {
                        posi[principle.id] = { status: null, notes: "" };
                    });
                });
                return { name: "", activities: [], posi };
            }

            function compute_summary(state) {
                const entries = Object.values(state.posi);
                const scored = entries.filter((e) => e.status !== null);
                const compliant = scored.filter((e) => e.status === "compliant").length;
                const progress = scored.filter((e) => e.status === "progress").length;
                const nonCompliant = scored.filter((e) => e.status === "non-compliant").length;
                const percentCompliant = scored.length ? Math.round((compliant / scored.length) * 100) : 0;
                return {
                    compliant,
                    progress,
                    nonCompliant,
                    scoredCount: scored.length,
                    totalCount: entries.length,
                    percentCompliant,
                };
            }

            function build_export_object(state) {
                const posi = {};
                Object.keys(state.posi).forEach((id) => {
                    posi[id] = { status: state.posi[id].status, notes: state.posi[id].notes };
                });
                return { name: state.name, activities: state.activities.slice(), posi };
            }

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

            function ring_segment_offsets(radius, count) {
                const circumference = 2 * Math.PI * radius;
                const segmentLength = circumference / count;
                const gap = circumference - segmentLength;
                const offsets = [];
                for (let i = 0; i < count; i++) {
                    offsets.push({
                        dasharray: segmentLength + " " + gap,
                        dashoffset: -(i * segmentLength),
                    });
                }
                return offsets;
            }
        </script>
```

- [ ] **Step 5: Run the verification script again to confirm it passes**

Run (from the repository root): `node /path/to/verify-logic.mjs`
Expected: PASS — prints `All pure-logic checks passed.`

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add POSI data model and pure scoring/serialization logic"
```

(Do not commit the scratchpad verification script — it lives outside the repo.)

---

### Task 2: Core rendering — name, classify, POSI sections, summary

**Files:**
- Modify: `index.html` (`<style>` block, entire `<body>`)

**Interfaces:**
- Consumes: `ACTIVITIES`, `POSI`, `initial_state()`, `compute_summary(state)` from Task 1.
- Produces:
  - Global `let state` (initialized via `initial_state()`)
  - `render(): void` — rebuilds `#app` from `state`
  - `render_header(): HTMLElement`
  - `render_summary_section(): HTMLElement`
  - `render_classify_section(): HTMLElement`
  - `render_activity_checkbox(value: string, labelText: string): HTMLElement`
  - `render_posi_section(section): HTMLElement`
  - `render_principle(principle): HTMLElement`
  - `wire_up_events(app: HTMLElement): void`
  - `toggle_activity(label: string, checked: boolean): void`
  - `set_status(principleId: string, status: string): void`
  - `update_summary(): void`
  - DOM contract later tasks rely on: `#app` (root mount), `#infra-name` (name input), `#classify-section`, `#summary-heading`, `#summary-counts`, elements with `[data-principle-id]` wrapping `.status-card[data-status]` and `textarea.notes`.

This task removes the old SPII values/principles markup and scoring script entirely. There is no export functionality yet (JSON/PDF/badge land in Tasks 3–5) — this task's deliverable is the assessment UI itself: name it, classify it, score every POSI principle, see the summary update live.

- [ ] **Step 1: Replace the `<style>` block**

Replace the entire existing `<style>...</style>` block in `<head>` with:

```html
        <style>
            :root {
                --primary: rgba(6,75,203,1);
                --warning: rgba(202,138,4,1);
                --destructive: rgba(185,28,28,1);
                --compliant: rgba(13,148,136,1);
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

            body {
                margin: 20pt;
                font-family: var(--font-sans);
            }

            h2 {
                font-size: x-large;
                margin-left: 20pt;
                margin-bottom: 8pt;
            }

            h3 {
                font-size: medium;
                margin-bottom: 0pt;
            }

            section > p {
                margin-left: 24pt;
                margin-bottom: 6pt;
            }

            div.row {
                padding-right: 24pt;
            }

            section {
                margin-top: 24pt;
            }

            summary {
                padding: 10pt;
            }

            .status-cards {
                display: flex;
                gap: 8pt;
                padding-right: 24pt;
            }

            .status-card {
                cursor: pointer;
                border: 1pt solid transparent;
            }

            .status-card.selected[data-status="compliant"] { border-color: var(--compliant); }
            .status-card.selected[data-status="progress"] { border-color: var(--warning); }
            .status-card.selected[data-status="non-compliant"] { border-color: var(--destructive); }

            textarea.notes {
                width: 100%;
                min-height: 40pt;
                margin-top: 6pt;
                font-family: var(--font-sans);
            }

            #summary-section {
                display: flex;
                align-items: baseline;
                gap: 24pt;
                margin-left: 20pt;
            }

            #summary-heading {
                font-size: large;
                color: var(--primary);
            }

            .taxonomy-group {
                margin-left: 16pt;
            }

            #watermark {
                position: fixed;
                bottom: 50%;
                left: 20%;
                opacity: 0.3;
                z-index: 99;
                color: black;
                font-size: 36pt;
                font-weight: bold;
                transform: rotate(-45deg);
            }

            @media print {
                #actions {
                    display: none;
                }
                .status-cards {
                    display: none;
                }
            }
        </style>
```

- [ ] **Step 2: Replace `<body>`**

Replace everything from `<body>` to `</body>` (i.e. the watermark span, old `<main id="main">...</main>`, the two old buttons, and the old bottom `<script>` block) with:

```html
    <body>

        <span id="watermark">DRAFT – WORK IN PROGRESS</span>

        <main id="app"></main>

        <script id="app-render">
            let state = initial_state();

            function render() {
                const app = document.getElementById("app");
                app.innerHTML = "";
                app.appendChild(render_header());
                app.appendChild(render_summary_section());
                app.appendChild(render_classify_section());
                POSI.forEach((section) => app.appendChild(render_posi_section(section)));
                wire_up_events(app);
                update_summary();
            }

            function render_header() {
                const wrap = document.createElement("div");
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

            function render_activity_checkbox(value, labelText) {
                const label = document.createElement("label");
                label.style.display = "block";
                const input = document.createElement("input");
                input.type = "checkbox";
                input.value = value;
                input.checked = state.activities.includes(value);
                label.appendChild(input);
                label.appendChild(document.createTextNode(" " + labelText));
                return label;
            }

            function render_posi_section(section) {
                const el = document.createElement("section");
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
                [
                    { status: "compliant", label: "Compliant" },
                    { status: "progress", label: "Making progress" },
                    { status: "non-compliant", label: "Not compliant" },
                ].forEach((option) => {
                    const card = document.createElement("article");
                    card.className = "card col-3 status-card";
                    card.dataset.status = option.status;
                    if (state.posi[principle.id].status === option.status) {
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
                notes.value = state.posi[principle.id].notes;
                details.appendChild(notes);

                return details;
            }

            function wire_up_events(app) {
                const nameInput = document.getElementById("infra-name");
                nameInput.addEventListener("input", (e) => {
                    state.name = e.target.value;
                });

                document.getElementById("classify-section").addEventListener("change", (e) => {
                    if (e.target.matches('input[type="checkbox"]')) {
                        toggle_activity(e.target.value, e.target.checked);
                    }
                });

                app.addEventListener("click", (e) => {
                    const card = e.target.closest(".status-card");
                    if (!card) return;
                    const details = card.closest("[data-principle-id]");
                    set_status(details.dataset.principleId, card.dataset.status);
                });

                app.addEventListener("input", (e) => {
                    if (e.target.matches("textarea.notes")) {
                        const details = e.target.closest("[data-principle-id]");
                        state.posi[details.dataset.principleId].notes = e.target.value;
                    }
                });
            }

            function toggle_activity(label, checked) {
                if (checked) {
                    if (!state.activities.includes(label)) state.activities.push(label);
                } else {
                    state.activities = state.activities.filter((a) => a !== label);
                }
            }

            function set_status(principleId, status) {
                state.posi[principleId].status = state.posi[principleId].status === status ? null : status;
                const details = document.querySelector('[data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    card.classList.toggle("selected", card.dataset.status === state.posi[principleId].status);
                });
                update_summary();
            }

            function update_summary() {
                const summary = compute_summary(state);
                document.getElementById("summary-heading").textContent = "Overall: " + summary.percentCompliant + "% compliant";
                document.getElementById("summary-counts").textContent =
                    summary.compliant + " compliant · " + summary.progress + " making progress · " +
                    summary.nonCompliant + " not compliant (" + summary.scoredCount + "/" + summary.totalCount + " scored)";
            }

            render();
        </script>
    </body>
```

- [ ] **Step 3: Manually verify in a browser**

Open `index.html` directly (`file://`) and confirm:
- No console errors.
- Typing a name updates the `<h1>` input.
- Checking "Manage & Organise" and a couple of Research Lifecycle sub-items works (nested checkboxes render and toggle independently).
- Expanding a POSI `<details>` shows its verbatim text and a link to the POSI source.
- Clicking a status card (e.g. "Compliant") highlights it with a colored border and un-highlights the others in that principle; clicking the same card again deselects it (back to unscored).
- Typing into a notes `<textarea>` doesn't lose focus or reset while typing.
- The summary line at the top updates its `%` and counts immediately after each status card click, and only counts principles that have been scored.
- All three POSI sections (Governance — 7, Sustainability — 8, Insurance — 5) are present and correctly labelled.
- Toggle OS/browser dark mode (or devtools' rendering emulation for `prefers-color-scheme`) and confirm the selected status card's border color switches to the dark-mode token value (e.g. a selected "Not compliant" card's red border becomes the lighter dark-mode red).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Replace SPII assessment UI with data-driven POSI rendering"
```

---

### Task 3: JSON export and reload

**Files:**
- Modify: `index.html` (inside `<script id="app-render">`)

**Interfaces:**
- Consumes: `state` (global), `build_export_object(state)`, `merge_loaded_state(loaded)` from Task 1, `render()` from Task 2.
- Produces: `render_actions_section(): HTMLElement`, `download_json(): void`, `load_json(file: File): void`. Introduces `#actions` (mount point later tasks append buttons into) and `#load-json-input`.

- [ ] **Step 1: Add the actions section, JSON download, and JSON reload functions**

Add these three functions inside `<script id="app-render">` (anywhere after `update_summary()` and before the trailing `render();` call is fine — e.g. directly after `update_summary()`):

```js
            function render_actions_section() {
                const section = document.createElement("section");
                section.id = "actions";

                const downloadJsonBtn = document.createElement("button");
                downloadJsonBtn.textContent = "Download results as JSON";
                downloadJsonBtn.addEventListener("click", download_json);
                section.appendChild(downloadJsonBtn);

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
                section.appendChild(loadJsonLabel);

                return section;
            }

            function download_json() {
                const data = build_export_object(state);
                const json = JSON.stringify(data, null, 2);
                const blob = new Blob([json], { type: "application/json" });
                const link = document.createElement("a");
                link.href = URL.createObjectURL(blob);
                link.download = "results.json";
                link.style.display = "none";
                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
            }

            function load_json(file) {
                const reader = new FileReader();
                reader.onload = () => {
                    let parsed;
                    try {
                        parsed = JSON.parse(reader.result);
                    } catch (err) {
                        alert("That file is not valid JSON.");
                        return;
                    }
                    state = merge_loaded_state(parsed);
                    render();
                };
                reader.readAsText(file);
            }
```

- [ ] **Step 2: Wire the actions section into `render()`**

Replace this line inside `render()`:

```js
                POSI.forEach((section) => app.appendChild(render_posi_section(section)));
                wire_up_events(app);
```

with:

```js
                POSI.forEach((section) => app.appendChild(render_posi_section(section)));
                app.appendChild(render_actions_section());
                wire_up_events(app);
```

- [ ] **Step 3: Manually verify the round trip in a browser**

Open `index.html`, enter a name, select a few activities, score a mix of principles across all 3 sections with some notes text, leave others unscored. Then:
- Click "Download results as JSON", open the downloaded file, confirm it contains `name`, `activities`, and a `posi` object with all 20 principle ids, each with the correct `status`/`notes`.
- Reload the page (fresh, empty state), use "Load JSON" to select the file just downloaded.
- Confirm the name field, activity checkboxes, every principle's selected status card, and every notes textarea are restored exactly, and the summary line recomputes to match.
- Try loading a JSON file with `{}` as content — confirm it doesn't crash and resets to a valid empty state.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add JSON export and reload"
```

---

### Task 4: PDF export

**Files:**
- Modify: `index.html` (inside `<script id="app-render">`)

**Interfaces:**
- Consumes: `render_actions_section()` from Task 3 (extended here), the existing `jspdf` global from the CDN script already loaded in `<head>`.
- Produces: `download_pdf(): void`. Adds a `beforeprint` listener.

- [ ] **Step 1: Add a "Download report" button to the actions section**

In `render_actions_section()` (Task 3), add a new button after the `loadJsonLabel` append and before `return section;`:

```js
                const downloadPdfBtn = document.createElement("button");
                downloadPdfBtn.textContent = "Download report";
                downloadPdfBtn.addEventListener("click", download_pdf);
                section.appendChild(downloadPdfBtn);
```

So the full function now reads:

```js
            function render_actions_section() {
                const section = document.createElement("section");
                section.id = "actions";

                const downloadJsonBtn = document.createElement("button");
                downloadJsonBtn.textContent = "Download results as JSON";
                downloadJsonBtn.addEventListener("click", download_json);
                section.appendChild(downloadJsonBtn);

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
                section.appendChild(loadJsonLabel);

                const downloadPdfBtn = document.createElement("button");
                downloadPdfBtn.textContent = "Download report";
                downloadPdfBtn.addEventListener("click", download_pdf);
                section.appendChild(downloadPdfBtn);

                return section;
            }
```

- [ ] **Step 2: Add `download_pdf()` and the print-collapse listener**

Add near the end of `<script id="app-render">`, after `load_json()` and before the trailing `render();` call:

```js
            function download_pdf() {
                const doc = new jspdf.jsPDF();
                doc.html(document.body, {
                    callback: (doc) => doc.save("report.pdf"),
                    x: 10,
                    width: 200,
                    windowWidth: 1000,
                });
            }

            window.addEventListener("beforeprint", () => {
                document.querySelectorAll("details").forEach((el) => el.removeAttribute("open"));
            });
```

- [ ] **Step 3: Manually verify in a browser**

Enter a name, score several principles with notes, click "Download report". Confirm `report.pdf` downloads, opens, and shows the entered data. Confirm the buttons/file input from the actions section do not appear in the PDF (hidden via the `@media print` rule from Task 2). Use the browser's print preview (Ctrl/Cmd+P) to confirm all `<details>` collapse to just their summaries.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add PDF export"
```

---

### Task 5: Doughnut certification badge

**Files:**
- Modify: `index.html` (inside `<script id="app-render">`)

**Interfaces:**
- Consumes: `state`, `POSI`, `compute_summary(state)`, `ring_segment_offsets(radius, count)` from Task 1; `render()`, `set_status()`, the name `<input>` handler from Task 2; `render_actions_section()` insertion point from Task 3.
- Produces: `RING_CONFIG`, `STATUS_COLOR_VAR`, `status_color(status)`, `escape_xml(str)`, `build_badge_svg_markup(state)`, `update_badge()`, `render_badge_section()`, `trigger_download(href, filename)`, `download_badge_svg()`, `download_badge_png()`.

- [ ] **Step 1: Add the badge-building and download functions**

Add these near the end of `<script id="app-render">`, after `download_pdf()`/the `beforeprint` listener and before the trailing `render();` call:

```js
            const RING_CONFIG = [
                { section: "Governance", radius: 34 },
                { section: "Sustainability", radius: 48 },
                { section: "Insurance", radius: 62 },
            ];

            const STATUS_COLOR_VAR = {
                compliant: "--compliant",
                progress: "--warning",
                "non-compliant": "--destructive",
            };

            function status_color(status) {
                const varName = STATUS_COLOR_VAR[status];
                if (!varName) return "#cccccc";
                const value = getComputedStyle(document.documentElement).getPropertyValue(varName).trim();
                return value || "#cccccc";
            }

            function escape_xml(str) {
                return str
                    .replace(/&/g, "&amp;")
                    .replace(/</g, "&lt;")
                    .replace(/>/g, "&gt;")
                    .replace(/"/g, "&quot;");
            }

            function build_badge_svg_markup(state) {
                const size = 200;
                const center = size / 2;
                const strokeWidth = 10;
                let circles = "";
                RING_CONFIG.forEach((ring) => {
                    const section = POSI.find((s) => s.section === ring.section);
                    const offsets = ring_segment_offsets(ring.radius, section.principles.length);
                    section.principles.forEach((principle, i) => {
                        const status = state.posi[principle.id].status;
                        const color = status_color(status);
                        const off = offsets[i];
                        circles +=
                            '<circle cx="' + center + '" cy="' + center + '" r="' + ring.radius +
                            '" fill="none" stroke="' + color + '" stroke-width="' + strokeWidth +
                            '" stroke-dasharray="' + off.dasharray + '" stroke-dashoffset="' + off.dashoffset +
                            '" transform="rotate(-90 ' + center + " " + center + ')" />';
                    });
                });
                const summary = compute_summary(state);
                const name = escape_xml(state.name || "Unnamed infrastructure");
                return (
                    '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ' + size + " " + size + '" width="' + size + '" height="' + size + '">' +
                    '<rect width="' + size + '" height="' + size + '" fill="white" />' +
                    circles +
                    '<text x="' + center + '" y="' + (center - 4) + '" text-anchor="middle" font-size="22" font-family="Source Sans 3, sans-serif" font-weight="bold">' + summary.percentCompliant + "%</text>" +
                    '<text x="' + center + '" y="' + (center + 14) + '" text-anchor="middle" font-size="8" font-family="Source Sans 3, sans-serif">' + name + "</text>" +
                    "</svg>"
                );
            }

            function update_badge() {
                const container = document.getElementById("badge-preview");
                if (!container) return;
                container.innerHTML = build_badge_svg_markup(state);
            }

            function render_badge_section() {
                const section = document.createElement("section");
                section.id = "badge-section";
                const h2 = document.createElement("h2");
                h2.textContent = "Certification badge";
                section.appendChild(h2);

                const preview = document.createElement("div");
                preview.id = "badge-preview";
                section.appendChild(preview);

                const svgBtn = document.createElement("button");
                svgBtn.textContent = "Download badge (SVG)";
                svgBtn.addEventListener("click", download_badge_svg);
                section.appendChild(svgBtn);

                const pngBtn = document.createElement("button");
                pngBtn.textContent = "Download badge (PNG)";
                pngBtn.addEventListener("click", download_badge_png);
                section.appendChild(pngBtn);

                return section;
            }

            function trigger_download(href, filename) {
                const link = document.createElement("a");
                link.href = href;
                link.download = filename;
                link.style.display = "none";
                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
            }

            function download_badge_svg() {
                const markup = build_badge_svg_markup(state);
                const blob = new Blob([markup], { type: "image/svg+xml" });
                trigger_download(URL.createObjectURL(blob), "badge.svg");
            }

            function download_badge_png() {
                const markup = build_badge_svg_markup(state);
                const svgBlob = new Blob([markup], { type: "image/svg+xml" });
                const url = URL.createObjectURL(svgBlob);
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement("canvas");
                    canvas.width = 200;
                    canvas.height = 200;
                    const ctx = canvas.getContext("2d");
                    ctx.drawImage(img, 0, 0);
                    URL.revokeObjectURL(url);
                    canvas.toBlob((blob) => {
                        trigger_download(URL.createObjectURL(blob), "badge.png");
                    });
                };
                img.src = url;
            }
```

- [ ] **Step 2: Mount the badge section and refresh it on every change**

Replace this line inside `render()` (added in Task 3):

```js
                app.appendChild(render_actions_section());
```

with:

```js
                app.appendChild(render_badge_section());
                app.appendChild(render_actions_section());
```

Replace the end of `render()`:

```js
                wire_up_events(app);
                update_summary();
            }
```

with:

```js
                wire_up_events(app);
                update_summary();
                update_badge();
            }
```

Replace the name input handler inside `wire_up_events()`:

```js
                nameInput.addEventListener("input", (e) => {
                    state.name = e.target.value;
                });
```

with:

```js
                nameInput.addEventListener("input", (e) => {
                    state.name = e.target.value;
                    update_badge();
                });
```

Replace `set_status()`:

```js
            function set_status(principleId, status) {
                state.posi[principleId].status = state.posi[principleId].status === status ? null : status;
                const details = document.querySelector('[data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    card.classList.toggle("selected", card.dataset.status === state.posi[principleId].status);
                });
                update_summary();
            }
```

with:

```js
            function set_status(principleId, status) {
                state.posi[principleId].status = state.posi[principleId].status === status ? null : status;
                const details = document.querySelector('[data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    card.classList.toggle("selected", card.dataset.status === state.posi[principleId].status);
                });
                update_summary();
                update_badge();
            }
```

- [ ] **Step 3: Manually verify in a browser**

Score a spread of principles across all 3 sections (some compliant, some progress, some not compliant, some left unscored) and enter a name. Confirm:
- The on-page badge preview shows 3 concentric rings (7 segments inner, 8 middle, 5 outer), colored teal/amber/red per status and grey for unscored, with the center `%` matching the summary line and the name shown underneath.
- The preview updates immediately after each status card click and after editing the name.
- "Download badge (SVG)" produces a `.svg` file that opens correctly in a new browser tab/image viewer on its own (no missing references).
- "Download badge (PNG)" produces a `.png` file that opens correctly and visually matches the SVG.
- Toggle OS/browser dark mode (or devtools' rendering emulation) and confirm the badge ring colors switch to the dark-mode token values.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add doughnut certification badge with SVG/PNG export"
```

---

### Task 6: Update project documentation

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`

**Interfaces:** none (documentation only).

- [ ] **Step 1: Update `README.md`**

Current content:

```markdown
# Open Science infrastucture maturity assessment

This repository provides a simple tool to perform a self-assessment for an Open Science infrastructure.

It uses a standalone HTML file with styling provided by [Oat](https://oat.ink).
```

Replace with:

```markdown
# Open Science infrastructure maturity assessment

This repository provides a simple tool to perform a self-assessment for an Open Science infrastructure against the [Principles of Open Scholarly Infrastructure (POSI) v2.0](https://openscholarlyinfrastructure.org/).

It uses a standalone HTML file with styling provided by [Oat](https://oat.ink) and color/font tokens from the [SURF Design System](https://surfnet.github.io/DesignSystem/).
```

- [ ] **Step 2: Update `CLAUDE.md`'s Project overview**

The `## Project overview` paragraph still describes the old SPII-values framing. Replace:

```markdown
A self-assessment tool for the maturity of Open Science infrastructures, based on the values and principles collated in the SPII project. It is a single standalone `index.html` file — there is no build step, package manager, server, or test suite. Styling comes from the [Oat](https://oat.ink) CSS framework, vendored locally as `oat.min.css`.
```

with:

```markdown
A self-assessment tool for Open Science infrastructures against the [Principles of Open Scholarly Infrastructure (POSI) v2.0](https://openscholarlyinfrastructure.org/). It is a single standalone `index.html` file — there is no build step, package manager, server, or committed test suite. Styling comes from the [Oat](https://oat.ink) CSS framework (vendored locally as `oat.min.css`) layered with color/font tokens hand-copied from the [SURF Design System](https://surfnet.github.io/DesignSystem/).
```

- [ ] **Step 3: Update `CLAUDE.md`'s Architecture section**

Replace the `## Architecture` section (from `## Architecture` up to but not including `## Notes`) with:

```markdown
## Architecture

Everything lives in `index.html`: markup, inline `<style>`, and inline `<script>`. There is no JS framework — all rendering and interactivity is vanilla DOM manipulation, driven by a single in-memory `state` object.

**Data model** (`<script id="app-logic">` in `<head>`): `ACTIVITIES` is the "supports the following activities" taxonomy (flat labels plus one grouped entry for the Research Lifecycle sub-stages). `POSI` is the full Principles of Open Scholarly Infrastructure v2.0 content (3 sections — Governance, Sustainability, Insurance — 20 principles total), each principle with a stable `id`, `title`, and verbatim `text` transcribed from openscholarlyinfrastructure.org. `initial_state()` builds a fresh `state` object: `{name, activities, posi: {[id]: {status, notes}}}`, where `status` is `"compliant"`, `"progress"`, `"non-compliant"`, or `null`. `compute_summary(state)`, `build_export_object(state)`, `merge_loaded_state(loaded)`, and `ring_segment_offsets(radius, count)` are pure functions with no DOM access — this is what makes them testable outside a browser (see Testing below).

**Rendering** (`<script id="app-render">` in `<body>`): `render()` rebuilds `#app` entirely from the global `state` — name input, activity checkboxes, one `<details data-principle-id="...">` per POSI principle (3 status cards + a notes `<textarea>`), the live summary, the doughnut badge preview, and the actions row. Interaction handlers (`set_status`, `toggle_activity`, the notes/name input listeners) mutate `state` directly and call the narrower `update_summary()`/`update_badge()` rather than a full `render()`, so typing in a textarea doesn't lose focus. `render()` itself is only called on first load and after a JSON file is loaded (`load_json`), since those are the only times the whole DOM tree needs rebuilding from a new `state`.

**When adding a new POSI principle or activity**: add it to the `POSI`/`ACTIVITIES` data in `<script id="app-logic">` — the UI, scoring, JSON export/reload, and badge all render from that data automatically. Don't hand-edit the generated `<details>` markup; it's rebuilt by `render()` on every load.

**Export flows**:
- `download_json()` / `load_json(file)` serialize/deserialize `state` directly via `build_export_object`/`merge_loaded_state` — no DOM scraping.
- `download_pdf()` renders `document.body` via `jsPDF` (loaded from a CDN, `doc.html()`) into `report.pdf`. `@media print` rules hide the actions row and status cards, and the `beforeprint` listener collapses all `<details>` so the PDF/print view only shows scored summaries.
- `build_badge_svg_markup(state)` builds a self-contained `<svg>` — 3 concentric rings (Governance/Sustainability/Insurance), each ring split into one arc per principle via `ring_segment_offsets`, colored by that principle's status. `download_badge_svg()`/`download_badge_png()` export it as a static image the infrastructure can host on its own site; there is no live/dynamic embed since the tool has no backend.

**Styling**: color and font tokens are hand-copied from the [SURF Design System](https://surfnet.github.io/DesignSystem/) as CSS custom properties (`--primary`, `--warning`, `--destructive`, `--compliant`, `--font-sans`) in `:root`, with dark-mode overrides under `@media (prefers-color-scheme: dark)`. This is a copy of token *values*, not a dependency on SURF's packages — the page has no new runtime dependency beyond the pre-existing jsPDF/html2canvas CDN scripts.
```

- [ ] **Step 4: Add a Testing section to `CLAUDE.md`**

`CLAUDE.md`'s top `## Instructions` block is generic boilerplate and doesn't need edits. Instead, add a short new subsection right after the Architecture section added in Step 3 (before `## Notes`):

```markdown
## Testing

There is no committed test suite — this matches the project's "no build step" philosophy. The pure functions in `<script id="app-logic">` (`compute_summary`, `build_export_object`, `merge_loaded_state`, `ring_segment_offsets`) have no DOM dependency and can be exercised from Node by extracting that script block's text (regex on `<script id="app-logic">...</script>`) and running it in a `vm.createContext`, if you need to verify logic changes without a browser. Everything else (rendering, interactivity, PDF, badge image export) is verified manually by opening `index.html` in a browser.
```

- [ ] **Step 5: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "Update docs for POSI-based assessment architecture"
```
