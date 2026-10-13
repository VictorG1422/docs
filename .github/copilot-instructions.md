# Repository instructions

Apply these concise, repository-wide rules to every task. Follow relevant
path-scoped instructions in `.github/instructions/`. When a task matches a
skill description in `.github/skills/`, load that skill; do not load unrelated
skills.

## Work efficiently and safely

- Inspect Git status and only the code, callers, tests, and documentation
  relevant to the request. Preserve existing user changes.
- Search narrowly, read only needed sections, summarize findings, and avoid
  repeating source or validation that already produced usable results.
- Make the smallest complete change that follows nearby patterns. Avoid
  unrelated refactors, dependencies, generated files, and speculative behavior.
- Keep implementation, public interfaces, configuration, tests, and user-facing
  documentation consistent across affected surfaces.
- Treat external input and persisted data as untrusted. Preserve validation,
  authentication, authorization, and least privilege. Never expose or commit
  credentials, private data, or secrets.
- Do not run destructive operations against databases, external services, or
  deployments without explicit authorization.
- Run the narrowest meaningful checks for the change. Report actual outcomes
  and anything not verified; never claim success without evidence.
- Ask before proceeding when a materially different or high-impact behavior is
  ambiguous. Otherwise use established project conventions and proceed.
