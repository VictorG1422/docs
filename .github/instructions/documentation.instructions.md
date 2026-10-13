---
applyTo: "docs/**/*.md,mkdocs.yml"
---

# Documentation rules

- Treat pages under `docs/` as the canonical published content. Read the
  relevant page and directly related pages before editing; avoid duplicate
  sources of truth.
- Verify code behavior, commands, and configuration against the owning
  repository when available. Label examples and known limitations accurately;
  do not invent results or production-readiness claims.
- Keep prose concise and actionable. Preserve page URLs and heading anchors
  where practical; update `mkdocs.yml` navigation only for new published pages.
- Keep credentials, private account data, and real secrets out of examples.
- When project behavior changes, update the corresponding code repository's
  user-facing instructions or links where relevant.
- Validate with `.\.venv\Scripts\python.exe -m mkdocs build` from the docs
  repository root. Review warnings and do not edit generated `site/` output.
