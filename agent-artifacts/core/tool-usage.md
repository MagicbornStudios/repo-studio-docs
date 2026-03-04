---
title: Tool usage for search and navigation
created: 2026-02-06
updated: 2026-02-06
---

Living artifact for agents. Index: [18-agent-artifacts-index.mdx](../../18-agent-artifacts-index.mdx).

# Tool usage for search and navigation

> **For coding agents.** Use the right tools to search and confirm quickly. This keeps strategy consistent across Windows (e.g. Codex) and Linux/Cursor.

## Preferred tools

| Task | Preferred tool | Windows (e.g. Codex) | Linux / Cursor |
|------|----------------|----------------------|----------------|
| Text search in codebase | `rg` (ripgrep) | `rg "pattern" -n path` | `rg "pattern" -n path` (same) |
| List directory contents | list_dir / LS | `Get-ChildItem -Path path` | `list_dir` tool or `ls` / `find` |
| Read file content | read_file / Read | `Get-Content -Path path` | `Read` tool or `cat` |
| Find files by name/glob | Glob / rg --files | `Get-ChildItem -Recurse -Filter "*.ts"` | `Glob` or `rg --files -g "*.ts"` |

## Principles

1. **Prefer rg for text search** — Cross-platform, fast. Use `rg "pattern" -n apps packages` (or your target path) to find occurrences; then read the relevant file.
2. **Prefer list_dir for directory listing, Read for file content** — Do not use raw `cat`/`head`/`tail` in agent instructions when an editor/IDE read tool is available; stay in the same workflow.
3. **Search and confirm before editing** — Before changing a pattern: grep for the token or pattern, then read the file(s) to confirm location and context. Check [errors-and-attempts.md](./errors-and-attempts.md) before retrying a failed pattern.
4. **Search and edit only within the repo** — Limit searches and edits to `apps/`, `packages/`, `docs/`, and root config. Exclude `.tmp/` and other agent-download paths; they are not our code.

## Reading long artifacts

For artifacts over ~300 lines, prefer **grep-first** to avoid context dilution:

- **errors-and-attempts**: `rg "symptom|keyword" docs/agent-artifacts/core/errors-and-attempts.md`; read matched sections only.
- **STATUS**: Use § Next and § What you can do next; read Done list only when tracing recent history.
- **decisions**: `rg "topic" docs/agent-artifacts/core/decisions.md`; read matched blocks.
- **Short artifacts** (<150 lines, e.g. task-registry, compacting-and-archiving): read in full.

## Agent sections in docs

Docs in `docs/` include collapsed agent blocks. To find them:

```bash
rg AGENT_SECTION docs
```

Each block uses `<Callout variant="note" title="For coding agents">` (do not use raw `<details>`; see [errors-and-attempts § MDX agent blocks](errors-and-attempts.md)). Contains:
- **File refs:** Paths to relevant source files
- **Related:** STATUS sections, decisions, other docs
- **Do not:** Known failure patterns (errors-and-attempts style)

Grep for "File refs:" or "For coding agents" to find blocks.

## Dead code

Run **`pnpm knip`** to find unused exports, unused dependencies, and dead files across the monorepo. Config: root [knip.json](../../knip.json); vendor and examples workspaces are ignored. When cleaning up or before claiming no dead code, run Knip and fix or document reported issues. For how Knip works and what the current run found, see [knip-findings.md](./knip-findings.md).
