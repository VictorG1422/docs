# AI instructions

Apply these rules to all work in this documentation repository. Use
[skill.md](./skill.md) for the task workflow.

## Repository context

- This repository is the canonical source for the published MkDocs site.
- Markdown source pages live under `docs/`; site structure and navigation are
  configured in `mkdocs.yml`. `site/` is generated output, not source.
- Project documentation is grouped under `docs/<project>/`. The trading
  platform source is in `docs/trading-platform/`; its code repository links
  here for detailed architecture, setup, operations, testing, safety, and
  strategy guidance.

## Required practices

- Read the relevant source page, nearby related pages, and `mkdocs.yml` before
  editing. Preserve existing user changes and avoid unrelated rewrites.
- Keep claims accurate, scoped, and consistent with the corresponding code and
  project behavior. Do not invent configuration, commands, test results, or
  production-readiness guarantees.
- When a documented behavior or workflow changes, update the canonical page
  here and keep affected repository README instructions or links consistent.
- Add or update navigation only when introducing a page intended to appear in
  the published site. Preserve existing URLs and heading anchors when practical.
- Use concise, task-relevant context. Search narrowly, read only needed
  sections, and avoid duplicating information already maintained in another
  canonical page.
- Keep secrets, private account data, and unredacted operational output out of
  examples. Use clearly fictional values and safe placeholders.
- Do not stage or commit changes unless requested. Report validation warnings
  and checks that were not run; never describe an unverified build as passing.
