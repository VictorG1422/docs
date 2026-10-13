# Documentation engineering skill

Use this playbook for writing, editing, and reviewing pages in this repository.
For repository-wide constraints, see [instruction.md](./instruction.md).

## 1. Find the source of truth

1. Identify the page and project area that own the requested topic.
2. Read that page, only the directly related pages, and the relevant navigation
   entries in `mkdocs.yml`.
3. For behavior or code instructions, verify against the linked code repository
   or its existing tests when available. Distinguish documented facts from
   examples and known limitations.

Keep context focused: search for the topic, inspect only relevant sections, and
summarize findings instead of copying large passages into the response.

## 2. Make the documentation change

- Edit the existing canonical page rather than creating a duplicate.
- Match neighboring Markdown, MkDocs Material, table, admonition, and code-block
  conventions.
- Prefer direct language, actionable steps, and short examples. State
  prerequisites, defaults, units, date/time assumptions, and failure cases
  wherever readers need them.
- Mark illustrative values as examples. Never include credentials, private
  account details, real tokens, or unverified outcomes.
- Update `mkdocs.yml` navigation only for a new page that should be published.
  Preserve existing page paths and anchors where possible.
- If project behavior or commands changed, update cross-references or user
  instructions in the code repository when relevant.

## 3. Validate and report

Run the smallest useful documentation check from the repository root:

```powershell
.\.venv\Scripts\python.exe -m mkdocs build
```

Review warnings for broken links or navigation, especially when editing links
or adding pages. Use `--strict` when the changed pages are expected to be
warning-free; identify pre-existing warnings separately instead of silently
changing unrelated content. Do not edit generated `site/` output.

Inspect the final diff and Git status. Report pages changed, build/check results,
known warnings, and any remaining documentation gaps.
