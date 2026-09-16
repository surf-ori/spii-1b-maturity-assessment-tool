# CLAUDE.md

Agent context for working on the `spii-1b-maturity-assessment-tool` repository.

## What this repo is

The feedback and issue tracker for SPII deliverable 1B, Maturity Assessment Tool. It holds no
application code: its content is community feedback issues, this tracker's own configuration,
and these context files. The tool itself lives at
[surf-ori.github.io/open-science-maturity](https://surf-ori.github.io/open-science-maturity/)
(a separate repository); this tracker is only for feedback on deliverable 1B, referenced from the
[spii-overview](https://github.com/surf-ori/spii-overview) page.

## Issue template

`.github/ISSUE_TEMPLATE/feedback.yml` collects name, organisation, role, and "representing
infrastructure" alongside the feedback itself, so the origin of each issue stays traceable. See
`AGENTS.md` for how to process incoming issues.

## Way of working

See the "Way of working" section on
[spii-overview](https://surf-ori.github.io/spii-overview/#way-of-working) for the full process: a
curator triages each issue and records the outcome (accepted, rejected, or already covered)
directly on the issue with a short rationale before closing it. Curators are not yet assigned for
this deliverable. Until one is, leave issues open for a human curator rather than closing them on
your own judgement.

## AI-generated content

If an agent drafts a closing comment or decision on an issue here, that comment is
public-interest text about a policy deliverable. Follow the `labeling-ai-generated-content` skill:
say plainly in the comment that it was drafted with AI assistance, mirroring the disclosure
already on the spii-overview page.

## Conventions

- English throughout.
- No code changes are expected in this repo beyond `.github/` configuration and these context
  files.
