# SPII v0.0.1 assessment framework — design

Date: 2026-08-27

## Goal

Add a second `FRAMEWORKS` entry — SPII v0.0.1 — built entirely on the
generalized machinery from the prior plan
(`docs/superpowers/specs/2026-08-26-multi-framework-machinery-design.md`,
merged to `main`). This is primarily a data-entry task: the rendering,
badge, sidebar nav, summary, and draft-label mechanisms all already work
generically. This spec's job is to nail down SPII's actual content —
5 principles, 19 scoreable subprinciples, each with a description, a
3-level rubric (compliant / making progress / not compliant), and a
suggested improvement action.

## Out of scope

- Any change to the rendering machinery — if implementation surfaces a
  need for one, that's a signal the prior plan's generalization was
  incomplete and should be flagged, not patched ad hoc here.
- POSI's content — untouched.
- Anything beyond what the user's principles/subprinciples/actions list
  and the three reference sources (PLOS "knowledge stack", Digital
  Autonomy Assessment Tool, Australian Government evaluation maturity
  matrix) support. Where the source material didn't specify something
  (e.g. no subprinciple was given for "5.2"), it's called out explicitly
  below rather than silently invented.

## Framework metadata

```js
{
    id: "spii",
    title: "SPII v0.0.1 Assessment",
    shortName: "SPII",
    description: "A community-drafted framework (v0.0.1, June 2026 consultation) covering openness, autonomy, sustainability, interoperability, and researcher-centricity for open science infrastructure.",
    aboutHtml: "...", // see Content, below — background + reference list
    draft: true,
    sections: [ /* 5 sections, see Content below */ ],
}
```

`draft: true` is the whole point of the mechanism built in the prior plan
— SPII shows a "Draft – work in progress" label next to its title and in
the sidebar nav; POSI (no `draft` field) shows none. `description`/
`aboutHtml` follow the exact same shape POSI already uses.

## Structure

Five principles (sections), 19 scoreable subprinciples (principles) total
— confirmed with the user:

| Section | Subprinciples | Count |
|---|---|---|
| Openness | Governance, Operations, Provenance, Economical, Strategical (all under "1.1 Transparency", scored individually), Accessibility (1.2) | 6 |
| Autonomy | Stakeholder governed/community-led, Discipline-specific, Digital sovereignty, Responsibility | 4 |
| Sustainability | Financial sustainability, Human capacity sustainability, Legal and institutional compliance, Enduring availability | 4 |
| Interoperability | Technical interoperability, Organisational & social interoperability, Legal interoperability | 3 |
| Researcher-centric | Seamless user-experience, Recognition and rewards | 2 |

"Recognition and rewards" (5.2) was not explicitly listed as a
subprinciple in the source material — only implied by action item "52.
Recognition and rewards framework" appearing alongside "51. User-driven
design" under Researcher-centric. Confirmed with the user: treat it as
its own scoreable subprinciple (5.2), not folded into 5.1.

Each section's `sourceUrl` points to the most topically relevant
reference from the source material's own footnotes (not a single
repeated link, since SPII has no dedicated external site the way POSI
does):
- Openness, Autonomy, Sustainability → NPOS2030 Ambition Document
  (`https://zenodo.org/record/7433767`) — the principle statements for
  Openness and Autonomy were footnoted to this reference in the source
  material; Sustainability has no more specific footnote, so it defaults
  to the same overarching source.
- Interoperability → Seven Guiding Principles for Open Research
  Information (`https://zenodo.org/records/6074944`) — footnoted under
  "40a. Open standards" in the source material.
- Researcher-centric → The TRUST Principles for digital repositories
  (`https://doi.org/10.1038/s41597-020-0486-7`, Lin et al., Sci. Data 7,
  144 (2020) — verified via Crossref) — footnoted under "5.1 Seamless
  user-experience" in the source material.

## Content

Full text for all 19 subprinciples. Each entry: id, title, description
text, 3-level criteria, improvement action.

### Openness (`sourceUrl`: NPOS2030)

**1. `spii-governance` — Governance**
- Text: "Transparency of how the infrastructure is governed — decision-making bodies, policies, and how the community can see and influence them."
- Compliant: "Governance structures, decision-making processes, and policies are fully documented and publicly available."
- Making progress: "Some governance information is public, but key decisions or policies are not consistently documented or disclosed."
- Not compliant: "Governance is opaque; decision-making processes are not documented or accessible to the community."
- Improvement action: "Publish governance charters, meeting minutes/decisions, and policy documents openly."

**2. `spii-operations` — Operations**
- Text: "Transparency of day-to-day operations — how the infrastructure runs, its procedures, and operational metrics."
- Compliant: "Operational procedures, SLAs, incident reports, and key operational metrics are openly published and kept up to date."
- Making progress: "Some operational information is shared, but not consistently or not comprehensively."
- Not compliant: "Operational practices are undocumented or not shared with users/community."
- Improvement action: "Publish operating procedures and regularly updated status/incident reports."

**3. `spii-provenance` — Provenance**
- Text: "Ability to trace the origin, custody, and changes of data, content, software, and decisions within the infrastructure."
- Compliant: "Full provenance (origin, versioning, custody chain) is tracked and exposed for data, software, and content."
- Making progress: "Provenance is tracked internally but not consistently exposed or documented for users."
- Not compliant: "Provenance information is not tracked or is unavailable."
- Improvement action: "Implement provenance tracking (e.g. versioning, changelogs, PID-linked history)."

**4. `spii-economical` — Economical**
- Text: "Transparency about the financial model — costs, funding sources, pricing, and how money flows through the infrastructure."
- Compliant: "Detailed financials, funding sources, and cost/pricing models are openly published."
- Making progress: "High-level financial information is available, but details (e.g. cost breakdowns, funding sources) are limited or inconsistent."
- Not compliant: "Financial information is not disclosed."
- Improvement action: "Publish an annual financial transparency report."

**5. `spii-strategical` — Strategical**
- Text: "Transparency about strategic direction — roadmaps, priorities, and long-term plans."
- Compliant: "Strategic/product roadmaps and priorities are publicly available and regularly updated."
- Making progress: "A roadmap exists but is not public, not regularly updated, or lacks detail."
- Not compliant: "No strategic roadmap exists or is shared."
- Improvement action: "Publish and maintain a public strategic roadmap."

**6. `spii-accessibility` — Accessibility**
- Text: "The infrastructure's software, data, APIs, and outputs are openly accessible, using open standards and licences, secured by design, and reachable by all relevant users including citizen scientists."
- Compliant: "Software is open-source, data/metadata/APIs are open, licences are clear and open, the infrastructure runs on a public/open infrastructure stack, security is built-in by design, and identity/access management supports broad access including citizen scientists."
- Making progress: "Some components are open (e.g. open data or open-source software) but others remain closed, proprietary, or access-restricted."
- Not compliant: "Software, data, APIs, or access are closed/proprietary with no clear licensing or public access path."
- Improvement action: "Work through the open-X checklist — open-source software, open data, open APIs, open repositories, open metadata, open licences, public infrastructure stack, security-by-design, and inclusive IAM."

### Autonomy (`sourceUrl`: NPOS2030)

**7. `spii-stakeholder-governed` — Stakeholder governed/community-led**
- Text: "The infrastructure is governed by its stakeholder community rather than a single external party."
- Compliant: "A representative governing body drawn from the stakeholder community makes key decisions, with documented community input mechanisms."
- Making progress: "Some community input is solicited, but governance is largely controlled by a single organisation or vendor."
- Not compliant: "Governance is entirely external/vendor-controlled with no community input."
- Improvement action: "Establish a stakeholder governing board and open, non-discriminatory participation."

**8. `spii-discipline-specific` — Discipline-specific**
- Text: "The infrastructure is fit for the specific needs, norms, and workflows of its research discipline(s)."
- Compliant: "The infrastructure is explicitly designed and regularly reviewed against the needs of its target discipline(s), with documented fit-for-purpose criteria."
- Making progress: "Some discipline-specific needs are addressed, but the infrastructure is largely generic and not tailored."
- Not compliant: "No consideration of discipline-specific needs; infrastructure is a one-size-fits-all solution regardless of context."
- Improvement action: "Run periodic fit-for-purpose reviews with discipline representatives."

**9. `spii-digital-sovereignty` — Digital sovereignty**
- Text: "The infrastructure safeguards digital and academic sovereignty — control over data, infrastructure, and decisions stays with the academic community rather than being ceded to external commercial or geopolitical interests."
- Compliant: "Data, infrastructure control, and critical decisions remain within community/academic governance; dependency on external vendors is actively managed with exit strategies."
- Making progress: "Some dependency on external vendors/platforms exists, with partial mitigation (e.g. contractual safeguards) but no full exit strategy."
- Not compliant: "Critical infrastructure or data is fully dependent on external vendors/platforms with no contractual safeguards or exit plan."
- Improvement action: "Adopt value-driven procurement, balance agency against costs & risks, and put contractual clauses and exit strategies in place — consider using the Digital Autonomy Assessment Framework for a deeper dive."

**10. `spii-responsibility` — Responsibility**
- Text: "Clear ownership of responsibilities for the infrastructure's operation, decisions, and consequences."
- Compliant: "Responsibilities are clearly assigned, documented, and regularly reviewed against the infrastructure's purpose and community value."
- Making progress: "Some responsibilities are assigned but not comprehensively documented or reviewed."
- Not compliant: "Responsibilities are unclear or undocumented."
- Improvement action: "Document a responsibility matrix (who is accountable for what) and schedule regular reviews of purpose and community value."

### Sustainability (`sourceUrl`: NPOS2030)

**11. `spii-financial-sustainability` — Financial sustainability**
- Text: "The infrastructure has a sustainable financial model beyond short-term grants."
- Compliant: "Structural/recurrent funding is secured (e.g. memberships, recurrent grants), with a mission-consistent revenue model and clear cost-of-ownership understanding."
- Making progress: "Funding is time-limited (e.g. a one-off grant) with plans underway to secure recurrent funding."
- Not compliant: "No funding secured for the foreseeable future."
- Improvement action: "Develop mission-consistent revenue generation and a total-cost-of-ownership analysis; consider make-or-buy guidelines."

**12. `spii-human-capacity` — Human capacity sustainability**
- Text: "Sufficient, resilient staffing to operate and maintain the infrastructure."
- Compliant: "A dedicated, adequately resourced team maintains the infrastructure, with succession/continuity planning in place."
- Making progress: "The infrastructure is maintained but relies heavily on one or few individuals with no succession plan."
- Not compliant: "No one is actively maintaining the infrastructure."
- Improvement action: "Build a resourcing/succession plan and invest in sustained maintenance capacity."

**13. `spii-legal-compliance` — Legal and institutional compliance**
- Text: "The infrastructure meets applicable legal and institutional requirements."
- Compliant: "Legal/regulatory compliance (e.g. data protection, IP, institutional policy) is actively managed, documented, and audited or certified."
- Making progress: "Compliance is partially addressed, with known gaps being worked on."
- Not compliant: "Compliance status is unknown or has major gaps."
- Improvement action: "Conduct a compliance audit and address gaps against relevant legal/institutional requirements."

**14. `spii-enduring-availability` — Enduring availability**
- Text: "The infrastructure and its outputs remain available over the long term, with plans for what happens if it winds down."
- Compliant: "A documented end-of-life/succession policy exists, content is deposited with trusted third-party archives, and continuity of the public stack is supported."
- Making progress: "Some continuity measures exist (e.g. backups) but no formal end-of-life policy or archival deposit."
- Not compliant: "No continuity plan; data/content would be lost if the infrastructure stopped."
- Improvement action: "Publish an end-of-life policy (a 'living will') and deposit outputs with a trusted third-party digital archive."

### Interoperability (`sourceUrl`: Seven Guiding Principles for Open Research Information)

**15. `spii-technical-interoperability` — Technical interoperability**
- Text: "Systems, data and services can be connected, migrated or integrated using open, syntactic standards."
- Compliant: "The infrastructure uses open standards, persistent identifiers (PIDs), and maintains active connections to relevant international infrastructures."
- Making progress: "Some open standards/PIDs are used, but integration with international infrastructures is limited or ad hoc."
- Not compliant: "Proprietary formats/protocols are used with no PIDs and no external connections."
- Improvement action: "Adopt open standards and PIDs, and establish connections to relevant international infrastructures."

**16. `spii-organisational-interoperability` — Organisational & social interoperability**
- Text: "Shared semantics — vocabularies and ontologies — enable the infrastructure's data and processes to be understood consistently across organisations and communities."
- Compliant: "Community-recognised vocabularies/ontologies are used consistently and documented."
- Making progress: "Some shared vocabularies are used, but inconsistently or without documentation."
- Not compliant: "No shared vocabularies/ontologies are used; semantics are ad hoc or undocumented."
- Improvement action: "Adopt and document relevant community vocabularies/ontologies."

**17. `spii-legal-interoperability` — Legal interoperability**
- Text: "Clear, compatible terms of use enable data, software, and content to be legally reused and combined across systems."
- Compliant: "Terms of use are clear, machine-readable where possible, and compatible with common open licences used by peer infrastructures."
- Making progress: "Terms of use exist but are unclear, inconsistent, or not machine-readable."
- Not compliant: "No clear terms of use, or terms actively conflict with reuse/interoperability."
- Improvement action: "Publish clear, standard, machine-readable terms of use aligned with common open licences."

### Researcher-centric (`sourceUrl`: TRUST Principles)

**18. `spii-seamless-ux` — Seamless user-experience**
- Text: "The infrastructure is designed around researchers' actual workflows and needs."
- Compliant: "User-driven design practices (e.g. user research, usability testing) are embedded in the infrastructure's development process, and adoption/satisfaction is actively monitored."
- Making progress: "Some user feedback is gathered, but design is not systematically user-driven."
- Not compliant: "No user research or feedback process exists; design decisions are made without researcher input."
- Improvement action: "Establish ongoing user-driven design practices (user research, usability testing, feedback loops)."

**19. `spii-recognition-rewards` — Recognition and rewards**
- Text: "Contributions made via the infrastructure (e.g. data, code, review, curation) are recognised and rewarded within academic evaluation and career systems."
- Compliant: "The infrastructure actively supports recognition and rewards for contributions (e.g. integrates with CRediT, ORCID, institutional recognition/reward frameworks)."
- Making progress: "Some recognition mechanisms exist (e.g. basic attribution) but are not integrated with formal reward/evaluation systems."
- Not compliant: "No mechanism exists to recognise or reward contributions made via the infrastructure."
- Improvement action: "Integrate with recognition and rewards frameworks (e.g. CRediT, institutional recognition policies)."

### `aboutHtml` (framework-level "About SPII" accordion)

Approved drafts, adapted to this framework's format:

> SPII (version 0.0.1) is a draft assessment framework developed through the SPII project's community consultation, concluding in June 2026. It reflects five principles — Openness, Autonomy, Sustainability, Interoperability, and Researcher-centric — each broken into subprinciples with concrete actions, enablers, and practices infrastructures can adopt.
>
> The framework's underlying values — quality and integrity, collective benefit, and equity, fairness, diversity and inclusion — draw on UNESCO's Open Science Recommendation and the Dutch National Plan Open Science (NPOS). This is a v0.0.1 draft: principles and criteria may change as community feedback continues.
>
> References: NPOS, *Open Science 2030 in the Netherlands: NPOS2030 Ambition Document and Rolling Agenda* (2022), zenodo.org/record/7433767. UNESCO, *Recommendation on Open Science*, unesdoc.unesco.org. Lin, D. et al., *The TRUST Principles for digital repositories*, Sci. Data 7, 144 (2020), doi.org/10.1038/s41597-020-0486-7. *The Principles of Open Scholarly Infrastructure* (v2.0, 2025), openscholarlyinfrastructure.org. Bijsterbosch, M. et al., *Seven Guiding Principles for Open Research Information* (2022), zenodo.org/records/6074944. Universities of the Netherlands, *A new open access strategy for the Netherlands* (2026), doi.org/10.5281/zenodo.19548009.

## File-level impact

- `index.html` only — a new `SPII` data constant (mirroring `POSI`'s
  shape: array of `{section, sourceUrl, principles}`) in
  `<script id="app-logic">`, and a second entry appended to the
  `FRAMEWORKS` array referencing it. No rendering/JS logic changes
  expected — if implementation reveals one is needed, that's a signal to
  stop and reconsider rather than patch around it.
- `CLAUDE.md` — a one-line update noting SPII is now a second, draft
  framework entry (matching how the machinery-generalization plan's
  Task 6 documented the mechanism generically).

## Testing

No committed test suite (project convention, unchanged). Manual
verification in a browser after implementation:

1. Sidebar shows "SPII v0.0.1 Assessment (draft)" as a second expandable
   group, with 5 sub-links (Openness, Autonomy, Sustainability,
   Interoperability, Researcher-centric).
2. SPII's `<h2>` shows the "Draft – work in progress" label; POSI's does
   not.
3. All 19 SPII subprinciples render with their description, 3 status
   cards (each showing its criteria rubric text, per the machinery from
   the prior plan), and a "To improve:" note.
4. Clicking SPII's nav link (or any of its sub-links) switches the
   assessment badge to "SPII Compliance" with 5 concentric rings
   (6/4/4/3/2 segments) sized correctly (this exercises the ring-scaling
   fix from the prior plan's final review — confirm the rings don't
   overlap the badge's name/date text).
5. Scoring SPII principles updates SPII's own summary line independently
   of POSI's (confirms the framework-scoped summary fix).
6. JSON export/reload round-trips both frameworks' data correctly
   (`frameworks.posi` and `frameworks.spii` both present and correct).
7. POSI's appearance/behavior is completely unaffected by SPII's
   addition — the primary regression check.
