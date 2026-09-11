---
name: siyuan-skill
description: >
  Operate SiYuan Note CLI (siyuan) to manage workspaces, notebooks, documents, blocks, SQL queries,
  search, export/import, assets, attributes, bookmarks, tags, history, refs, repo snapshots, sync, serve,
  database attribute views, templates, outlines, and system info. Use this skill when the user mentions
  SiYuan, 思源笔记, or asks to query/manipulate SiYuan data via CLI. The skill enforces the correct
  workflow: discover workspaces first, then target a specific workspace for all subsequent commands.
agent_created: true
---

# SiYuan Note CLI Skill

## Overview

SiYuan Note provides a CLI (`siyuan`) to manage workspaces, notebooks, documents, blocks, and more.
The binary must be available on `PATH` (verify with `siyuan --version`).

> **CLI Version:** v3.8.4-alpha.5. Commands shown below reflect this version.

> **Note:** If `siyuan` is not found on `PATH`, try `SiYuan-Kernel` as the alternative command name.

## Workflow

Follow this sequence for every user request:

### 1. Discover Available Workspaces

Run `siyuan workspace list` (no `-w` flag needed) to obtain the list of registered workspaces.
If the user has not specified a workspace, present the list and ask them to choose.

```
siyuan workspace list
siyuan workspace info          # show current default workspace
```

### 2. Lock in the Workspace Path

After the user selects a workspace (or provides a path directly), store the workspace path and use it
as the `-w` argument for **all subsequent commands** (except `workspace` subcommands).

### 3. Execute the Requested Command

Construct and run the target command with `-w <workspace-path>` appended (except `workspace` subcommand).
Use `-f json` for programmatic output when parsing results.

If a command fails, check: (1) workspace path is valid, (2) required flags like `--id` are provided, (3) the subcommand exists in the installed CLI version.

## Common Patterns

### Obtaining Block/Document IDs

Many commands require a block or document ID (`--id`). Obtain IDs via:
- `siyuan sql "SELECT id, content FROM blocks WHERE type='d'" -w <path> -f json` — list document IDs
- `siyuan document list --notebook <id> -w <path> -f json` — list documents in a notebook
- `siyuan search <keyword> -w <path> -f json` — search and get IDs from results

### Dry-Run for Destructive Operations

Before running destructive commands (`remove`, `delete`, `clean`, `clear`, `purge`), prepend `--dry-run` to preview the effect without making changes:
```
siyuan notebook remove --id <id> --dry-run -w <path>
```

### JSON Output Parsing

Use `-f json` for programmatic parsing. The output is a JSON array of objects; field names match the SiYuan kernel API response schema. Pipe to `jq` or similar tools for filtering.

### Workspace Lock Errors

If a command exits with code 24 and reports `lock workspace [...] failed`, the workspace is already in use by another SiYuan process (e.g., the desktop app). Either close the other process or target a different workspace.

### Snapshot After Write Operations

When a conversation session involved any write operations (create, append, prepend, insert, update, move, rename, remove, delete, clean, clear, purge, set, import, convert), summarize the changes made and create a data snapshot before the session ends:
```
siyuan repo create --memo "<brief summary of changes>" -w <path>
```
The `--memo` should concisely describe what was created/modified/deleted (e.g. `"Created 3 docs in /Projects, updated README, removed obsolete notes"`). Skip the snapshot if the session was read-only.

## Domain Concepts

Understanding these SiYuan-specific concepts is essential for correct CLI usage:

### Block — The Fundamental Unit
Everything is a block with a unique ID. A document is a document block (type `d`, i.e. NodeDocument) that serves as the root; content blocks (headings, paragraphs, lists, code, tables) form a tree beneath it.

**Container blocks** (can hold child blocks, valid as `--parent`): document, blockquote, list, list-item, super-block, callout.
**Leaf blocks** (cannot hold children, invalid as `--parent`): heading, paragraph, code-block, math-block, table, HTML-block, thematic-break, video, audio, widget, iframe, attribute-view, block-query-embed.

### Heading Hierarchy — Use `--previous`, Not `--parent`
Headings (h1-h6) are **leaf blocks**. Blocks that appear "under" a heading in the UI are its *following siblings* in the AST, not its children. To insert a block below a heading, pass the heading's ID (or the ID of the last block currently below it) as `--previous`, **not** as `--parent`:
```
siyuan block insert --parent <docID> --previous <headingID> --data "text" -w <path>
```

### Nested Lists — list-item Cannot Directly Hold list-item
A list-item cannot directly contain another list-item. To nest lists, create a list (NodeList) as a child of the outer list-item, then add list-items to that inner list. The parent of a list-item must always be a list.

### hPath — Human-Readable Path
The `--path`/`--hpath` parameter in `document create`/`move`/`list` refers to the **title-based path** shown in the document tree (e.g. `/Diary/2024/June`), not the internal ID-based filesystem path. Renaming a document changes its hPath but not its ID.

### block update — Replace, Not Append
`block update` replaces **ONE** block's entire content with new markdown. It does NOT create or append new blocks. To both modify existing content and add new content, call `block update` first, then `block append`/`prepend`/`insert` as separate calls. Pass `--lock-type` to reject the update when the parsed block type of the new content differs from the existing block's type.

### document move vs block move
`document move` relocates an entire document (and all its children) to a new hPath within a notebook. `block move` repositions a single content block under a new parent block.

### Daily Note — When to Use dailynote vs document
A daily note is a special document on the notebook's daily-note save path. For diary/journal/daily-note requests, use `dailynote create` (not `document create`) to open today's note, then `dailynote append`/`prepend` to add content.

### Notebook Icon vs Document Icon
`attr set` changes a document **block's** icon — it cannot set a **notebook's** icon. For notebooks use `notebook set-icon` (specific emoji) or `notebook random-icon` (random emoji).

## Global Flags

These flags are available on every subcommand:

| Flag | Description |
|------|-------------|
| `-w, --workspace <path>` | Workspace path (required for all commands except `workspace`; optional for `serve`, which defaults to the default workspace). |
| `-f, --format <table\|json>` | Output format (default: `table`) |
| `-v, --log-level <string>` | Log level: off | trace | debug | info | warn | error | fatal (defaults to conf.json system.logLevel) |
| `--dry-run` | Validate and print what would happen without making changes |

> **Note:** All flags except `-w` (or `--workspace`) are optional. The CLI will use default values if optional flags are not specified.

## Subcommand Reference

> **For detailed flags and examples of each command, load `references/commands.md`.**

| Command | Subcommands |
|---------|-------------|
| `siyuan workspace` | `list`, `info` |
| `siyuan notebook` | `list`, `create`, `open`, `close`, `rename`, `remove`, `set-icon`, `random-icon` |
| `siyuan document` | `list`, `get`/`info`, `create`, `rename`, `move`, `duplicate`, `remove`, `search` |
| `siyuan block` | `get`, `children`, `insert`, `append`, `prepend`, `update`, `delete`, `move`, `stat`, `dom`, `kramdown`, `breadcrumb`, `batch-get`, `batch-kramdown` |
| `siyuan sql "<stmt>"` | (direct) |
| `siyuan search <query>` | (direct, with flags) |
| `siyuan export` | `md`, `html`, `docx`, `sy`, `md-zip`, `preview`, `data` |
| `siyuan import` | `md`, `sy`, `data` |
| `siyuan dailynote` | `create`, `append`, `prepend` |
| `siyuan asset` | `unused`, `clean`, `stat`, `upload` |
| `siyuan attr` | `get`, `set`, `batch-get` |
| `siyuan bookmark` | `list`, `labels`, `rename`, `remove` |
| `siyuan tag` | `list`, `rename`, `remove` |
| `siyuan history` | `list`, `get`, `search`, `rollback`, `clear` |
| `siyuan ref` | `backlinks`, `mentions`, `refresh` |
| `siyuan repo` | `list`, `create`, `diff`, `checkout`, `tag`, `untag`, `purge`, `search`, `file` |
| `siyuan sync` | `pull`, `push`, `status` |
| `siyuan database` | `search`, `get`, `keys`, `render`, `unused`, `clean`, `key`, `item` |
| `siyuan file` | `list`, `read`, `write`, `copy`, `rename`, `delete`, `find`, `grep`, `stat` |
| `siyuan inbox` | `list`, `get`, `convert` |
| `siyuan outline` | `get` |
| `siyuan template` | `search`, `get`, `create`, `save-as`, `render`, `remove` |
| `siyuan system` | `current-time` |
| `siyuan serve` | (direct, with flags) |
| `siyuan completion` | `bash`, `fish`, `powershell`, `zsh` |

> **Note:** `serve` accepts all global flags plus its own flags (`--accessAuthCode`, `--attach-ui`, `--enable-pprof`, `--lang`, `--mode`, `--port`, `--readonly`, `--ssl`, `--safe-mode`, `--wd`). It does not require `-w`; see `references/commands.md` for the full flag list.
