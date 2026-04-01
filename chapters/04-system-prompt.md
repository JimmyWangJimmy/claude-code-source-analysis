# 第四章 System Prompt 工程

> Claude Code 的 system prompt 是 Anthropic 投入大量调优工作的产物。理解它的结构和策略，对构建 AI Agent 有直接参考价值。

## Prompt 身份定义

```
默认模式:
  "You are Claude Code, Anthropic's official CLI for Claude."

SDK Agent 模式 (非交互 + append):
  "You are Claude Code, Anthropic's official CLI for Claude,
   running within the Claude Agent SDK."

纯 Agent 模式:
  "You are a Claude agent, built on Anthropic's Claude Agent SDK."
```

## 动态 Prompt 组装架构

System Prompt 不是一个静态字符串，而是由多个**缓存段**动态拼装:

```
┌─────────────────────────────────┐
│   STATIC BOUNDARY (全局可缓存)    │
│  ┌───────────────────────────┐  │
│  │ 身份声明                    │  │
│  │ 工具使用指南                 │  │
│  │ 安全规则                    │  │
│  │ 代码编辑规范                 │  │
│  │ 输出风格                    │  │
│  └───────────────────────────┘  │
│                                 │
│   DYNAMIC BOUNDARY (用户特定)    │
│  ┌───────────────────────────┐  │
│  │ 当前日期                    │  │
│  │ Git 状态快照                │  │
│  │ CLAUDE.md 用户记忆           │  │
│  │ 可用工具列表                 │  │
│  │ Feature flag 特定指令        │  │
│  │ Coordinator 模式指令         │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

关键设计: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记分隔静态与动态内容，使 Anthropic API 的 **prompt caching** 能缓存静态部分。

## Prompt 优先级系统

| 优先级 | 类型 | 说明 |
|--------|------|------|
| 1 (最高) | Override Prompt | 完全替换所有默认 prompt |
| 2 | Coordinator Prompt | 多 Agent 编排模式 |
| 3 | Agent Prompt | 专用 Agent / Proactive 模式附加 |
| 4 | Custom Prompt | CLI `--prompt` 参数 |
| 5 | Default Prompt | 默认 prompt |
| 6 | Append Prompt | 始终附加 (Override 除外) |

---

## 核心 Prompt 策略分析

### 策略 1: 强调"先读后改"

```
"Do not propose changes to code you haven't read."
"If a user asks about or wants you to modify a file, read it first."
"Understand existing code before suggesting modifications."
```

> **分析:** 通过 prompt 约束 LLM 的 "幻觉式编码"——强制模型在修改前建立完整上下文。

### 策略 2: 抑制过度工程

```
"Avoid over-engineering. Only make changes that are directly requested."
"Don't add features, refactor code, or make 'improvements' beyond what was asked."
"Don't add docstrings, comments, or type annotations to code you didn't change."
"Three similar lines of code is better than a premature abstraction."
```

> **分析:** Anthropic 观察到 LLM 有过度工程化的倾向 (加注释、加类型、提取函数)，用 prompt 显式抑制。

### 策略 3: 安全操作的确认机制

```
"For actions that are hard to reverse, affect shared systems,
 or could be risky, check with the user before proceeding."
"A user approving an action once does NOT mean
 they approve it in all contexts."
"Authorization stands for the scope specified, not beyond."
```

> **分析:** 权限不是二元的。一次授权不等于永久授权——防止 Agent 越权的关键设计。

### 策略 4: 不给时间估计

```
"Avoid giving time estimates or predictions for how long tasks will take."
```

> **分析:** LLM 无法准确估时，不说比说错好。

### 策略 5: 安全编码意识

```
"Be careful not to introduce security vulnerabilities such as
 command injection, XSS, SQL injection, and other OWASP top 10
 vulnerabilities."
```

### 策略 6: 不暴力重试

```
"If your approach is blocked, do not attempt to brute force
 your way to the outcome."
```

> **分析:** LLM Agent 常见反模式——遇到错误就无脑重试。Prompt 明确禁止这种行为。

### 策略 7: 避免向后兼容 Hack

```
"Avoid backwards-compatibility hacks like renaming unused _vars,
 re-exporting types, adding // removed comments for removed code."
```

---

## 上下文收集: context.ts

```typescript
getSystemContext()  // Git 状态快照 (memoized, 会话内不更新)
getUserContext()    // CLAUDE.md + 记忆文件 (自动发现、缓存)
```

关键实现细节:
- Git 状态在会话开始时快照，**整个会话期间不会更新**
- 状态文本限制 `MAX_STATUS_CHARS = 2000` 字符，超出截断并警告
- CCR (Cloud Remote) 模式下跳过 git 状态获取

---

## 记忆指令

```
"MEMORY.md is always loaded into your conversation context
 — lines after 200 will be truncated, so keep it concise"

"Create separate topic files for detailed notes
 and link from MEMORY.md"

"Update or remove memories that turn out to be wrong or outdated"
```

### 记忆分类法

| 类型 | 隐私 | 用途 |
|------|------|------|
| `user` | 始终私有 | 用户角色、目标、偏好 |
| `feedback` | 默认私有 | 方法指导 |
| `project` | 私有或团队 | 进行中的工作、目标 |
| `reference` | 通常团队 | 外部系统指针 (Linear, Slack) |

### 记忆漂移警告

```
"Memories are snapshots — must verify against current state before acting"
"Read before recommending from memory (check files exist, grep functions)"
```

> **分析:** 记忆可能过期，prompt 要求模型在使用记忆前先验证当前状态。
