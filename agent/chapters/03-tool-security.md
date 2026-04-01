# Tool & Security Design Patterns

Patterns extracted from Claude Code's 40+ tool implementations and 6-layer permission system.

---

## Tool framework pattern

```
Tool<Input, Output, Progress>
  - inputSchema: Zod validation
  - checkPermissions(): tool-specific permission logic
  - call(): execution logic
  - renderToolResultMessage(): UI rendering
  - onProgress: streaming progress callback
```

Factory: `buildTool()` provides sensible defaults. New tools only define what's different.

Context object pattern: single `ToolUseContext` parameter instead of many args. Enables shallow-copy cloning for subagent context.

## Tool result size management

```
Tool returns result
  → size > maxResultSizeChars?
    YES → save to ~/.claude/tool-results/ + content hash dedup
          API only receives summary preview
    NO  → send directly to API
  → session-level budget tracking
```

Key: large results don't blow up the context window.

## 6-layer permission chain

```
Layer 1: Deny rules     → match → REJECT (highest priority)
Layer 2: Allow rules     → match → ALLOW
Layer 3: Ask rules       → match → prompt user
Layer 4: Hook execution  → hook returns allow/deny → adopt
Layer 5: ML classifier   → safe → allow / uncertain → continue
Layer 6: User prompt     → or auto-deny based on mode
```

## Permission modes

| Mode | Behavior |
|------|----------|
| default | Prompt for each tool use |
| acceptEdits | Auto-accept file edits |
| bypassPermissions | Allow all (dangerous) |
| plan | Planning mode, special handling |
| auto | ML classifier decides (feature-gated) |
| bubble | Surface prompts to parent agent |

## Permission rule syntax

```
"Bash(git:*)"          — all git commands
"Read(*.ts)"           — read TypeScript files
"Edit(src/**/*.tsx)"   — edit in src directory
"Write"                — all write operations
```

## Bash security: AST-level analysis

Not regex matching — parses Bash AST via `utils/bash/ast.ts`:

| Category | Examples |
|----------|---------|
| Filesystem destruction | `rm -rf /`, `dd if=/dev/zero` |
| Credential theft | access `.env`, credentials files |
| Data exfiltration | `curl` sending sensitive data |
| System damage | `chmod 777`, `kill -9` |

## Command semantic classification

```
Search: grep, rg, find, ag    → safe, UI collapsible
Read: cat, head, tail, less    → safe, UI collapsible
List: ls, dir, tree            → safe, UI collapsible
Neutral: echo, printf          → ignored in pipelines
Modify: sed, awk               → needs validation
Dangerous: rm, dd, chmod       → needs permission
```

## Permission fatigue detection

```
Track consecutive denials
  → exceed threshold N
    → stop asking one-by-one
    → prompt for batch permission instead
```

## Auto-mode classifier (2-stage)

```
Stage 1 (fast): classify based on tool_use block
  → high confidence → decide
Stage 2 (thinking): XML + chain-of-thought reasoning
  → return allow/deny + confidence (high/medium/low)
```
