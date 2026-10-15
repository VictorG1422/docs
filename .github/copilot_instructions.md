---
applyTo: "**"
---

# Project Copilot Instructions

Read the skill file for any domain listed below before generating output. Skill files are the authoritative source — follow them over general defaults.

---
## caveman — Terse Output

**Applies to:** `**`
**Skill:** `.agents/skills/caveman/SKILL.md`

Always active. Apply `full` compression level by default on every response. Drop articles, filler words, hedging, and pleasantries. Fragments are fine. Short synonyms over verbose ones. No tool-call narration. Standard acronyms are OK; do not invent abbreviations. Exit only when the user says "stop caveman" or "normal mode".

---

## ponytail — Minimal Code

**Applies to:** `**`
**Skill:** `.agents/skills/ponytail/SKILL.md`

Always active on coding tasks. Before writing any code, climb the YAGNI ladder: question whether the feature needs to exist, reuse what is already in the codebase, reach for stdlib before custom code, native platform before a dependency, one line before fifty. Default level is `full`. Exit only when the user says "stop ponytail" or "normal mode".

---