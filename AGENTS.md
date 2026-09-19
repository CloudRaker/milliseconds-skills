# milliseconds-skills

Public agent skills for decision-machine-1 (api.milliseconds.ai). Skills live in `skills/<name>/SKILL.md` (agentskills.io format); `skills/milliseconds/SKILL.md` is the umbrella. Plugin manifests: `.claude-plugin/`, `.codex-plugin/`, `.agents/plugins/`. `.mcp.json` points at the docs MCP server.

Rules when editing:

- Skills are hand-written, not generated. Frontmatter is `name` and `description` only.
- Every product fact, field name, limit and number comes from the live docs (https://docs.milliseconds.ai, append `.md`) or a real API capture. Never from memory. Re-capture examples when the API changes.
- Never mention other products or companies. The product is decision-machine-1; the company is milliseconds.ai.
- Writing style: active voice, short sentences, one or two ideas per sentence, no filler.
- Keep the house style consistent across skills: describe the case not the verdict, threshold per action, batch the cheap axis, constants in one file, golden set before tuning.
- Bump `version` in both plugin manifests on every content change.
