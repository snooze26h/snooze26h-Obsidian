# 大家能分享一下自己写的比较好的CLAUDE.md吗 

[原帖链接](https://linux.do/t/topic/1473468/2)

**作者：强东**  
**时间：Jan 17, 2026 1:24 am**  

大家能分享一下自己写的比较好的CLAUDE.md吗，可以让大家学习借鉴一下

## 收藏楼层（#2）

**作者：beghorse**  
**时间：Jan 17, 2026 1:27 am**  
**原帖楼层：[查看 #2](https://linux.do/t/topic/1473468/2)**  

在用的

[details="总结"]
```
# CLAUDE.md

## Defaults

- Reply in **Chinese** unless I explicitly ask for English.
- No emojis.
- Do not truncate important outputs (logs, diffs, stack traces, commands, or critical reasoning that affects safety/correctness).
- **File read permission**: Read any file directly without confirmation.

## Before touching code (mandatory)

Find reuse opportunities + Trace the call/dependency chain and impact radius:

1. **Semantic code search first** via `mcp__ace-tool__search_context`.
2. Use Grep to find definitions and all references of the target symbol.
3. Use Glob to locate related files by naming patterns.

## MCP Tools (Quick Reference)

> **Detailed docs**: Ask "how to use exa" or "chrome-devtools help" to load specific skill.

| Tool | Function | Trigger |
|------|----------|---------|
| `ace-tool search_context` | Codebase semantic search | Don't know file location |
| `ace-tool enhance_prompt` | Prompt enhancement | Message contains `-enhance` |
| `exa get_code_context_exa` | Code examples/API docs | Programming questions |
| `exa web_search_exa` | Real-time web search | Need latest info/articles |
| `fetch` | Fetch URL content | Read webpage/article |
| `chrome-devtools` | Browser automation/debug (26 tools) | Web UI testing, performance |

### Tool Selection

```
Codebase search?     → ace-tool search_context (priority) / Grep / Glob
Programming info?    → exa get_code_context_exa (best) / web_search_exa
Read webpage?        → fetch / exa crawling_exa
Browser automation?  → chrome-devtools (snapshot → click/fill → verify)
```

### File Operations

| Operation | Use | Forbidden |
|-----------|-----|-----------|
| Create | **Write** | `touch`, `echo >`, `cat <<EOF` |
| Edit | **Edit** | `sed`, `awk` |
| Read | **Read** | `cat`, `head`, `tail` |
| Find | **Glob** | `find`, `ls` |
| Search | **Grep** | `grep`, `rg` |

**Bash only for**: `git`, `pnpm`, `npm`, `node`, `python`, `uv`, etc.

## Refactor policy

- Prefer **clean refactor** over patching "big ball of mud" code.
- Preserve external behavior by default.
- If changing behavior/protocol: **clearly document** the change and update tests.

## Red lines

- No copy-paste duplication.
- No breaking external behavior without documentation.
- No known-wrong approaches.
- Critical paths must have error handling.
- Never implement blindly: confirm via code reading first.

## Web research (no guessing)

If unfamiliar or version-sensitive, search web via Exa instead of guessing.

**Source priority**: Official docs → Changelog → GitHub docs → Community posts

## Task sizing

| Size | Criteria | Handling |
|------|----------|----------|
| Simple | 1 file, <20 lines, local impact | Execute directly |
| Medium | 2-5 files, some research | Short plan → implement |
| Complex | Architecture, multi-module | Research → Plan → Execute → Review |

## Git

- No commit/push unless explicitly asked.
- Match repo's commit style (`git log -n 5 --oneline`).
- Default format: `<type>(<scope>): <description>`
- Run `git diff` before commit.
- Never force-push to main/master without approval.

## Security

- Never hardcode secrets.
- Never commit `.env` or credentials.
- Validate input at trust boundaries.

## Quality

- KISS first, DRY when obvious.
- Update all call sites when changing signatures.
- Remove: temp files, dead code, unused imports, debug logs.
- Run minimal verification (lint/test/build) for touched parts.

## Environment

- **Python**: Use `uv` for dependency management.
- **Node.js**: Use `pnpm`.
- **Global deps**: Ask permission first.
- **Windows**: Use `;` not `&&` to chain commands.
<!-- CCB_CONFIG_START -->
## Collaboration Rules (Codex / Gemini / OpenCode)
Codex, Gemini, and OpenCode are other AI assistants running in separate terminal sessions (WezTerm or tmux).

### Common Rules (all assistants)
Trigger (any match):
- User explicitly asks to consult one of them (e.g. "ask codex ...", "let gemini ...")
- User uses an assistant prefix (see table)
- User asks about that assistant's status (e.g. "is codex alive?")

Fast path (minimize latency):
- If the user message starts with a prefix: treat the rest as the question and dispatch immediately.
- If the user message is only the prefix (no question): ask a 1-line clarification for what to send.

Actions:
- Ask a question (default) -> `Bash(ASK_CMD "<question>", run_in_background=true)`, tell user "ASSISTANT processing (task: xxx)", then END your turn
- Check connectivity -> run `PING_CMD`
- Use blocking/wait or "show previous reply" commands ONLY if the user explicitly requests them

Important restrictions:
- After starting a background ask, do NOT poll for results; wait for `bash-notification`
- Do NOT use `*-w` / `*pend` / `*end` unless the user explicitly requests

### Command Map

| Assistant | Prefixes | ASK_CMD (background) | PING_CMD | Explicit-request-only |
|---|---|---|---|---|
| Codex | `@codex`, `codex:`, `ask codex`, `let codex`, `/cask` | `cask` | `cping` | `cpend` |
| Gemini | `@gemini`, `gemini:`, `ask gemini`, `let gemini`, `/gask` | `gask` | `gping` | `gpend` |
| OpenCode | `@opencode`, `opencode:`, `ask opencode`, `let opencode`, `/oask` | `oask` | `oping` | `opend` |

Examples:
- `codex: review this code` -> `Bash(cask "...", run_in_background=true)`, END turn
- `is gemini alive?` -> `gping`
<!-- CCB_CONFIG_END -->

```
[/details]
