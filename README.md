# milliseconds.ai agent skills

Agent skills for [decision-machine-1](https://docs.milliseconds.ai), the decisions API at `api.milliseconds.ai`. It turns text into typed decisions in about 0.4 s: labels, probabilities and spans, never prose. The skills follow the [Agent Skills](https://agentskills.io) format and ship as a plugin for Claude Code and OpenAI Codex.

Read [`skills/milliseconds/SKILL.md`](skills/milliseconds/SKILL.md) first. It also points at the official SDKs and the `dm1` command line, which most projects should use instead of raw HTTP. It is the umbrella. The rest go one level deeper per capability.

## Install

Claude Code:

```bash
claude plugin marketplace add CloudRaker/milliseconds-skills
claude plugin install milliseconds@milliseconds-skills
```

Other agents, via skills.sh:

```bash
npx skills add CloudRaker/milliseconds-skills
```

Or copy a prompt into your agent:

```text
Install the milliseconds.ai skills. In Claude Code run `claude plugin marketplace add CloudRaker/milliseconds-skills` then `claude plugin install milliseconds@milliseconds-skills`. In another agent run `npx skills add CloudRaker/milliseconds-skills`. You can also read the umbrella skill directly at https://raw.githubusercontent.com/CloudRaker/milliseconds-skills/main/skills/milliseconds/SKILL.md. Then use the milliseconds skill when working on this project.
```

The plugin also registers the docs MCP server (`https://docs.milliseconds.ai/_mcp/server`) so the agent can search the live documentation.

Get an API key at [console.milliseconds.ai](https://console.milliseconds.ai/signup). The free plan includes 125 million input tokens per month. Export it as `MS_API_KEY`.

## Use

Name the skill in your prompt, or invoke `/milliseconds:milliseconds` in Claude Code.

```text
Using the milliseconds skill, explore this project and find places where a typed
decision (classify, yes-no, rate, extract) could replace fragile parsing or an
LLM prompt-and-parse step.
```

```text
Using the milliseconds skill and the key in MS_API_KEY, route inbound support
tickets to billing, shipping or account, with a review queue for uncertain cases.
Put the labels and thresholds in one file.
```

```text
Using the milliseconds skill, build a golden set from the last 100 tickets in
tickets.jsonl, score it, and propose thresholds per action.
```

## Catalog

Every capability skill also shows the same call in the TypeScript SDK, the Python SDK and the `dm1` command line.

| Skill | What it covers |
| --- | --- |
| [`milliseconds`](skills/milliseconds) | Umbrella: auth, choosing a capability, writing labels that describe the case, reading probability and confidence, thresholds per action, batching, errors, patterns, working method |
| [`milliseconds-yes-no`](skills/milliseconds-yes-no) | Statements and `when_true` / `when_false` hints, batched flags, guardrails, RAG filtering |
| [`milliseconds-classify`](skills/milliseconds-classify) | Described labels, `probability` with `confidence`, intent routing, cascade, and `classify-tree` |
| [`milliseconds-rate`](skills/milliseconds-rate) | Ordered scales, routing on `score` not `level`, composite scoring |
| [`milliseconds-answer`](skills/milliseconds-answer) | One span per question with offsets, `null` handling, role-not-type questions |
| [`milliseconds-extract`](skills/milliseconds-extract) | JSON Schema to typed object, the schema support matrix, treat results as a draft |
| [`milliseconds-entities`](skills/milliseconds-entities) | Every span per type with offsets, redaction loop, PII |
| [`milliseconds-verify`](skills/milliseconds-verify) | Check a held value against the text, read `found[]`, the matching rule |
| [`milliseconds-openai`](skills/milliseconds-openai) | OpenAI client compatibility: `json_schema` and `tools`, what the facade discards, when to go native |
| [`milliseconds-evaluate`](skills/milliseconds-evaluate) | Golden sets, threshold sweeps, monitoring, publishing numbers |

## The house style these skills teach

1. **Describe the case, never the verdict.** `yes`, `no`, `A`, `1` mean nothing to the model. Every label, statement, level, question and field carries a description of the case it names.
2. **Choose the capability from the shape of the answer.** Do you supply the possible answers, or does the text? Do you need offsets?
3. **Thresholds belong to the action, not the model.** Act, confirm, escalate bands per action, raised with the blast radius. Never inherit the built-in 0.5.
4. **Read both numbers.** `probability` says how strong the winner is; `confidence` says how separated the alternatives were.
5. **Batch the cheap axis.** Statements and questions ride in one inference call; each text costs one.
6. **Constants in one place.** Labels, scales, schemas and thresholds live in one reviewable file.
7. **Measure before you tune.** A golden set of 50 to 200 real examples, scored once, thresholds read off the table.

## Docs

- Guides and API reference: [docs.milliseconds.ai](https://docs.milliseconds.ai)
- Markdown for agents: append `.md` to any docs URL, or start at [docs.milliseconds.ai/llms.txt](https://docs.milliseconds.ai/llms.txt)

## License

[MIT](LICENSE).
