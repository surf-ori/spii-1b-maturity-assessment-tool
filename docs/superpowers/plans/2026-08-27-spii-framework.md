# SPII v0.0.1 Framework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add SPII v0.0.1 as a second `FRAMEWORKS` entry in `index.html` — 5 principles, 19 scoreable subprinciples, each with a description, a 3-level rubric (compliant/making progress/not compliant), and a suggested improvement action — built entirely on the generalized multi-framework machinery already merged to `main`.

**Architecture:** This is a data-only change. A new `SPII` constant (mirroring `POSI`'s existing shape: an array of `{section, sourceUrl, principles}`) is added to `<script id="app-logic">`, and a second entry referencing it is appended to the existing `FRAMEWORKS` array. No rendering/JS logic changes are expected — the machinery (frameworkId-scoped state, scoped summaries, `activeFrameworkId`-driven badge with dynamic ring count, `section_id`, the draft-label mechanism) already handles an arbitrary second framework generically.

**Tech Stack:** Vanilla HTML/CSS/JS, no build step, no new dependency.

**Spec:** `docs/superpowers/specs/2026-08-27-spii-framework-design.md`

## Global Constraints

- Everything lives in one file, `index.html` — no build step, no package manager, no server, no JS framework.
- **No rendering/logic changes.** If implementing a task reveals a need to modify anything outside `<script id="app-logic">`'s data constants, STOP and escalate — that means the prior machinery-generalization plan missed something, and patching around it here would be the wrong fix.
- POSI's data and behavior must be completely unaffected — every task's manual verification must confirm this.
- All principle/criteria/improvementAction text must match the spec's Content section verbatim — including typographic punctuation (curly quotes ’ “ ” and em dashes —, matching the house style already used throughout `POSI`'s own data), not straight ASCII equivalents. **Use straight double quotes (`"`) only as JS string delimiters** — never Unicode curly quotes as delimiters (a prior task on this file once used curly quotes as JS string delimiters, which is a fatal syntax error; verify with `node --check` per every task's Step instructions below).
- SPII's `draft: true` flag and `id: "spii"` framework id are fixed values — do not omit or rename them.

---

### Task 1: SPII framework shell + Openness section

**Files:**
- Modify: `index.html` (`<script id="app-logic">`)

**Interfaces:**
- Consumes: existing `POSI`/`FRAMEWORKS` declarations (unchanged), `initial_state()` (unchanged — it already builds `state.frameworks[frameworkId]` generically from whatever `FRAMEWORKS` contains, so adding a second entry automatically gets a correctly-shaped state slice with no code change).
- Produces: `SPII` (new top-level `const`, an array with one section object so far — "Openness", 6 principles), `FRAMEWORKS[1]` (new entry: `{id: "spii", title, shortName, description, aboutHtml, draft: true, sections: SPII}`).

This task creates the SPII framework "shell" (all its metadata) plus its first section. Tasks 2-5 each append one more section to the same `SPII` array.

- [ ] **Step 1: Insert the `SPII` constant with its first section (Openness)**

Find, in `<script id="app-logic">` (this is the line right after `POSI`'s closing `];`, immediately before `const FRAMEWORKS = [`):
```js
            ];

            const FRAMEWORKS = [
                {
                    id: "posi",
```
Replace with:
```js
            ];

            const SPII = [
                {
                    section: "Openness",
                    sourceUrl: "https://zenodo.org/record/7433767",
                    principles: [
                        { id: "spii-governance", title: "Governance", text: "Transparency of how the infrastructure is governed — decision-making bodies, policies, and how the community can see and influence them.", criteria: { compliant: "Governance structures, decision-making processes, and policies are fully documented and publicly available.", progress: "Some governance information is public, but key decisions or policies are not consistently documented or disclosed.", "non-compliant": "Governance is opaque; decision-making processes are not documented or accessible to the community." }, improvementAction: "Publish governance charters, meeting minutes/decisions, and policy documents openly." },
                        { id: "spii-operations", title: "Operations", text: "Transparency of day-to-day operations — how the infrastructure runs, its procedures, and operational metrics.", criteria: { compliant: "Operational procedures, SLAs, incident reports, and key operational metrics are openly published and kept up to date.", progress: "Some operational information is shared, but not consistently or not comprehensively.", "non-compliant": "Operational practices are undocumented or not shared with users/community." }, improvementAction: "Publish operating procedures and regularly updated status/incident reports." },
                        { id: "spii-provenance", title: "Provenance", text: "Ability to trace the origin, custody, and changes of data, content, software, and decisions within the infrastructure.", criteria: { compliant: "Full provenance (origin, versioning, custody chain) is tracked and exposed for data, software, and content.", progress: "Provenance is tracked internally but not consistently exposed or documented for users.", "non-compliant": "Provenance information is not tracked or is unavailable." }, improvementAction: "Implement provenance tracking (e.g. versioning, changelogs, PID-linked history)." },
                        { id: "spii-economical", title: "Economical", text: "Transparency about the financial model — costs, funding sources, pricing, and how money flows through the infrastructure.", criteria: { compliant: "Detailed financials, funding sources, and cost/pricing models are openly published.", progress: "High-level financial information is available, but details (e.g. cost breakdowns, funding sources) are limited or inconsistent.", "non-compliant": "Financial information is not disclosed." }, improvementAction: "Publish an annual financial transparency report." },
                        { id: "spii-strategical", title: "Strategical", text: "Transparency about strategic direction — roadmaps, priorities, and long-term plans.", criteria: { compliant: "Strategic/product roadmaps and priorities are publicly available and regularly updated.", progress: "A roadmap exists but is not public, not regularly updated, or lacks detail.", "non-compliant": "No strategic roadmap exists or is shared." }, improvementAction: "Publish and maintain a public strategic roadmap." },
                        { id: "spii-accessibility", title: "Accessibility", text: "The infrastructure’s software, data, APIs, and outputs are openly accessible, using open standards and licences, secured by design, and reachable by all relevant users including citizen scientists.", criteria: { compliant: "Software is open-source, data/metadata/APIs are open, licences are clear and open, the infrastructure runs on a public/open infrastructure stack, security is built-in by design, and identity/access management supports broad access including citizen scientists.", progress: "Some components are open (e.g. open data or open-source software) but others remain closed, proprietary, or access-restricted.", "non-compliant": "Software, data, APIs, or access are closed/proprietary with no clear licensing or public access path." }, improvementAction: "Work through the open-X checklist — open-source software, open data, open APIs, open repositories, open metadata, open licences, public infrastructure stack, security-by-design, and inclusive IAM." },
                    ],
                },
            ];

            const FRAMEWORKS = [
                {
                    id: "posi",
```

- [ ] **Step 2: Add the SPII entry to `FRAMEWORKS`**

Find:
```js
                    sections: POSI,
                },
            ];
```
Replace with:
```js
                    sections: POSI,
                },
                {
                    id: "spii",
                    title: "SPII v0.0.1 Assessment",
                    shortName: "SPII",
                    description: "A community-drafted framework (v0.0.1, June 2026 consultation) covering openness, autonomy, sustainability, interoperability, and researcher-centricity for open science infrastructure.",
                    aboutHtml:
                        "<p>SPII (version 0.0.1) is a draft assessment framework developed through the SPII project’s community consultation, concluding in June 2026. It reflects five principles — Openness, Autonomy, Sustainability, Interoperability, and Researcher-centric — each broken into subprinciples with concrete actions, enablers, and practices infrastructures can adopt.</p>" +
                        "<p>The framework’s underlying values — quality and integrity, collective benefit, and equity, fairness, diversity and inclusion — draw on UNESCO’s Open Science Recommendation and the Dutch National Plan Open Science (NPOS). This is a v0.0.1 draft: principles and criteria may change as community feedback continues.</p>" +
                        "<p>References: NPOS, <em>Open Science 2030 in the Netherlands: NPOS2030 Ambition Document and Rolling Agenda</em> (2022), <a href=\"https://zenodo.org/record/7433767\">https://zenodo.org/record/7433767</a>. UNESCO, <em>Recommendation on Open Science</em>, <a href=\"https://unesdoc.unesco.org/ark:/48223/pf0000379949\">unesdoc.unesco.org</a>. Lin, D. et al., <em>The TRUST Principles for digital repositories</em>, Sci. Data 7, 144 (2020), <a href=\"https://doi.org/10.1038/s41597-020-0486-7\">https://doi.org/10.1038/s41597-020-0486-7</a>. <em>The Principles of Open Scholarly Infrastructure</em> (v2.0, 2025), <a href=\"https://openscholarlyinfrastructure.org/\">openscholarlyinfrastructure.org</a>. Bijsterbosch, M. et al., <em>Seven Guiding Principles for Open Research Information</em> (2022), <a href=\"https://zenodo.org/records/6074944\">https://zenodo.org/records/6074944</a>. Universities of the Netherlands, <em>A new open access strategy for the Netherlands</em> (2026), <a href=\"https://doi.org/10.5281/zenodo.19548009\">https://doi.org/10.5281/zenodo.19548009</a>.</p>",
                    draft: true,
                    sections: SPII,
                },
            ];
```

- [ ] **Step 2: Verify — Node structural check**

Write a throwaway script (anywhere outside the repo, e.g. `/tmp`) and run it from the repository root:
```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
assert.ok(match, 'app-logic script block not found');
const context = vm.createContext({});
vm.runInContext(match[1], context);

assert.equal(context.FRAMEWORKS.length, 2, 'expected 2 frameworks');
const spii = context.FRAMEWORKS[1];
assert.equal(spii.id, 'spii');
assert.equal(spii.draft, true);
assert.equal(spii.sections.length, 1, 'expected 1 section so far (Openness)');
assert.equal(spii.sections[0].section, 'Openness');
assert.equal(spii.sections[0].principles.length, 6);
const ids = spii.sections[0].principles.map((p) => p.id);
assert.deepEqual(ids, ['spii-governance', 'spii-operations', 'spii-provenance', 'spii-economical', 'spii-strategical', 'spii-accessibility']);
spii.sections[0].principles.forEach((p) => {
    assert.ok(p.criteria.compliant && p.criteria.progress && p.criteria['non-compliant'], `${p.id} missing criteria`);
    assert.ok(p.improvementAction, `${p.id} missing improvementAction`);
});

// POSI untouched
assert.equal(context.FRAMEWORKS[0].id, 'posi');
assert.equal(context.FRAMEWORKS[0].sections.length, 3);

console.log('Task 1 structural checks passed.');
```
Run: `node /path/to/verify.mjs` (from the repo root). Expected: PASS, prints "Task 1 structural checks passed."

- [ ] **Step 3: JS syntax and CRLF checks**

```python
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
for sid in ['app-logic', 'app-render']:
    m = re.search(r'<script id=\"' + sid + r'\">(.*?)</script>', html, re.S)
    open('/tmp/' + sid + '-check.js', 'w', encoding='utf-8').write(m.group(1))
"
node --check /tmp/app-logic-check.js && echo "app-logic OK"
node --check /tmp/app-render-check.js && echo "app-render OK"
```
```
python3 -c "d=open('index.html','rb').read(); print('crlf', d.count(b'\r\n')); print('lf_only', d.count(b'\n') - d.count(b'\r\n'))"
```
`lf_only` must be `0` both before and after your edit (run the command before you start too, and compare). This file uses CRLF line endings throughout and has had corruption from imprecise (full-file-rewrite) edits multiple times on prior plans — use targeted `Edit` calls only, never rewrite the whole file.

- [ ] **Step 4: Manually verify in a browser**

Open `index.html`. Confirm: the sidebar shows a second expandable group "SPII v0.0.1 Assessment (draft)" with one sub-link, "Openness"; SPII's `<h2>` shows the "Draft – work in progress" label; clicking SPII's nav link switches the badge to "SPII Compliance" with exactly 1 ring (6 segments); POSI's own section/nav/badge behavior is completely unchanged. If no browser is available, the Node structural check above plus a careful read of the inserted JSON-like data (confirming braces/commas balance and every principle has `id`/`title`/`text`/`criteria`/`improvementAction`) is sufficient — the rendering machinery itself is unchanged and already covered by the prior plan's tests.

- [ ] **Step 5: Commit and push**

```bash
git add index.html
git commit -m "Add SPII v0.0.1 framework shell and Openness section"
git push -u origin $(git branch --show-current)
```

---

### Task 2: Add Autonomy section

**Files:**
- Modify: `index.html` (`<script id="app-logic">`)

**Interfaces:**
- Consumes: `SPII` (Task 1).
- Produces: `SPII` now has 2 sections (Openness, Autonomy — 10 principles total).

- [ ] **Step 1: Append the Autonomy section to `SPII`**

Find, at the end of the `SPII` array (this exact closing sequence is unique in the file — it's the only place `];` is immediately followed by `const FRAMEWORKS = [`):
```js
                    ],
                },
            ];

            const FRAMEWORKS = [
```
Replace with:
```js
                    ],
                },
                {
                    section: "Autonomy",
                    sourceUrl: "https://zenodo.org/record/7433767",
                    principles: [
                        { id: "spii-stakeholder-governed", title: "Stakeholder governed/community-led", text: "The infrastructure is governed by its stakeholder community rather than a single external party.", criteria: { compliant: "A representative governing body drawn from the stakeholder community makes key decisions, with documented community input mechanisms.", progress: "Some community input is solicited, but governance is largely controlled by a single organisation or vendor.", "non-compliant": "Governance is entirely external/vendor-controlled with no community input." }, improvementAction: "Establish a stakeholder governing board and open, non-discriminatory participation." },
                        { id: "spii-discipline-specific", title: "Discipline-specific", text: "The infrastructure is fit for the specific needs, norms, and workflows of its research discipline(s).", criteria: { compliant: "The infrastructure is explicitly designed and regularly reviewed against the needs of its target discipline(s), with documented fit-for-purpose criteria.", progress: "Some discipline-specific needs are addressed, but the infrastructure is largely generic and not tailored.", "non-compliant": "No consideration of discipline-specific needs; infrastructure is a one-size-fits-all solution regardless of context." }, improvementAction: "Run periodic fit-for-purpose reviews with discipline representatives." },
                        { id: "spii-digital-sovereignty", title: "Digital sovereignty", text: "The infrastructure safeguards digital and academic sovereignty — control over data, infrastructure, and decisions stays with the academic community rather than being ceded to external commercial or geopolitical interests.", criteria: { compliant: "Data, infrastructure control, and critical decisions remain within community/academic governance; dependency on external vendors is actively managed with exit strategies.", progress: "Some dependency on external vendors/platforms exists, with partial mitigation (e.g. contractual safeguards) but no full exit strategy.", "non-compliant": "Critical infrastructure or data is fully dependent on external vendors/platforms with no contractual safeguards or exit plan." }, improvementAction: "Adopt value-driven procurement, balance agency against costs & risks, and put contractual clauses and exit strategies in place — consider using the Digital Autonomy Assessment Framework for a deeper dive." },
                        { id: "spii-responsibility", title: "Responsibility", text: "Clear ownership of responsibilities for the infrastructure’s operation, decisions, and consequences.", criteria: { compliant: "Responsibilities are clearly assigned, documented, and regularly reviewed against the infrastructure’s purpose and community value.", progress: "Some responsibilities are assigned but not comprehensively documented or reviewed.", "non-compliant": "Responsibilities are unclear or undocumented." }, improvementAction: "Document a responsibility matrix (who is accountable for what) and schedule regular reviews of purpose and community value." },
                    ],
                },
            ];

            const FRAMEWORKS = [
```

- [ ] **Step 2: Verify — Node structural check**

Same script pattern as Task 1's Step 2, updated for the new state:
```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
const context = vm.createContext({});
vm.runInContext(match[1], context);

const spii = context.FRAMEWORKS[1];
assert.equal(spii.sections.length, 2, 'expected 2 sections (Openness, Autonomy)');
assert.equal(spii.sections[1].section, 'Autonomy');
assert.equal(spii.sections[1].principles.length, 4);
const ids = spii.sections[1].principles.map((p) => p.id);
assert.deepEqual(ids, ['spii-stakeholder-governed', 'spii-discipline-specific', 'spii-digital-sovereignty', 'spii-responsibility']);
spii.sections[1].principles.forEach((p) => {
    assert.ok(p.criteria.compliant && p.criteria.progress && p.criteria['non-compliant'], `${p.id} missing criteria`);
    assert.ok(p.improvementAction, `${p.id} missing improvementAction`);
});
// Openness (Task 1) still intact
assert.equal(spii.sections[0].principles.length, 6);
assert.equal(context.FRAMEWORKS[0].sections.length, 3); // POSI untouched

console.log('Task 2 structural checks passed.');
```
Run from the repo root. Expected: PASS.

- [ ] **Step 3: JS syntax and CRLF checks**

Same commands as Task 1 Step 3. `lf_only` must be `0` before and after.

- [ ] **Step 4: Manually verify in a browser**

Confirm SPII's sidebar group now shows 2 sub-links (Openness, Autonomy); clicking SPII's badge now shows 2 rings (6 + 4 segments); POSI unaffected. Code-trace verification is acceptable if no browser is available, per Task 1's Step 4 note.

- [ ] **Step 5: Commit and push**

```bash
git add index.html
git commit -m "Add SPII Autonomy section"
git push
```

---

### Task 3: Add Sustainability section

**Files:**
- Modify: `index.html` (`<script id="app-logic">`)

**Interfaces:**
- Consumes: `SPII` (Tasks 1-2).
- Produces: `SPII` now has 3 sections (Openness, Autonomy, Sustainability — 14 principles total).

- [ ] **Step 1: Append the Sustainability section to `SPII`**

Find (the unique closing sequence, now after Autonomy):
```js
                    ],
                },
            ];

            const FRAMEWORKS = [
```
Replace with:
```js
                    ],
                },
                {
                    section: "Sustainability",
                    sourceUrl: "https://zenodo.org/record/7433767",
                    principles: [
                        { id: "spii-financial-sustainability", title: "Financial sustainability", text: "The infrastructure has a sustainable financial model beyond short-term grants.", criteria: { compliant: "Structural/recurrent funding is secured (e.g. memberships, recurrent grants), with a mission-consistent revenue model and clear cost-of-ownership understanding.", progress: "Funding is time-limited (e.g. a one-off grant) with plans underway to secure recurrent funding.", "non-compliant": "No funding secured for the foreseeable future." }, improvementAction: "Develop mission-consistent revenue generation and a total-cost-of-ownership analysis; consider make-or-buy guidelines." },
                        { id: "spii-human-capacity", title: "Human capacity sustainability", text: "Sufficient, resilient staffing to operate and maintain the infrastructure.", criteria: { compliant: "A dedicated, adequately resourced team maintains the infrastructure, with succession/continuity planning in place.", progress: "The infrastructure is maintained but relies heavily on one or few individuals with no succession plan.", "non-compliant": "No one is actively maintaining the infrastructure." }, improvementAction: "Build a resourcing/succession plan and invest in sustained maintenance capacity." },
                        { id: "spii-legal-compliance", title: "Legal and institutional compliance", text: "The infrastructure meets applicable legal and institutional requirements.", criteria: { compliant: "Legal/regulatory compliance (e.g. data protection, IP, institutional policy) is actively managed, documented, and audited or certified.", progress: "Compliance is partially addressed, with known gaps being worked on.", "non-compliant": "Compliance status is unknown or has major gaps." }, improvementAction: "Conduct a compliance audit and address gaps against relevant legal/institutional requirements." },
                        { id: "spii-enduring-availability", title: "Enduring availability", text: "The infrastructure and its outputs remain available over the long term, with plans for what happens if it winds down.", criteria: { compliant: "A documented end-of-life/succession policy exists, content is deposited with trusted third-party archives, and continuity of the public stack is supported.", progress: "Some continuity measures exist (e.g. backups) but no formal end-of-life policy or archival deposit.", "non-compliant": "No continuity plan; data/content would be lost if the infrastructure stopped." }, improvementAction: "Publish an end-of-life policy (a “living will”) and deposit outputs with a trusted third-party digital archive." },
                    ],
                },
            ];

            const FRAMEWORKS = [
```

- [ ] **Step 2: Verify — Node structural check**

```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
const context = vm.createContext({});
vm.runInContext(match[1], context);

const spii = context.FRAMEWORKS[1];
assert.equal(spii.sections.length, 3, 'expected 3 sections (Openness, Autonomy, Sustainability)');
assert.equal(spii.sections[2].section, 'Sustainability');
assert.equal(spii.sections[2].principles.length, 4);
const ids = spii.sections[2].principles.map((p) => p.id);
assert.deepEqual(ids, ['spii-financial-sustainability', 'spii-human-capacity', 'spii-legal-compliance', 'spii-enduring-availability']);
spii.sections[2].principles.forEach((p) => {
    assert.ok(p.criteria.compliant && p.criteria.progress && p.criteria['non-compliant'], `${p.id} missing criteria`);
    assert.ok(p.improvementAction, `${p.id} missing improvementAction`);
});
assert.equal(spii.sections[0].principles.length, 6);
assert.equal(spii.sections[1].principles.length, 4);
assert.equal(context.FRAMEWORKS[0].sections.length, 3); // POSI untouched

console.log('Task 3 structural checks passed.');
```

- [ ] **Step 3: JS syntax and CRLF checks**

Same commands as Task 1 Step 3.

- [ ] **Step 4: Manually verify in a browser**

Confirm SPII's sidebar shows 3 sub-links; badge shows 3 rings (6+4+4 segments); POSI unaffected.

- [ ] **Step 5: Commit and push**

```bash
git add index.html
git commit -m "Add SPII Sustainability section"
git push
```

---

### Task 4: Add Interoperability section

**Files:**
- Modify: `index.html` (`<script id="app-logic">`)

**Interfaces:**
- Consumes: `SPII` (Tasks 1-3).
- Produces: `SPII` now has 4 sections (Openness, Autonomy, Sustainability, Interoperability — 17 principles total).

- [ ] **Step 1: Append the Interoperability section to `SPII`**

Find (the unique closing sequence, now after Sustainability):
```js
                    ],
                },
            ];

            const FRAMEWORKS = [
```
Replace with:
```js
                    ],
                },
                {
                    section: "Interoperability",
                    sourceUrl: "https://zenodo.org/records/6074944",
                    principles: [
                        { id: "spii-technical-interoperability", title: "Technical interoperability", text: "Systems, data and services can be connected, migrated or integrated using open, syntactic standards.", criteria: { compliant: "The infrastructure uses open standards, persistent identifiers (PIDs), and maintains active connections to relevant international infrastructures.", progress: "Some open standards/PIDs are used, but integration with international infrastructures is limited or ad hoc.", "non-compliant": "Proprietary formats/protocols are used with no PIDs and no external connections." }, improvementAction: "Adopt open standards and PIDs, and establish connections to relevant international infrastructures." },
                        { id: "spii-organisational-interoperability", title: "Organisational & social interoperability", text: "Shared semantics — vocabularies and ontologies — enable the infrastructure’s data and processes to be understood consistently across organisations and communities.", criteria: { compliant: "Community-recognised vocabularies/ontologies are used consistently and documented.", progress: "Some shared vocabularies are used, but inconsistently or without documentation.", "non-compliant": "No shared vocabularies/ontologies are used; semantics are ad hoc or undocumented." }, improvementAction: "Adopt and document relevant community vocabularies/ontologies." },
                        { id: "spii-legal-interoperability", title: "Legal interoperability", text: "Clear, compatible terms of use enable data, software, and content to be legally reused and combined across systems.", criteria: { compliant: "Terms of use are clear, machine-readable where possible, and compatible with common open licences used by peer infrastructures.", progress: "Terms of use exist but are unclear, inconsistent, or not machine-readable.", "non-compliant": "No clear terms of use, or terms actively conflict with reuse/interoperability." }, improvementAction: "Publish clear, standard, machine-readable terms of use aligned with common open licences." },
                    ],
                },
            ];

            const FRAMEWORKS = [
```

- [ ] **Step 2: Verify — Node structural check**

```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
const context = vm.createContext({});
vm.runInContext(match[1], context);

const spii = context.FRAMEWORKS[1];
assert.equal(spii.sections.length, 4, 'expected 4 sections');
assert.equal(spii.sections[3].section, 'Interoperability');
assert.equal(spii.sections[3].principles.length, 3);
const ids = spii.sections[3].principles.map((p) => p.id);
assert.deepEqual(ids, ['spii-technical-interoperability', 'spii-organisational-interoperability', 'spii-legal-interoperability']);
spii.sections[3].principles.forEach((p) => {
    assert.ok(p.criteria.compliant && p.criteria.progress && p.criteria['non-compliant'], `${p.id} missing criteria`);
    assert.ok(p.improvementAction, `${p.id} missing improvementAction`);
});
assert.equal(spii.sections[0].principles.length, 6);
assert.equal(spii.sections[1].principles.length, 4);
assert.equal(spii.sections[2].principles.length, 4);
assert.equal(context.FRAMEWORKS[0].sections.length, 3); // POSI untouched

console.log('Task 4 structural checks passed.');
```

- [ ] **Step 3: JS syntax and CRLF checks**

Same commands as Task 1 Step 3.

- [ ] **Step 4: Manually verify in a browser**

Confirm SPII's sidebar shows 4 sub-links; badge shows 4 rings (6+4+4+3 segments) — this is the count where the prior plan's ring-radius-capping fix (`ring_config_for`) starts to matter (radii no longer simply `34 + i*14`); confirm the rings don't overlap or clip. POSI unaffected.

- [ ] **Step 5: Commit and push**

```bash
git add index.html
git commit -m "Add SPII Interoperability section"
git push
```

---

### Task 5: Add Researcher-centric section (completes SPII)

**Files:**
- Modify: `index.html` (`<script id="app-logic">`)

**Interfaces:**
- Consumes: `SPII` (Tasks 1-4).
- Produces: `SPII` complete — 5 sections, 19 principles total.

- [ ] **Step 1: Append the Researcher-centric section to `SPII`**

Find (the unique closing sequence, now after Interoperability):
```js
                    ],
                },
            ];

            const FRAMEWORKS = [
```
Replace with:
```js
                    ],
                },
                {
                    section: "Researcher-centric",
                    sourceUrl: "https://doi.org/10.1038/s41597-020-0486-7",
                    principles: [
                        { id: "spii-seamless-ux", title: "Seamless user-experience", text: "The infrastructure is designed around researchers’ actual workflows and needs.", criteria: { compliant: "User-driven design practices (e.g. user research, usability testing) are embedded in the infrastructure’s development process, and adoption/satisfaction is actively monitored.", progress: "Some user feedback is gathered, but design is not systematically user-driven.", "non-compliant": "No user research or feedback process exists; design decisions are made without researcher input." }, improvementAction: "Establish ongoing user-driven design practices (user research, usability testing, feedback loops)." },
                        { id: "spii-recognition-rewards", title: "Recognition and rewards", text: "Contributions made via the infrastructure (e.g. data, code, review, curation) are recognised and rewarded within academic evaluation and career systems.", criteria: { compliant: "The infrastructure actively supports recognition and rewards for contributions (e.g. integrates with CRediT, ORCID, institutional recognition/reward frameworks).", progress: "Some recognition mechanisms exist (e.g. basic attribution) but are not integrated with formal reward/evaluation systems.", "non-compliant": "No mechanism exists to recognise or reward contributions made via the infrastructure." }, improvementAction: "Integrate with recognition and rewards frameworks (e.g. CRediT, institutional recognition policies)." },
                    ],
                },
            ];

            const FRAMEWORKS = [
```

- [ ] **Step 2: Verify — Node structural check**

```js
import fs from 'node:fs';
import vm from 'node:vm';
import assert from 'node:assert/strict';

const html = fs.readFileSync('index.html', 'utf8');
const match = html.match(/<script id="app-logic">([\s\S]*?)<\/script>/);
const context = vm.createContext({});
vm.runInContext(match[1], context);

const spii = context.FRAMEWORKS[1];
assert.equal(spii.sections.length, 5, 'expected all 5 sections');
assert.equal(spii.sections[4].section, 'Researcher-centric');
assert.equal(spii.sections[4].principles.length, 2);
const ids = spii.sections[4].principles.map((p) => p.id);
assert.deepEqual(ids, ['spii-seamless-ux', 'spii-recognition-rewards']);
spii.sections[4].principles.forEach((p) => {
    assert.ok(p.criteria.compliant && p.criteria.progress && p.criteria['non-compliant'], `${p.id} missing criteria`);
    assert.ok(p.improvementAction, `${p.id} missing improvementAction`);
});

// Full SPII structure: 6+4+4+3+2 = 19 principles across 5 sections
const totalPrinciples = spii.sections.reduce((sum, s) => sum + s.principles.length, 0);
assert.equal(totalPrinciples, 19, 'expected 19 principles total');
const allIds = spii.sections.flatMap((s) => s.principles.map((p) => p.id));
assert.equal(new Set(allIds).size, 19, 'expected all 19 principle ids to be unique');

// initial_state() produces a correctly-shaped slice for spii
const state = context.initial_state();
assert.equal(Object.keys(state.frameworks.spii).length, 19);
assert.equal(state.frameworks.spii['spii-governance'].status, null);

// compute_summary works for spii
state.frameworks.spii['spii-governance'].status = 'compliant';
const summary = context.compute_summary(state, 'spii');
assert.equal(summary.totalCount, 19);
assert.equal(summary.compliant, 1);
assert.equal(summary.percentCompliant, 100);

// POSI still fully untouched
assert.equal(context.FRAMEWORKS[0].sections.length, 3);
assert.equal(Object.keys(state.frameworks.posi).length, 20);

console.log('Task 5 structural checks passed — SPII complete (19 principles, 5 sections).');
```

- [ ] **Step 3: JS syntax and CRLF checks**

Same commands as Task 1 Step 3.

- [ ] **Step 4: Manually verify in a browser**

Confirm SPII's sidebar shows all 5 sub-links (Openness, Autonomy, Sustainability, Interoperability, Researcher-centric); badge shows 5 rings (6+4+4+3+2 segments), title "SPII Compliance", correctly sized with no overlap/clipping of the name/date text; the "About SPII" accordion shows the background text and reference list; every principle shows its 3 status cards with criteria text and a "To improve:" note. Score a mix of SPII principles and confirm SPII's summary line updates independently of POSI's. Export JSON and confirm it contains both `frameworks.posi` (unchanged, 20 principles) and `frameworks.spii` (19 principles, correctly reflecting scored statuses). Reload the exported JSON and confirm both frameworks' data round-trips correctly. Confirm POSI's appearance and behavior remain completely unaffected throughout.

- [ ] **Step 5: Commit and push**

```bash
git add index.html
git commit -m "Add SPII Researcher-centric section, completing the framework"
git push
```

---

### Task 6: Update project documentation

**Files:**
- Modify: `CLAUDE.md`

**Interfaces:** none (documentation only).

- [ ] **Step 1: Update `CLAUDE.md`'s Architecture section**

Read the current `## Architecture` section of `CLAUDE.md`. Find the paragraph describing the data model (it should already describe `FRAMEWORKS` generically, per the prior machinery-generalization plan's Task 6 update). Add a short sentence noting that `FRAMEWORKS` now has two entries: `POSI` (stable) and `SPII` (a second, `draft: true` framework — 5 sections, 19 principles — added as a data-only change on top of the existing generic rendering/badge/summary machinery, with no code changes required beyond the data itself). Read the actual current paragraph text first and make a precise, natural-reading addition — don't guess at exact wording, match what's actually there.

- [ ] **Step 2: Commit and push**

```bash
git add CLAUDE.md
git commit -m "Document the SPII v0.0.1 framework addition"
git push
```
