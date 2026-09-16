# Multi-Framework Machinery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generalize `index.html`'s rendering machinery (per-principle state access, framework summaries, the assessment badge, and the "draft" label) so it genuinely supports more than one assessment framework, without changing POSI's current appearance or behavior at all.

**Architecture:** All changes stay inside `index.html`'s existing `<script id="app-logic">`/`<script id="app-render">`/`<style>` structure. Framework identity (`frameworkId`) gets threaded through the functions that currently hardcode `"posi"`/`state.frameworks.posi`, a new `activeFrameworkId` module variable drives the badge, and two DOM-id/id-generation schemes (summary section, badge ring radii) move from hardcoded to computed-from-`FRAMEWORKS`.

**Tech Stack:** Vanilla HTML/CSS/JS, Oat CSS (`oat.min.css`, vendored), no new dependency.

**Spec:** `docs/superpowers/specs/2026-08-26-multi-framework-machinery-design.md`

## Global Constraints

- Everything lives in one file, `index.html` — no build step, no package manager, no server, no JS framework, no new runtime dependency.
- With only POSI in `FRAMEWORKS` (the state throughout this entire plan), the page's appearance and behavior must be **identical** to before this plan — every task's manual verification must explicitly confirm this "no visible change for POSI" property.
- `activeFrameworkId` is NOT persisted in `build_export_object`/`merge_loaded_state` — it's view state, not assessment data.
- **After every task's commit, push the current branch to `origin`** (`git push -u origin <branch-name>` the first time, `git push` thereafter) so progress is visible on GitHub as each task lands — not only at the final merge.

---

### Task 1: Thread `frameworkId` through per-principle state access

**Files:**
- Modify: `index.html` (`<script id="app-render">`)

**Interfaces:**
- Consumes: `state.frameworks[frameworkId][principleId]` shape (already correct from a prior plan — this task fixes the *code that reads/writes it*, not the shape itself).
- Produces: `render_framework_content_section(section, frameworkId)`, `render_principle(principle, frameworkId)`, `set_status(frameworkId, principleId, status)` — all gain a `frameworkId` parameter. `details.dataset.frameworkId` is a new data attribute on each principle's `<details>`, alongside the existing `details.dataset.principleId`.

This is a correctness fix: today, `render_principle`, `set_status`, and the notes-textarea input listener all hardcode `state.frameworks.posi[...]`, regardless of which framework's section they render under. A second framework's principles would crash or silently corrupt POSI's state without this fix.

- [ ] **Step 1: Update `render_framework_content_section` to accept and pass through `frameworkId`**

Replace:
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
with:
```js
            function render_framework_content_section(section, frameworkId) {
                const el = document.createElement("section");
                el.id = posi_section_id(section);
                const h3 = document.createElement("h3");
                h3.textContent = section.section;
                el.appendChild(h3);
                const sourceNote = document.createElement("p");
                sourceNote.innerHTML = 'Source: <a href="' + section.sourceUrl + '">' + section.sourceUrl + "</a>";
                el.appendChild(sourceNote);
                section.principles.forEach((principle) => {
                    el.appendChild(render_principle(principle, frameworkId));
                });
                return el;
            }
```

- [ ] **Step 2: Update `render_framework_section`'s call site**

Replace:
```js
                framework.sections.forEach((frameworkSection) => {
                    section.appendChild(render_framework_content_section(frameworkSection));
                });
```
with:
```js
                framework.sections.forEach((frameworkSection) => {
                    section.appendChild(render_framework_content_section(frameworkSection, framework.id));
                });
```

- [ ] **Step 3: Update `render_principle` to accept `frameworkId` and use it everywhere it reads/writes state**

Replace the entire function:
```js
            function render_principle(principle) {
                const details = document.createElement("details");
                details.className = "container";
                details.dataset.principleId = principle.id;

                const summary = document.createElement("summary");
                const h4 = document.createElement("h4");
                h4.textContent = principle.title;
                summary.appendChild(h4);
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
                    const icon = document.createElement("span");
                    icon.className = "status-icon";
                    icon.innerHTML = ICONS[option.status];
                    card.appendChild(icon);

                    const label = document.createElement("p");
                    label.className = "status-label";
                    label.textContent = option.label;
                    card.appendChild(label);
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
with:
```js
            function render_principle(principle, frameworkId) {
                const details = document.createElement("details");
                details.className = "container";
                details.dataset.principleId = principle.id;
                details.dataset.frameworkId = frameworkId;

                const summary = document.createElement("summary");
                const h4 = document.createElement("h4");
                h4.textContent = principle.title;
                summary.appendChild(h4);
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
                    card.setAttribute("aria-pressed", state.frameworks[frameworkId][principle.id].status === option.status ? "true" : "false");
                    if (state.frameworks[frameworkId][principle.id].status === option.status) {
                        card.classList.add("selected");
                    }
                    const icon = document.createElement("span");
                    icon.className = "status-icon";
                    icon.innerHTML = ICONS[option.status];
                    card.appendChild(icon);

                    const label = document.createElement("p");
                    label.className = "status-label";
                    label.textContent = option.label;
                    card.appendChild(label);
                    cards.appendChild(card);
                });
                details.appendChild(cards);

                const notes = document.createElement("textarea");
                notes.className = "notes";
                notes.placeholder = "Explain your reasoning (optional)";
                notes.value = state.frameworks[frameworkId][principle.id].notes;
                details.appendChild(notes);

                return details;
            }
```

- [ ] **Step 4: Update `set_status` to accept and use `frameworkId`**

Replace:
```js
            function set_status(principleId, status) {
                state.frameworks.posi[principleId].status = state.frameworks.posi[principleId].status === status ? null : status;
                const details = document.querySelector('[data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    const isSelected = card.dataset.status === state.frameworks.posi[principleId].status;
                    card.classList.toggle("selected", isSelected);
                    card.setAttribute("aria-pressed", isSelected ? "true" : "false");
                });
                update_summary();
                update_badge();
            }
```
with:
```js
            function set_status(frameworkId, principleId, status) {
                state.frameworks[frameworkId][principleId].status = state.frameworks[frameworkId][principleId].status === status ? null : status;
                const details = document.querySelector('[data-framework-id="' + frameworkId + '"][data-principle-id="' + principleId + '"]');
                details.querySelectorAll(".status-card").forEach((card) => {
                    const isSelected = card.dataset.status === state.frameworks[frameworkId][principleId].status;
                    card.classList.toggle("selected", isSelected);
                    card.setAttribute("aria-pressed", isSelected ? "true" : "false");
                });
                update_summary();
                update_badge();
            }
```

- [ ] **Step 5: Update the three `appRoot` listeners' call sites**

Replace:
```js
            const appRoot = document.getElementById("app");
            appRoot.addEventListener("click", (e) => {
                const card = e.target.closest(".status-card");
                if (!card) return;
                const details = card.closest("[data-principle-id]");
                set_status(details.dataset.principleId, card.dataset.status);
            });
            appRoot.addEventListener("input", (e) => {
                if (e.target.matches("textarea.notes")) {
                    const details = e.target.closest("[data-principle-id]");
                    state.frameworks.posi[details.dataset.principleId].notes = e.target.value;
                }
            });
            appRoot.addEventListener("keydown", (e) => {
                if (e.key !== "Enter" && e.key !== " ") return;
                const card = e.target.closest(".status-card");
                if (!card) return;
                e.preventDefault();
                const details = card.closest("[data-principle-id]");
                set_status(details.dataset.principleId, card.dataset.status);
            });
```
with:
```js
            const appRoot = document.getElementById("app");
            appRoot.addEventListener("click", (e) => {
                const card = e.target.closest(".status-card");
                if (!card) return;
                const details = card.closest("[data-principle-id]");
                set_status(details.dataset.frameworkId, details.dataset.principleId, card.dataset.status);
            });
            appRoot.addEventListener("input", (e) => {
                if (e.target.matches("textarea.notes")) {
                    const details = e.target.closest("[data-principle-id]");
                    state.frameworks[details.dataset.frameworkId][details.dataset.principleId].notes = e.target.value;
                }
            });
            appRoot.addEventListener("keydown", (e) => {
                if (e.key !== "Enter" && e.key !== " ") return;
                const card = e.target.closest(".status-card");
                if (!card) return;
                e.preventDefault();
                const details = card.closest("[data-principle-id]");
                set_status(details.dataset.frameworkId, details.dataset.principleId, card.dataset.status);
            });
```

- [ ] **Step 6: Manually verify in a browser**

Open `index.html`. Confirm POSI behaves exactly as before: score a few principles (click cards, confirm toggle-to-deselect still works), type notes, confirm the summary and badge update, confirm JSON export/reload round-trips `frameworks.posi` correctly. Inspect a principle's `<details>` element in devtools and confirm it now has both `data-principle-id` and `data-framework-id="posi"`.

- [ ] **Step 7: Commit and push**

```bash
git add index.html
git commit -m "Thread frameworkId through per-principle state access"
git push -u origin $(git branch --show-current)
```

---

### Task 2: Optional per-level criteria and improvement-action text

**Files:**
- Modify: `index.html` (`<script id="app-render">`, `<style>`)

**Interfaces:**
- Consumes: `render_principle(principle, frameworkId)` from Task 1.
- Produces: `render_principle` now also renders `principle.criteria[status]` (per status card, when present) and `principle.improvementAction` (once per principle, when present). Both are optional fields — absent for POSI's principles.

- [ ] **Step 1: Extend `render_principle` to show optional criteria and improvement text**

Replace the entire function (as it stands after Task 1) with:
```js
            function render_principle(principle, frameworkId) {
                const details = document.createElement("details");
                details.className = "container";
                details.dataset.principleId = principle.id;
                details.dataset.frameworkId = frameworkId;

                const summary = document.createElement("summary");
                const h4 = document.createElement("h4");
                h4.textContent = principle.title;
                summary.appendChild(h4);
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
                    card.setAttribute("aria-pressed", state.frameworks[frameworkId][principle.id].status === option.status ? "true" : "false");
                    if (state.frameworks[frameworkId][principle.id].status === option.status) {
                        card.classList.add("selected");
                    }
                    const icon = document.createElement("span");
                    icon.className = "status-icon";
                    icon.innerHTML = ICONS[option.status];
                    card.appendChild(icon);

                    const label = document.createElement("p");
                    label.className = "status-label";
                    label.textContent = option.label;
                    card.appendChild(label);

                    if (principle.criteria && principle.criteria[option.status]) {
                        const criteriaText = document.createElement("p");
                        criteriaText.className = "status-criteria";
                        criteriaText.textContent = principle.criteria[option.status];
                        card.appendChild(criteriaText);
                    }

                    cards.appendChild(card);
                });
                details.appendChild(cards);

                const notes = document.createElement("textarea");
                notes.className = "notes";
                notes.placeholder = "Explain your reasoning (optional)";
                notes.value = state.frameworks[frameworkId][principle.id].notes;
                details.appendChild(notes);

                if (principle.improvementAction) {
                    const improvement = document.createElement("p");
                    improvement.className = "improvement-action";
                    improvement.textContent = "To improve: " + principle.improvementAction;
                    details.appendChild(improvement);
                }

                return details;
            }
```

- [ ] **Step 2: Add CSS for the new optional elements**

In `<style>`, add:
```css
            .status-criteria {
                font-size: 8pt;
                margin: 2pt 0 0;
                font-weight: normal;
            }

            .improvement-action {
                margin-top: 6pt;
                font-style: italic;
            }
```

- [ ] **Step 3: Manually verify in a browser**

Open `index.html`. Confirm POSI's principles render EXACTLY as before — no extra text under any status card's label, no "To improve:" line anywhere (POSI's principles have no `criteria`/`improvementAction` fields, so nothing new should appear). Then, temporarily edit one POSI principle object in `POSI` (e.g. `coverage`) to add `criteria: { compliant: "Test compliant text.", progress: "Test progress text.", "non-compliant": "Test non-compliant text." }` and `improvementAction: "Test improvement text."`, reload, confirm the three cards show their respective criteria text and the "To improve: Test improvement text." line appears below the notes textarea — then **revert this temporary edit** before committing (POSI's actual data must not change in this task).

- [ ] **Step 4: Commit and push**

```bash
git add index.html
git commit -m "Add optional per-level criteria and improvement-action rendering"
git push
```

---

### Task 3: Framework-scoped summary sections

**Files:**
- Modify: `index.html` (`<script id="app-render">`, `<style>`)

**Interfaces:**
- Consumes: `FRAMEWORKS`, `compute_summary(state, frameworkId)` (unchanged from a prior plan).
- Produces: `render_framework_summary(framework)` (gains a parameter, produces framework-scoped ids `#summary-heading-<id>`/`#summary-counts-<id>`/`.summary-section` class), `update_summary()` (now loops over all frameworks instead of hardcoding `"posi"`).

This fixes the bug described in the spec: with multiple frameworks, `#summary-heading`/`#summary-counts` are non-unique ids, so only the first framework's summary would ever update.

- [ ] **Step 1: Update `render_framework_summary` to take a `framework` parameter and produce scoped ids**

Replace:
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
```
with:
```js
            function render_framework_summary(framework) {
                const section = document.createElement("section");
                section.className = "summary-section";
                const heading = document.createElement("strong");
                heading.id = "summary-heading-" + framework.id;
                heading.className = "summary-heading";
                heading.textContent = "Overall: 0% compliant";
                const counts = document.createElement("span");
                counts.id = "summary-counts-" + framework.id;
                section.appendChild(heading);
                section.appendChild(counts);
                return section;
            }
```

- [ ] **Step 2: Update `render_framework_section`'s call site**

Replace:
```js
                section.appendChild(render_framework_summary());
```
with:
```js
                section.appendChild(render_framework_summary(framework));
```

- [ ] **Step 3: Update `update_summary` to loop over all frameworks**

Replace:
```js
            function update_summary() {
                const summary = compute_summary(state, "posi");
                document.getElementById("summary-heading").textContent = "Overall: " + summary.percentCompliant + "% compliant";
                document.getElementById("summary-counts").textContent =
                    summary.compliant + " compliant · " + summary.progress + " making progress · " +
                    summary.nonCompliant + " not compliant (" + summary.scoredCount + "/" + summary.totalCount + " scored)";
            }
```
with:
```js
            function update_summary() {
                FRAMEWORKS.forEach((framework) => {
                    const summary = compute_summary(state, framework.id);
                    const heading = document.getElementById("summary-heading-" + framework.id);
                    const counts = document.getElementById("summary-counts-" + framework.id);
                    if (!heading || !counts) return;
                    heading.textContent = "Overall: " + summary.percentCompliant + "% compliant";
                    counts.textContent =
                        summary.compliant + " compliant · " + summary.progress + " making progress · " +
                        summary.nonCompliant + " not compliant (" + summary.scoredCount + "/" + summary.totalCount + " scored)";
                });
            }
```

- [ ] **Step 4: Update the CSS selectors from id-based to class-based**

Replace:
```css
            #summary-section {
                display: flex;
                align-items: baseline;
                gap: 24pt;
                margin-left: 20pt;
            }

            #summary-heading {
                font-size: large;
                font-weight: bold;
            }
```
with:
```css
            .summary-section {
                display: flex;
                align-items: baseline;
                gap: 24pt;
                margin-left: 20pt;
            }

            .summary-heading {
                font-size: large;
                font-weight: bold;
            }
```

- [ ] **Step 5: Manually verify in a browser**

Open `index.html`. Confirm POSI's summary line still renders and updates correctly when scoring principles (visually identical to before — same layout, same text). Inspect the summary `<strong>` element in devtools and confirm its id is now `summary-heading-posi` (not the old bare `summary-heading`).

- [ ] **Step 6: Commit and push**

```bash
git add index.html
git commit -m "Scope framework summary sections to their framework id"
git push
```

---

### Task 4: Active framework state and generalized badge

**Files:**
- Modify: `index.html` (`<script id="app-render">`)

**Interfaces:**
- Consumes: `FRAMEWORKS`, `state`, `compute_summary`, `escape_xml`, `ring_segment_offsets`, `status_color` (all unchanged).
- Produces: `activeFrameworkId` (new module-level `let`), `ring_config_for(framework)` (new, replaces the `RING_CONFIG` constant), `build_badge_svg_markup(state)`/`build_badge_hovercard_html(state)` (both now read `activeFrameworkId` instead of the literal `"posi"`), `render_sidebar_nav()` (extended — clicking a framework's link or any of its sub-links sets `activeFrameworkId` and refreshes the badge).

- [ ] **Step 1: Add the `activeFrameworkId` module variable**

Find:
```js
            let state = initial_state();
```
and add immediately after it:
```js
            let state = initial_state();
            let activeFrameworkId = FRAMEWORKS[0].id;
```

- [ ] **Step 2: Replace `RING_CONFIG` with a function that computes ring radii from a framework's section count**

Replace:
```js
            const RING_CONFIG = [
                { section: "Governance", radius: 34 },
                { section: "Sustainability", radius: 48 },
                { section: "Insurance", radius: 62 },
            ];
```
with:
```js
            function ring_config_for(framework) {
                return framework.sections.map((section, i) => ({
                    section: section.section,
                    radius: 34 + i * 14,
                }));
            }
```
(This produces the exact same radii POSI has today — 34, 48, 62 for its 3 sections, since `34 + 0*14 = 34`, `34 + 1*14 = 48`, `34 + 2*14 = 62` — and scales to any other section count for a future framework.)

- [ ] **Step 3: Update `build_badge_svg_markup` to use `activeFrameworkId` and `ring_config_for`**

Replace:
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
with:
```js
            function build_badge_svg_markup(state) {
                const width = 200;
                const height = 260;
                const center = width / 2;
                const ringCenterY = 140;
                const strokeWidth = 10;
                let circles = "";
                const framework = FRAMEWORKS.find((f) => f.id === activeFrameworkId);
                ring_config_for(framework).forEach((ring) => {
                    const section = framework.sections.find((s) => s.section === ring.section);
                    const offsets = ring_segment_offsets(ring.radius, section.principles.length);
                    section.principles.forEach((principle, i) => {
                        const status = state.frameworks[framework.id][principle.id].status;
                        const color = status_color(status);
                        const off = offsets[i];
                        circles +=
                            '<circle cx="' + center + '" cy="' + ringCenterY + '" r="' + ring.radius +
                            '" fill="none" stroke="' + color + '" stroke-width="' + strokeWidth +
                            '" stroke-dasharray="' + off.dasharray + '" stroke-dashoffset="' + off.dashoffset +
                            '" transform="rotate(-90 ' + center + " " + ringCenterY + ')" />';
                    });
                });
                const summary = compute_summary(state, framework.id);
                const name = escape_xml(state.name || "Unnamed infrastructure");
                const date = escape_xml(state.assessedOn || "");
                const badgeTitle = escape_xml(framework.shortName + " Compliance");
                return (
                    '<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 ' + width + " " + height + '" width="' + width + '" height="' + height + '">' +
                    '<rect width="' + width + '" height="' + height + '" fill="white" />' +
                    '<text x="' + center + '" y="24" text-anchor="middle" font-size="12" font-family="&quot;Source Sans 3&quot;, sans-serif" font-weight="bold">' + badgeTitle + "</text>" +
                    circles +
                    '<text x="' + center + '" y="' + (ringCenterY + 7) + '" text-anchor="middle" font-size="26" font-family="&quot;Source Sans 3&quot;, sans-serif" font-weight="bold">' + summary.percentCompliant + "%</text>" +
                    '<text x="' + center + '" y="228" text-anchor="middle" font-size="11" font-family="&quot;Source Sans 3&quot;, sans-serif">' + name + "</text>" +
                    '<text x="' + center + '" y="244" text-anchor="middle" font-size="10" font-family="&quot;Source Sans 3&quot;, sans-serif">' + date + "</text>" +
                    "</svg>"
                );
            }
```

- [ ] **Step 4: Update `build_badge_hovercard_html` to use `activeFrameworkId`**

Replace:
```js
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
with:
```js
            function build_badge_hovercard_html(state) {
                const framework = FRAMEWORKS.find((f) => f.id === activeFrameworkId);
                const summary = compute_summary(state, framework.id);
                const name = state.name || "Unnamed infrastructure";
                let html = "<strong>" + escape_xml(name) + "</strong>";
                html += "<div>Assessed " + escape_xml(state.assessedOn) + "</div>";
                html += "<div>" + escape_xml(framework.shortName) + " overall: " + summary.percentCompliant + "%</div>";
                framework.sections.forEach((section) => {
                    let compliant = 0;
                    let progress = 0;
                    let nonCompliant = 0;
                    section.principles.forEach((principle) => {
                        const status = state.frameworks[framework.id][principle.id].status;
                        if (status === "compliant") compliant++;
                        else if (status === "progress") progress++;
                        else if (status === "non-compliant") nonCompliant++;
                    });
                    html += "<div>" + escape_xml(section.section) + ": " + compliant + "✓ " + progress + "◐ " + nonCompliant + "✗</div>";
                });
                return html;
            }
```

- [ ] **Step 5: Wire up "clicking a framework's nav link switches the active framework"**

Replace:
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
                    a.addEventListener("click", (e) => e.stopPropagation());
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
                    a.addEventListener("click", (e) => {
                        e.stopPropagation();
                        activeFrameworkId = framework.id;
                        update_badge();
                    });
                    groupDetails.appendChild(summary);

                    const subUl = document.createElement("ul");
                    framework.sections.forEach((section) => {
                        const subLink = render_nav_link("#" + posi_section_id(section), section.section);
                        subLink.querySelector("a").addEventListener("click", () => {
                            activeFrameworkId = framework.id;
                            update_badge();
                        });
                        subUl.appendChild(subLink);
                    });
                    groupDetails.appendChild(subUl);

                    li.appendChild(groupDetails);
                    ul.appendChild(li);
                });

                nav.appendChild(ul);
            }
```

- [ ] **Step 6: Manually verify in a browser**

Open `index.html`. Confirm the badge still shows "POSI Compliance" (since `activeFrameworkId` defaults to `FRAMEWORKS[0].id`, which is `"posi"`), 3 rings, same layout as before. Click "POSI v2.0 Assessment" in the sidebar and each of its sub-links (Governance/Sustainability/Insurance) — confirm the page still scrolls correctly and the badge keeps showing POSI's data (no change, since POSI is the only framework). Then, temporarily add a second, minimal `FRAMEWORKS` entry (e.g. `{ id: "test", title: "Test Framework", shortName: "Test", description: "...", aboutHtml: "<p>...</p>", sections: [{ section: "Test Section", sourceUrl: "https://example.org", principles: [{ id: "test-principle", title: "Test principle", text: "..." }] }] }`) to confirm: clicking its nav link switches the badge to show "Test Compliance" with 1 ring; clicking back to POSI's nav link switches the badge back. **Remove this temporary framework entry** before committing.

- [ ] **Step 7: Commit and push**

```bash
git add index.html
git commit -m "Generalize badge to follow the active framework"
git push
```

---

### Task 5: Per-framework draft label, replacing the global watermark

**Files:**
- Modify: `index.html` (`<body>` static markup, `<script id="app-render">`, `<style>`)

**Interfaces:**
- Consumes: `render_framework_section(framework)`, `render_sidebar_nav()` (both from earlier tasks/prior plan).
- Produces: an optional `draft` boolean field on a `FRAMEWORKS` entry (undocumented/absent for POSI); `render_framework_section` shows a `.draft-label` badge next to the framework's `<h2>` when `framework.draft` is true; the sidebar nav shows `"<title> (draft)"` for that framework's link.

- [ ] **Step 1: Remove the global watermark from `<body>`**

Find and delete this line from `<body>` (it currently sits right before `<div data-sidebar-layout id="app-shell">`):
```html
        <span id="watermark">DRAFT – WORK IN PROGRESS</span>

```
(Remove the line and the blank line after it, so `<body>` goes straight from its opening tag to the `<div data-sidebar-layout id="app-shell">` line.)

- [ ] **Step 2: Remove the `#watermark` CSS rule**

Find and delete, in `<style>`:
```css
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

```

- [ ] **Step 3: Add the `.draft-label` CSS rule**

In `<style>`, add:
```css
            .draft-label {
                margin-left: 8pt;
                padding: 1pt 6pt;
                font-size: 9pt;
                font-weight: normal;
                color: var(--warning);
                border: 1pt solid var(--warning);
                border-radius: 3pt;
                vertical-align: middle;
            }
```

- [ ] **Step 4: Show the draft label in `render_framework_section` when `framework.draft` is true**

Find, in `render_framework_section(framework)`:
```js
                const h2 = document.createElement("h2");
                h2.textContent = framework.title;
                section.appendChild(h2);
```
Replace with:
```js
                const h2 = document.createElement("h2");
                h2.textContent = framework.title;
                if (framework.draft) {
                    const draftLabel = document.createElement("span");
                    draftLabel.className = "draft-label";
                    draftLabel.textContent = "Draft – work in progress";
                    h2.appendChild(draftLabel);
                }
                section.appendChild(h2);
```

- [ ] **Step 5: Show a `(draft)` suffix on the framework's nav link when `framework.draft` is true**

Find, in `render_sidebar_nav()` (as it stands after Task 4):
```js
                    const a = document.createElement("a");
                    a.href = "#framework-" + framework.id;
                    a.textContent = framework.title;
```
Replace with:
```js
                    const a = document.createElement("a");
                    a.href = "#framework-" + framework.id;
                    a.textContent = framework.title + (framework.draft ? " (draft)" : "");
```

- [ ] **Step 6: Manually verify in a browser**

Open `index.html`. Confirm the old diagonal "DRAFT – WORK IN PROGRESS" watermark is completely gone, and POSI's title/nav entry show no draft label or suffix (POSI's `FRAMEWORKS` entry has no `draft` field). Then temporarily set `draft: true` on POSI's entry in `FRAMEWORKS`, reload, and confirm a small bordered "Draft – work in progress" label appears next to "POSI v2.0 Assessment" in the main content, and the sidebar nav link reads "POSI v2.0 Assessment (draft)". **Revert this temporary change** before committing — POSI must not actually be marked draft.

- [ ] **Step 7: Commit and push**

```bash
git add index.html
git commit -m "Replace global watermark with a per-framework draft label"
git push
```

---

### Task 6: Update project documentation

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:** none (documentation only).

- [ ] **Step 1: Update `CLAUDE.md`'s Architecture section**

Read the current `## Architecture` section of `CLAUDE.md`. Find the paragraph describing the data model (it currently mentions `FRAMEWORKS`, `ACTIVITIES`, `ICONS`, `initial_state()`, the `state.frameworks` shape, and `compute_summary(state, frameworkId)`). Add, after that paragraph (or merge into it, whichever reads more naturally given the exact current text), a new sentence/short paragraph:

```markdown
Each principle object may optionally carry `criteria: {compliant, progress, "non-compliant"}` (short rubric text shown per status card) and `improvementAction` (shown as a "To improve:" note) — both absent for POSI's principles today, rendered only when present, so a framework can opt into richer guidance without affecting others.
```

Find the paragraph describing rendering (mentions `render()`, the header, OS Infrastructure Description, framework sections, sidebar nav/footer). Add a sentence noting: `render_principle` now takes a `frameworkId` (stored as `details.dataset.frameworkId`) so each principle's status/notes read from and write to the correct framework's slice of `state.frameworks`; framework summary sections use scoped ids (`#summary-heading-<frameworkId>` etc.) so multiple frameworks' summaries update independently; and a module-level `activeFrameworkId` (not persisted in exported JSON) — set by clicking a framework's nav link or any of its section sub-links — drives which framework's data the assessment badge (title, rings, hovercard) displays, with `ring_config_for(framework)` computing ring radii from however many sections that framework has instead of a hardcoded list.

Find the sentence(s) describing the watermark (likely near a `## Notes` section, referencing `<span id="watermark">`). Replace with a note that the global watermark was replaced by a per-framework `draft: true` flag on a `FRAMEWORKS` entry, rendered as a small label next to that framework's title and nav entry (`.draft-label`) — read the actual current wording first and make a precise, accurate edit rather than guessing at exact phrasing.

- [ ] **Step 2: Commit and push**

```bash
git add CLAUDE.md
git commit -m "Update docs for multi-framework machinery generalization"
git push
```
