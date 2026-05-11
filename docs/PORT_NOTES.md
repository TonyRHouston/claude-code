# Claude Code → TcodeX Port Notes

This repository (`HoustoDev/claude-code`) is a reference fork of [anthropics/claude-code](https://github.com/anthropics/claude-code). It is **no longer used as a runtime dependency** of [TcodeX](https://github.com/TonyRHouston/TcodeX). As of the multi-agent upgrade (TcodeX#33), TcodeX talks to Claude directly via the Anthropic Python SDK and no longer invokes the Node `claude` binary.

This document records which Claude Code TypeScript modules informed which Python port in TcodeX. The mapping is intentional and minimal — TcodeX adopts the *patterns* (streaming agent loop, normalized tool events, sandboxed tool dispatch, slash commands, session persistence) without inheriting the Ink TUI, the npm build, or the Node runtime.

## Concept mapping

| Claude Code (this repo) | TcodeX | Notes |
|---|---|---|
| Agent loop in `src/agent/loop.ts` | [`tcodex/agents/base.py:BaseAgent.run`](https://github.com/TonyRHouston/TcodeX/blob/claude/integrate-codex-claude-19Rti/tcodex/agents/base.py) | Stream → accumulate tool_uses → dispatch → append tool_results → repeat. One shared implementation that every specialist subclasses with a different `system_prompt` and tool whitelist. |
| Tool definitions in `src/tools/` (Read, Write, Bash, Glob, Grep, …) | [`tcodex/tools/`](https://github.com/TonyRHouston/TcodeX/tree/claude/integrate-codex-claude-19Rti/tcodex/tools) | `read_file`, `write_file`, `list_dir`, `grep`, `shell_exec` in Phase 1. `apply_diff`, `glob`, `web_fetch`, `mcp_client` follow in Phase 3. |
| `src/tools/Bash` permission policy | [`tcodex/tools/sandbox.py`](https://github.com/TonyRHouston/TcodeX/blob/claude/integrate-codex-claude-19Rti/tcodex/tools/sandbox.py) | Three sandbox modes mirror Codex's; the shell allowlist is consulted by `SandboxPolicy.check_shell` before every `shell_exec`. |
| `src/services/anthropic.ts` streaming | [`tcodex/providers/anthropic_provider.py`](https://github.com/TonyRHouston/TcodeX/blob/claude/integrate-codex-claude-19Rti/tcodex/providers/anthropic_provider.py) | Same SDK (Anthropic Python instead of TypeScript), same ephemeral cache breakpoints on the system prompt and last two user turns. |
| `src/utils/slashCommands.ts` | [`tcodex/slash.py`](https://github.com/TonyRHouston/TcodeX/blob/claude/integrate-codex-claude-19Rti/tcodex/slash.py) | `/help /exit /reset /model /sandbox /cost` in Phase 1; `/plan /status /continue /replan /budget /done /agents` in Phase 2; `/save /load /files` in Phase 3. |
| Conversation persistence (`~/.claude/projects/*/sessions/`) | [`tcodex/state/session.py`](https://github.com/TonyRHouston/TcodeX/blob/claude/integrate-codex-claude-19Rti/tcodex/state/session.py) | JSON sessions under `~/.tcodex/sessions/<id>.json`, written after every turn for crash-resume. |
| TodoWrite tool for turn-level task tracking | `tcodex/loop/task_graph.py` *(Phase 2)* | Promoted from a tool to a first-class loop primitive — the orchestrator mutates the graph each iteration and the REPL renders it live. |
| MCP client in `src/services/mcp/` | `tcodex/tools/mcp_client.py` *(Phase 3)* | Minimal stdio JSON-RPC client; server list read from `~/.tcodex/config.toml`. |

## What's *not* ported

- The Ink/React TUI in `src/`. TcodeX uses `prompt_toolkit + rich` instead — both render a streaming chat with live tool-call panels, but TcodeX is single-process Python with no Node runtime.
- The npm packaging and the `@anthropic-ai/claude-code` distribution. TcodeX ships as a single `pyproject.toml` package installable with `pip`.
- IDE extensions (VS Code, JetBrains). TcodeX is a terminal application; IDE integration is out of scope.
- The hosted plugin system (`plugins/`). TcodeX-level plugins would attach via MCP rather than a per-plugin loader.

## Why fork-and-port instead of subprocess

Wrapping Claude Code as a binary would have required the user to install Node 18+, the `claude` package, and an additional runtime path for streaming. Porting the *capabilities* into Python directly:

- Removes the cross-runtime dependency.
- Lets TcodeX's orchestrator see tool calls as in-process events instead of stdout/stderr text.
- Keeps the token counter and cost meter authoritative — every chunk flows through `tcodex/providers/base.py:StreamChunk` and contributes to the session usage totals.
- Aligns with TcodeX's existing Python stack (`gpt_engineer/core/*`) which the new package reuses via the documented adapter map in `ARCHITECTURE.md`.

## See also

- [TcodeX#33 — Phase 1 PR](https://github.com/TonyRHouston/TcodeX/pull/33)
- [TcodeX `ARCHITECTURE.md`](https://github.com/TonyRHouston/TcodeX/blob/claude/integrate-codex-claude-19Rti/ARCHITECTURE.md)
- Upstream Claude Code source: [anthropics/claude-code](https://github.com/anthropics/claude-code)
