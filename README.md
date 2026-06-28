# craft-skill

A [Craft](https://craft.do) MCP skill for AI agents. Handles the block-based Craft API so documents come out right on the first pass — correct block structure, ordering, element syntax, and error recovery.

## What it does

- Maps every Craft element (headings, tasks, toggles, callouts, code blocks, dividers, whiteboards, collections, comments) to the right input format
- Handles shell-hostile content (backticks, quotes, emoji) via JSON mode
- Preserves block ordering with chained sibling anchors
- Documents the v2 unified tool model (`craft_write` / `craft_read` / `blocks_revert`)
- Covers collections, whiteboards, comments, page styling, and search
- Includes backoff/retry strategy for Craft's intermittent 502/503/504 errors
- Pre-finalize checklist to verify documents before reporting done

## MCP v2

This skill targets **Craft MCP v2** (released June 2026). MCP v2 collapsed 32+ individual tool endpoints into 3 unified tools (`craft_write`, `craft_read`, `blocks_revert`), reducing interface token count by 85.9%.

**Still on MCP v1?** Use the [v1.0.0 tag](https://github.com/mabaty/craft-skill/releases/tag/v1.0.0) — the v1 skill documents the original per-command interface (`blocks add`, `documents create`, etc.).

## Install

Copy `SKILL.md` into your agent's skills directory. For OpenClaw:

```bash
cp SKILL.md ~/.openclaw/workspace/skills/craft/SKILL.md
```

For other agent frameworks (Hermes, Claude Code, etc.), place it wherever your agent loads skill/reference documents.

## Requirements

- A Craft MCP v2 server connected to your agent
- Craft account (the MCP uses your space-scoped link — no separate API key)

## License

MIT — do whatever you want with it.