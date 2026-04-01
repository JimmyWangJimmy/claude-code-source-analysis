# Multi-Agent Design Checklist

14 design principles extracted from Claude Code's multi-agent system. Use as a checklist when building your own.

---

## Four agent modes

| Mode | Context | Lifecycle | Communication | Use when |
|------|---------|-----------|---------------|----------|
| Subagent | Clean | One-shot | Result return | Independent search/analysis |
| Fork | Inherits parent | One-shot | Result return | Parallel tasks needing current context |
| Teammate | Independent | Persistent | Message passing | Long-term collaboration roles |
| Coordinator | Orchestration layer | Persistent | Structured notifications | Complex multi-step workflows |

## Isolation matrix

| Resource | Strategy | Reason |
|----------|----------|--------|
| AbortController | Independent, parent abort propagates down | Cancel parent → cancel child, cancel child ≠ cancel parent |
| File state cache | Deep clone | Prevent concurrent write conflicts |
| Permission decisions | Fresh set | Child must not inherit parent's approvals |
| Tool list | Configurable subset | Least privilege principle |
| Message history | Fork: inherit / Subagent: independent | Fork needs context; Subagent needs clean start |
| MCP servers | Additive, independently managed | Child can add connections, cleanup on exit |
| Memory | Isolated by agent type | Different types accumulate different experience |
| System prompt | Overridable | Different agents have different roles |

## Fork cache sharing (saves ~75% input tokens)

```
Parent prompt = P
Fork child prompt = P (byte-identical)
  → Anthropic API prompt cache hit
  → Only the directive (last text block) differs per child
```

Requirements for cache hit:
- System prompt: byte-identical (via override)
- Tool list: exact same pool (useExactTools: true)
- Message history: full parent history preserved
- Tool results: same placeholder text
- Only divergence: per-child directive

## Inter-agent communication (4 modes)

| Need | Method |
|------|--------|
| Task result | Return value / callback |
| Agent coordination | Message queue (in-process) |
| Large data exchange | Shared filesystem (Scratchpad) |
| Cross-process/machine | Socket / WebSocket |

Structured message types: `shutdown_request`, `shutdown_response`, `plan_approval_response`

## Permission delegation rules

```
Rule 1: child tool set ⊂ parent tool set
Rule 2: child does NOT inherit parent's permission approvals
Rule 3: background agents cannot trigger interactive permission prompts
Rule 4: permission mode is configurable per agent type
```

## Anti-recursion (3 layers)

```
Layer 1 (Prompt): Tell child "you ARE the fork, do NOT spawn sub-agents"
Layer 2 (Code): Detect FORK_BOILERPLATE_TAG in message history, reject
Layer 3 (Limit): Max nesting depth / max total agent count
```

## Agent output format constraint

Child agent output is consumed by parent agent (another LLM), not humans:

```
Rules for child agent:
1. No explanations, no suggestions — report facts only
2. Strict output format: Scope / Result / Key files / Files changed / Issues
3. Under 500 words
4. Response MUST begin with "Scope:"
5. No text between tool calls — use tools silently, report once at end
```

## Resource budget per agent

```
max_turns: 200
max_tokens: 500,000
max_duration: 600s
min_output_per_turn: 500 (diminishing returns detection)
max_concurrent_agents: 32
```

## Agent memory: 3 scopes × N types

```
user scope:    ~/.claude/agent-memory/{type}/
project scope: .claude/agent-memory/{type}/       (git-trackable)
local scope:   .claude/agent-memory-local/{type}/ (gitignored)
```

Each agent type has a separate memory directory.

## Declarative agent definition

Agents defined via markdown frontmatter:

```yaml
---
name: researcher
tools: [Read, Glob, Grep]
model: claude-sonnet-4-6
permissionMode: acceptEdits
memory: project
isolation: worktree
maxTurns: 100
---
System prompt here...
```

Loading priority: built-in → plugin → user → project → flag → managed

---

## Full checklist

### Architecture
- [ ] Declarative agent type definitions (YAML/frontmatter)
- [ ] Isolation matrix: per-resource sharing strategy defined
- [ ] AbortController chain: cancel signal propagates downward only
- [ ] Resource cleanup: finally block cleans all held resources
- [ ] Multiple agent modes: one-shot / fork / persistent / orchestrator

### Communication
- [ ] Small data via message passing
- [ ] Large data via shared filesystem (Scratchpad)
- [ ] Coordination via structured messages (not natural language)
- [ ] Result notification via structured XML/JSON

### Security
- [ ] Permissions decrease per level, never inherited
- [ ] Child agent tool whitelist
- [ ] Background agents cannot trigger interactive permission prompts
- [ ] Anti-recursion: prompt + code tag + depth limit

### Cost
- [ ] Token budget constraint
- [ ] Turn limit
- [ ] Time timeout
- [ ] Diminishing returns detection
- [ ] Fork cache sharing optimization

### Memory
- [ ] Memory isolated by agent type
- [ ] Global / project / local scopes
- [ ] Auto-load agent memory on spawn
- [ ] Memory drift protection (verify before acting)
