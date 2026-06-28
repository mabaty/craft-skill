# craft-skill

A [Craft](https://craft.do) MCP skill for AI agents. Handles the block-based Craft API so documents come out right on the first pass — correct block structure, ordering, element syntax, and error recovery.

## What it does

- Maps every Craft element (headings, tasks, toggles, callouts, code blocks, dividers) to the right input format
- Handles shell-hostile content (backticks, quotes) via JSON mode
- Preserves block ordering with chained sibling anchors
- Includes backoff/retry strategy for Craft's intermittent 502/503/504 errors
- Pre-finalize checklist to verify documents before reporting done

## Install

Copy `SKILL.md` into your agent's skills directory. For OpenClaw:

```bash
cp SKILL.md ~/.openclaw/workspace/skills/craft/SKILL.md
```

For other agent frameworks (Hermes, Claude Code, etc.), place it wherever your agent loads skill/reference documents.

## Requirements

- A Craft MCP server connected to your agent
- Craft account (the MCP uses your space-scoped link — no separate API key)

## License

MIT — do whatever you want with it.