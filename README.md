# Gaspar API Skill

A Claude Code skill for working with the [Gaspar](https://gaspar.hidagama.com) email-campaign API.

Gaspar is the email-campaign platform at `gaspar.hidagama.com`. Its REST API is at `https://api.hidagama.com/api/gaspar/`. This skill teaches an AI agent (Claude Code, Claude Desktop, or any other LLM with file access) how to operate the API correctly — endpoints, auth, timezone handling, template syntax, watchdog behavior, frequency rules, and the gotchas that have cost real campaigns.

Companion to [`hidagama/gaspar-mcp`](https://github.com/hidagama/gaspar-mcp) — the MCP server gives the agent the tools; this skill gives it the operational know-how.

## What this is

A single self-contained `SKILL.md` that documents:

1. **API endpoints + auth** — every path, where to send requests, how to pass the bearer token
2. **Timezone conversion** — the #1 production mistake, and how to never make it
3. **10-point pre-flight checklist** — what to verify before pressing Schedule
4. **Campaign body shape** — exact request format for `POST /campaigns`
5. **Template syntax** — `{{var | fallback}}` works, every other syntax doesn't
6. **Watchdog behavior** — auto-pause math + the bounced→suppressed unpause workflow
7. **Click-tracking + attribution** — how clicks are counted, and cross-checking with UTMs / GA4
8. **Frequency rules** — cadence math and when to acknowledge over-sends
9. **Sequence staggering** — preventing the "3 emails at once" failure mode
10. **Completion report pipeline** — the ≥25-recipient gate

Plus a copy-paste command reference and a 5-mistakes-worth-memorizing summary at the end.

## Install

### As a user-level skill (available in every project)

```bash
mkdir -p ~/.claude/skills/gaspar
curl -sS https://raw.githubusercontent.com/hidagama/gaspar-skill/main/SKILL.md \
  -o ~/.claude/skills/gaspar/SKILL.md
```

### As a project-level skill (this project only)

```bash
cd your-project/
mkdir -p .claude/skills/gaspar
curl -sS https://raw.githubusercontent.com/hidagama/gaspar-skill/main/SKILL.md \
  -o .claude/skills/gaspar/SKILL.md
```

After install, the skill loads automatically when Claude Code sees a Gaspar-related task. You can also invoke it explicitly:

```
/gaspar
```

## Provisioning the API key

Get a Gaspar API key from `gaspar.hidagama.com` → **Settings** → **API**. Cache it as `GASPAR_API_KEY` in your environment (macOS Keychain, 1Password CLI, `.envrc`, GitHub Actions secret — any secrets manager except hard-coded files).

```bash
export GASPAR_API_KEY=...
curl -sS -H "Authorization: Bearer $GASPAR_API_KEY" https://api.hidagama.com/api/gaspar/campaigns
```

## Companion tools

- **[gaspar-mcp](https://github.com/hidagama/gaspar-mcp)** — Model Context Protocol server. Wraps the Gaspar REST API as MCP tools so an AI assistant can call them directly. Use this skill alongside the MCP for full coverage: the MCP gives the agent the verbs, this skill gives it the judgement.
- **Gaspar web app** — `https://gaspar.hidagama.com/app` for browser-based campaign management

## Contributing

Issues and PRs welcome. If you've hit a Gaspar gotcha not documented in `SKILL.md`, open an issue with the reproduction steps and we'll add it to the runbook.

## License

MIT
