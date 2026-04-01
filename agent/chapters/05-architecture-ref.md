# Architecture Quick Reference

Claude Code internals at a glance. ~1,902 files, 512K lines TypeScript, Bun runtime.

---

## Request flow

```
User input
  → main.tsx (Commander.js CLI parser, 4683 lines)
    → QueryEngine.ts (session mgmt, message history, 1295 lines)
      → query.ts (main loop: normalize → API call → tool execution, 1729 lines)
        → services/api/claude.ts (Anthropic SDK wrapper)
        → tools/* (40+ tool implementations)
      → compaction (if context exceeds token threshold)
    → render output (React/Ink)
```

## Key files

| File | Lines | Role |
|------|-------|------|
| main.tsx | 4,683 | CLI entry + React/Ink renderer |
| QueryEngine.ts | 1,295 | LLM call orchestration, retry, token counting |
| query.ts | 1,729 | Main loop, message normalization, streaming tool execution |
| Tool.ts | — | Tool type system: `Tool<Input, Output, Progress>` |
| commands.ts | — | 60+ slash command registry |
| context.ts | — | System/user context collection |
| cost-tracker.ts | — | Token cost tracking per model |

## Directory map

```
src/
├── tools/          40+ tool implementations (BashTool 161KB, AgentTool 235KB)
├── commands/       60+ slash commands
├── services/       API client, MCP, OAuth, LSP, analytics, compact
├── bridge/         IDE bidirectional communication (VS Code / JetBrains)
├── coordinator/    Multi-agent orchestration
├── plugins/        Plugin system
├── skills/         Skill system
├── memdir/         Persistent memory
├── hooks/          React hooks + permission hooks
├── tasks/          Background task management
├── utils/          Permissions engine, Bash AST parser, system prompt assembly
```

## Tech stack

| Category | Technology |
|----------|-----------|
| Runtime | Bun |
| Language | TypeScript (strict) |
| Terminal UI | React + Ink |
| CLI | Commander.js |
| Schema | Zod v4 |
| Code search | ripgrep |
| Protocols | MCP SDK, LSP |
| API | Anthropic SDK |
| Telemetry | OpenTelemetry + gRPC |
| Feature flags | GrowthBook (build-time: Bun `feature()`) |
| Auth | OAuth 2.0, JWT |

## 15 key design patterns

1. **Feature-gated dead code elimination** — `feature('FLAG') ? require('./mod') : null`, Bun strips false branches at build time
2. **Context object** — single `ToolUseContext` param, shallow-copy for subagent cloning
3. **Large result disk persistence** — save to `~/.claude/tool-results/`, API gets summary only, content hash dedup
4. **Parallel prefetch** — MDM/keychain/GrowthBook IO starts before module evaluation
5. **Memoized prompt sections** — cached until explicit invalidation (`/clear`, `/compact`)
6. **Lazy module loading** — OpenTelemetry, React, analytics loaded on first use
7. **Permission denial tracking** — consecutive denials → escalate to batch permission request
8. **Fork prompt cache sharing** — byte-identical child prompts → API cache hit → ~75% token savings
9. **Streaming tool executor** — start tool execution during model streaming response
10. **Multi-layer permission chain** — Deny→Allow→Ask→Hook→Classifier→Default
11. **Command semantic classification** — search/read commands auto-collapsed in UI
12. **Git status snapshot** — captured once at session start, never updated (performance vs freshness tradeoff)
13. **buildTool() factory** — factory with sensible defaults, new tools only define differences
14. **Analytics type safety** — `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` (type name as audit signature)
15. **Hybrid transport degradation** — WebSocket → SSE → HTTP polling with exponential backoff + jitter

## 40+ feature flags (unreleased)

| Flag | Likely feature |
|------|---------------|
| PROACTIVE | Autonomous agent mode |
| COORDINATOR_MODE | Multi-worker orchestration |
| FORK_SUBAGENT | Implicit fork with cache sharing |
| ULTRAPLAN | Enhanced planning |
| ULTRATHINK | Extended thinking |
| WEB_BROWSER_TOOL | Web browsing |
| VOICE_MODE | Voice input (16kHz/mono) |
| TRANSCRIPT_CLASSIFIER | Auto-mode ML classifier |
| EXTRACT_MEMORIES | Auto memory extraction |
| TEAMMEM | Team memory sync |
| WORKFLOW_SCRIPTS | Workflow automation |
| BUDDY | Terminal companion sprite |

## Cost tracking dimensions

```
inputTokens (per model), outputTokens, cacheReadTokens, cacheWriteTokens,
apiDuration (with/without retries), totalCostUSD,
linesAdded, linesRemoved, webSearchRequests
```

## Context compression

| Mode | Trigger | Mechanism |
|------|---------|-----------|
| Manual | `/compact [instructions]` | User-initiated with optional custom summary |
| Auto | Token threshold exceeded | AI summarizes history, replaces old messages |
| Incremental | CACHED_MICROCOMPACT flag | Only compress new content, keep existing |

## Bridge (IDE integration)

Transport: WebSocket (preferred) → SSE (fallback) → HTTP polling (last resort)
Auth: OAuth bearer + trusted device token + session ingress token + JWT
Concurrency: 32 concurrent sessions, 24h timeout
