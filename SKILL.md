---
name: craft
description: Create and format Craft documents reliably via the Craft MCP (mcp.craft.do) block API. Triggers on any Craft document or note creation, editing, or formatting — including when building project docs, meeting notes, specs, or any structured content in Craft. Ensures correct block structure, ordering, and element syntax on the first pass.
---

# craft-doc — Reliable Craft Document Creation

The Craft MCP is a **block-based API**, not a markdown-string API. Documents are
sequences of typed blocks identified by UUIDs. Understanding this model prevents
formatting failures, stray blocks, and ordering bugs.

## The Block Model

- Every paragraph, heading, list item, code block, divider, and callout is a
  **separate block** with its own ID.
- Blocks are added via `blocks add` (markdown or JSON) and modified via
  `blocks update`, `blocks delete`, `blocks move`.
- Read back with `blocks get <rootBlockId> --format markdown` (human) or
  `--format json` (inspect types, listStyle, taskInfo, decorations).

### The Newline Rule

In `--markdown` arguments:

| Input | Result |
|---|---|
| `\n\n` (double newline) | **New block** — each `\n\n`-separated segment becomes its own block |
| `\n` (single newline) | **Literal characters** `\` and `n` stored in the block text |
| Actual newline (shell `$'...\n...'`) | Soft line break within the same block |

**Single `\n` never creates a new block or a soft break.** It produces visible
`\n` text. This is the #1 cause of broken documents.

## Default Workflow: Whole-Document Creation

For new documents, use **one `blocks add` call with `\n\n`-separated markdown**
appended to the page root. This creates all blocks in one round trip with
correct ordering.

```
1. documents create → get rootBlockId
2. blocks add --id <rootBlockId> --markdown "all content\n\nseparated\n\nby\n\ndouble\n\nnewlines"
3. blocks get <rootBlockId> --format markdown → verify
```

### When to Use `--json` Instead of `--markdown`

The `--markdown` mode fails when content contains backticks, nested quotes, or
other shell-hostile characters. The shell mangles the string before Craft sees it.

**Use `--json` arrays when:**
- Content contains backticks (code references like `gh`, `/mnt/reference/`)
- Content contains quotes or special characters that break shell quoting
- You need explicit type control (tasks with `taskInfo`, code blocks, dividers)
- Building complex documents with mixed element types

`--markdown` is fine for simple prose, headings, and basic lists. When in doubt,
use JSON — it's always correct.

### When Incremental Adds Are Needed

Use individual `blocks add` calls only when:
- Appending to an existing document
- Inserting at a specific position
- Adding code blocks (require JSON mode)

### Sibling Ordering Rule

When adding blocks sequentially after a header/anchor:

- ❌ **Same anchor**: Adding multiple siblings `after` the same block ID produces
  **reverse order**. Each new block inserts immediately after the anchor,
  pushing previous additions down.
- ✅ **Chain anchors**: Each new block uses the **previously added block's ID**
  as its `--siblingId`. This preserves insertion order.

```
# WRONG (reversed):
blocks add --siblingId HEADER --markdown "A" --position after  → id_A
blocks add --siblingId HEADER --markdown "B" --position after  → id_B
# Result: HEADER → B → A

# CORRECT (chained):
blocks add --siblingId HEADER --markdown "A" --position after  → id_A
blocks add --siblingId id_A   --markdown "B" --position after  → id_B
# Result: HEADER → A → B
```

**Exception:** JSON array mode (`--json '[{...},{...}]'`) always preserves the
array order regardless of anchor.

## Element → Input Mapping

| Element | Markdown Input | JSON Input | Notes |
|---|---|---|---|
| Heading 1–4 | `# H1` … `#### H4` | `{"textStyle":"h1",...}` | `\n\n` before/after |
| Paragraph | Plain text | `{"type":"text","markdown":"..."}` | |
| **Bold** | `**text**` | Inline within markdown | |
| *Italic* | `*text*` | Inline within markdown | |
| ~~Strikethrough~~ | `~~text~~` or `~text~` | Inline | Craft normalizes `~~` to `~` |
| `Inline code` | `` `code` `` | Inline | |
| Bullet list | `- Item` | `{"listStyle":"bullet","markdown":"- Item"}` | One item = one block |
| Numbered list | `1. Item` | `{"listStyle":"numbered","markdown":"1. Item"}` | Craft renumbers on display |
| **Task (open)** | `- [ ] Task text` | `{"listStyle":"task","taskInfo":{"state":"todo"}}` | ✅ Preferred syntax |
| **Task (done)** | `- [x] Task text` | `{"listStyle":"task","taskInfo":{"state":"done"}}` | ✅ Preferred syntax |
| Toggle | `+ Toggle title` | `{"listStyle":"toggle"}` | Content = subsequent indented blocks |
| Blockquote | `> Quoted text` | `{"decorations":["quote"]}` | |
| Callout | `<callout>Text</callout>` | `{"decorations":["callout"]}` | |
| Highlight | `==Highlighted text==` | Inline | Renders yellow by default |
| Divider | `---` or `***` | `{"type":"line"}` | |
| Link | `[text](url)` | Inline | |
| **Code block** | ❌ Fenced ``` does NOT work | `{"type":"code","rawCode":"..."}` | **JSON only** |

### Code Block Detail

Fenced code blocks (```) are **not supported** in `--markdown` mode. The `\n`
characters become literal text. Use JSON mode:

```bash
blocks add --id <pageId> --json '{"type":"code","rawCode":"print(\"hello\")\nprint(\"world\")","codeLanguage":"python"}'
```

Within `rawCode`, `\n` produces actual newlines in the code block.

### Task/Checkbox Syntax

| Want | Use | Avoid |
|---|---|---|
| Open task | `- [ ] text` | `+ text` (this is a toggle, not a task) |
| Done task | `- [x] text` | `- [done] text` (creates a bullet) |
| Toggle section | `+ text` | `- [ ] text` (this is a task checkbox) |

### JSON Array for Bulk Blocks

Pass an array to create multiple blocks in one call with exact type control:

```bash
blocks add --id <pageId> --json '[
  {"type":"text","textStyle":"h2","markdown":"## Section"},
  {"type":"text","listStyle":"bullet","markdown":"- Item one"},
  {"type":"code","rawCode":"SELECT * FROM users","codeLanguage":"sql"},
  {"type":"text","listStyle":"task","markdown":"- [x] Completed","taskInfo":{"state":"done"}}
]'
```

Array order is always preserved.

## Rewriting Existing Documents

### Full Rewrite: Delete and Recreate

When overhauling an entire document, **delete the document and create a fresh one**
rather than deleting blocks individually:

```bash
# Fast: one call
documents delete --document <rootBlockId>
documents create --title "Same Title" --folder "<folderId>"

# Slow: N calls for N blocks (avoid)
blocks delete --id <block1>; blocks delete --id <block2>; ...
```

Block-by-block deletion is only worth it when preserving some existing content.

### Renaming a Document

`documents move` changes folder/location only — it **cannot rename**. To change
the title, update the root page block:

```bash
blocks update --id <rootBlockId> --markdown "New Document Title"
```

The root block ID is the same as the document ID returned by `documents create`.

## Handling Transient Upstream Errors (502 / 503 / 504)

Craft's MCP endpoint sits behind Cloudflare and intermittently returns
`502 Bad Gateway` (`origin_bad_gateway`), `503`, or `504`. These are **almost
always transient** — Cloudflare itself flags them `retryable: true` with a
`retry_after` hint. Do **not** treat them as fatal, and do **not** loop in
tight retries that burn tool budget and can wedge the session.

### Recognize the error

A Craft 502 surfaces as a tool error containing JSON like:

```
"status":502, "error_name":"origin_bad_gateway",
"retryable":true, "retry_after":60
```

Any `5xx` from `craft__craft_write` or `craft__craft_read` falls in this
category. `4xx` errors (400 bad request, 404 not found, 422 validation) do
**not** — those are real failures in the command and retrying won't help.

### Backoff policy

1. **First failure**: wait the server-provided `retry_after` (default **60s**
   if absent), then retry the **exact same** command once.
2. **Second failure**: wait **120s**, retry once more.
3. **Third failure**: stop. Report the outage to the user with the last error
   payload and ask whether to keep waiting or pause the work. Do not silently
   keep hammering.

Use `exec sleep <seconds>` between retries — never a tight loop of repeated
tool calls.

### Session-safety rules

- **Never abort mid-retry.** Interrupting a sleeping retry has wedged sessions
  into a `delivery-mirror` death-spiral that forces a transcript reset and
  drops the model override back to the agent default. If the user wants to
  cancel, finish the in-flight `sleep` or let the next retry fail cleanly,
  then stop.
- **Save progress before retrying.** If you've created the document and are
  partway through `blocks add` calls, note the `rootBlockId` and the
  last-successfully-added block ID in your reply before the sleep. That way a
  reset session can resume instead of restarting from scratch.
- **Prefer fewer, larger writes.** A single `blocks add --json '[...]'` array
  call has one chance to 502; ten sequential `blocks add` calls have ten
  chances. When upstream is flaky, batch aggressively.
- **Read-back is also a write target.** `blocks get` can 502 too. Treat its
  failures the same way — back off, don't assume the document is broken.

### Idempotency note

Craft `blocks add` is **not** idempotent — a retry after a 502 that actually
succeeded server-side will duplicate the blocks. After a successful retry,
run `blocks get <rootBlockId> --format markdown` and prune any duplicates
with `blocks delete` before continuing.

## Pre-Finalize Checklist

After creating or editing a document:

1. **Read back**: `blocks get <rootBlockId> --format markdown`
2. **Verify**: All sections present, no literal `\n` characters visible,
   correct ordering, no duplicate or stray blocks
3. **If broken**: Delete stray blocks with `blocks delete --id <blockId>`,
   fix and re-add
4. **Report done** only after confirming the readback is clean

## Canonical Example

Create a project doc with heading, tasks, code, and bullets. Uses JSON array
mode to avoid shell quoting issues:

```bash
# Step 1: Create document
documents create --title "API Spec" --folder "<folderId>" --icon "📋"
# → rootBlockId: ABC-123

# Step 2: Add all content (JSON array — safe for any characters)
blocks add --id ABC-123 --json '[
  {"type":"text","textStyle":"h2","markdown":"## Overview"},
  {"type":"text","markdown":"API spec for the auth service."},
  {"type":"text","textStyle":"h2","markdown":"## Tasks"},
  {"type":"text","listStyle":"task","markdown":"- [ ] Define endpoints","taskInfo":{"state":"todo"}},
  {"type":"text","listStyle":"task","markdown":"- [x] Choose framework","taskInfo":{"state":"done"}},
  {"type":"code","rawCode":"POST /auth/login\nContent-Type: application/json","codeLanguage":"http"},
  {"type":"text","decorations":["callout"],"markdown":"<callout>Do not store tokens in localStorage</callout>"}
]'

# Step 3: Verify
blocks get ABC-123 --format markdown
```

For simple prose-only content without backticks or special characters,
`--markdown` with `\n\n` separators works in a single call. When in doubt,
use JSON.
