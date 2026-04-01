# Ready-to-Paste Configs

Copy any block below into the corresponding file.

---

## settings.json

Location: `~/.claude/settings.json` (global) or `.claude/settings.json` (project)

```json
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "Bash(git:*)", "Bash(npm:test)", "Bash(npm:run:*)", "Bash(npx:*)",
      "Bash(node:*)", "Bash(python:*)",
      "Bash(ls:*)", "Bash(cat:*)", "Bash(head:*)", "Bash(tail:*)",
      "Bash(find:*)", "Bash(grep:*)", "Bash(rg:*)", "Bash(wc:*)",
      "Bash(echo:*)", "Bash(which:*)", "Bash(env:*)", "Bash(pwd)", "Bash(date)"
    ],
    "deny": [
      "Bash(rm:-rf:*)", "Bash(sudo:*)", "Bash(chmod:777:*)",
      "Bash(curl:*:POST:*)", "Bash(dd:*)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "command": "prettier --write --ignore-unknown $CHANGED_FILES 2>/dev/null || true",
        "if": "Edit(*)",
        "timeout": 10,
        "async": true
      }
    ]
  },
  "enableAutoMemory": true
}
```

## CLAUDE.md template

Location: project root `CLAUDE.md`

```markdown
# claude
- Language: TypeScript
- Framework: React + Node.js
- Package manager: pnpm
- Test runner: vitest
- Formatter: prettier + eslint
- Commit messages in Chinese
- Do not add comments or type annotations to unchanged code
- Read files before modifying them
- One task at a time
- On error: analyze cause first, do not retry the same command
```

## Global preferences

Location: `~/CLAUDE.md`

```markdown
# claude
- Respond in Chinese
- No emojis
- Explain approach before executing on important decisions
- Ask when uncertain, do not guess
- No time estimates
```

## Environment variables

Add to `.bashrc` / `.zshrc`:

```bash
export ANTHROPIC_SMALL_FAST_MODEL=claude-haiku-4-5-20251001
export BASH_DEFAULT_TIMEOUT_MS=120000
export BASH_MAX_TIMEOUT_MS=600000
```

## Shell aliases

```bash
alias cc='claude'
alias ccp='claude --print'
alias ccf='claude --fast'
alias ccb='claude --bare'
alias ccr='claude --resume'
alias ccc='claude --continue'
```

## Custom agent: reviewer

Location: `.claude/agents/reviewer.md`

```markdown
---
name: reviewer
description: Code review, read-only
tools: [Read, Glob, Grep]
model: claude-sonnet-4-6
permissionMode: acceptEdits
maxTurns: 50
---
Rules:
1. Do not modify any files
2. Check: security vulnerabilities, performance, maintainability, naming
3. Severity levels: Critical / Warning / Info
4. Output: table with file path, line number, issue, suggestion
```

## Hook: auto-format on edit (Ruff/Python)

Add to `settings.json` under `"hooks" > "PostToolUse"`:

```json
{
  "type": "command",
  "command": "ruff format $CHANGED_FILES 2>/dev/null && ruff check --fix $CHANGED_FILES 2>/dev/null || true",
  "if": "Edit(*.py)",
  "timeout": 10,
  "async": true
}
```

## Hook: auto-test on src/ edit

```json
{
  "type": "command",
  "command": "npm test -- --bail 2>&1 | tail -20",
  "if": "Edit(src/**/*)",
  "timeout": 60,
  "async": true
}
```

## Init script

Save as `setup-claude.sh`, run once per project:

```bash
#!/bin/bash
[ ! -f "CLAUDE.md" ] && cat > CLAUDE.md << 'EOF'
# claude
- Language: (fill in)
- Framework: (fill in)
- Test: (fill in)
- Read files before modifying
- One task at a time
- On error: analyze, do not retry
EOF
mkdir -p .claude/agents
[ ! -f ".claude/agents/reviewer.md" ] && cat > .claude/agents/reviewer.md << 'EOF'
---
name: reviewer
description: Code review, read-only
tools: [Read, Glob, Grep]
model: claude-sonnet-4-6
maxTurns: 50
---
Do not modify files. Check security, performance, maintainability. Output table.
EOF
[ ! -f ".claude/settings.json" ] && cat > .claude/settings.json << 'EOF'
{"permissions":{"allow":["Read","Glob","Grep","Bash(git:*)","Bash(npm:test)","Bash(npm:run:*)"],"deny":["Bash(rm:-rf:*)","Bash(sudo:*)"]}}
EOF
echo "Done. Run: claude"
```
