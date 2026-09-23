# AGENTS.md

Instructions for AI agents working in this repo. Placeholder; conventions will be added as the first
skills land.

## Context

This is a **public** repo of agent skills for LeanLaw customers. Skills run against a firm's live
billing data through the LeanLaw MCP connector. `README.md` is the external catalog.

## Constraints

- No internal context in anything committed: no customer or employee names, no internal metrics, no
  links to internal tools.
- Refer to LeanLaw MCP tools by their logical name (`list_matters`), never a connector-specific prefix.
- Every write a skill makes (client, matter, fixed fee, time entry) is confirmed by the user first.
- A skill's directory name must equal its frontmatter `name`, and the name must be plain (no `/` or
  `:`).
- Adding or changing a skill means three edits: the skill folder, its row in `README.md`, and the
  `version` in `plugins/leanlaw/.claude-plugin/plugin.json`.

## Layout

```
.claude-plugin/marketplace.json         marketplace "leanlaw-agent-skills"
plugins/leanlaw/.claude-plugin/plugin.json
plugins/leanlaw/skills/<name>/SKILL.md
plugins/leanlaw/skills/<name>/references/   detail the skill reads on demand
```

## Validating a change

No build or CI. Run before committing:

```bash
python3 -c "import json;json.load(open('.claude-plugin/marketplace.json'))"
python3 -c "import json;json.load(open('plugins/leanlaw/.claude-plugin/plugin.json'))"
for d in plugins/leanlaw/skills/*/; do
  n=$(sed -n 's/^name: //p' "$d/SKILL.md" | head -1)
  [ "$(basename "$d")" = "$n" ] || echo "MISMATCH: $d vs name=$n"
done
```
