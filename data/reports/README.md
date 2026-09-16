# Assessment reports

Community-submitted assessment reports, contributed via the "Save Report to GitHub Tracker"
option in the tool's Export menu: either as a new issue with the report file attached, or as a
pull request that adds the file directly to this folder.

Filenames follow `<infrastructure-slug>-<framework-id>-<date>.json`, matching what the tool
downloads, for example `openaire-posi-2026-09-16.json`.

A curator reviews each submission the same way as other feedback (see this repo's `AGENTS.md`
and the ["Way of working"](https://surf-ori.github.io/spii-overview/#way-of-working) section on
spii-overview) before merging it in. A report submitted as an issue attachment still needs a
follow-up commit or PR to land the file here; only files actually in this folder are part of the
public record.

Load any report back into the tool via the top bar's Import menu, or by opening
`index.html?report=data/reports/<file>.json`.
