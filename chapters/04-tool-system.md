# 工具系统：40+ 工具的设计与实现

## 工具框架核心: Tool.ts

### 泛型工具类型

```typescript
Tool<Input, Output, Progress>
```

每个工具定义:
- **输入 Schema** — Zod 校验
- **权限模型** — `checkPermissions()`
- **执行逻辑** — `call()`
- **UI 渲染** — `renderToolResultMessage()` / `renderToolUseMessage()`
- **进度回调** — `onProgress`

### buildTool() 工厂模式

提供合理默认值的工具构建工厂，避免每个工具重复定义通用方法。

### ToolUseContext — 上下文对象模式

单一参数对象替代多参数:

```typescript
{
  options,           // 配置选项
  abortController,   // 取消控制器
  stateManager,      // 状态管理
  permissionContext,  // 权限上下文
  uiCallbacks,       // UI 回调
  readFileTimestamps, // 文件读取时间戳追踪
}
```

好处: 子 Agent 上下文克隆时只需浅拷贝一个对象。

---

## 核心工具详解

### BashTool — 最复杂的工具 (161KB)

**文件组成:**
- `BashTool.tsx` — 主实现
- `bashPermissions.ts` (101KB) — 权限规则匹配
- `bashSecurity.ts` (105KB) — 危险命令检测

**关键特性:**

| 特性 | 实现 |
|------|------|
| 命令拆分 | 识别 `&&`, `\|\|`, `\|`, `;` 操作符 |
| 沙盒支持 | `SandboxManager` 隔离执行 |
| 只读检测 | 阻止写操作尝试 |
| 超时处理 | 默认 vs 最大超时 |
| 后台自动化 | 超阈值后自动转后台 |
| 进度流 | 2 秒后开始显示进度 |
| 搜索/读取折叠 | 检测 grep/find/ls/cat 等模式进行 UI 压缩 |
| Sed 解析器 | 验证 sed 编辑命令的安全性 |

**危险模式检测:**

```
rm -rf, dd if=/dev/zero, 凭证窃取, 数据外泄
→ 通过 Bash AST 解析进行模式匹配 (utils/bash/ast.ts)
→ 不是简单字符串匹配
```

---

### AgentTool — 第二复杂 (235KB)

**子 Agent 生成系统:**
- 子 Agent 类型通过 `loadAgentsDir.ts` 动态加载
- 支持模型覆盖 (sonnet / opus / haiku)
- 隔离模式: `worktree` (git checkout) 或 `remote` (CCR 云端)
- 后台执行 + 完成通知

**Agent 记忆系统:**
- 每个 Agent 类型有独立记忆快照
- 父子上下文隔离
- 记忆附件支持嵌套 Agent

**Fork 子 Agent (Feature: FORK_SUBAGENT):**
- 子 Agent 继承父的**完整上下文和 system prompt**
- 利用字节级相同的 fork children 实现 **prompt cache 共享**
- 防递归 fork 保护 (`FORK_BOILERPLATE_TAG`)

> **天才设计:** Fork Agent 的 prompt 与父完全一致 → Anthropic API prompt cache 命中 → 不需要重新编译 system prompt → 大幅节省 Token 和延迟。

---

### FileReadTool (40KB)

**多格式支持:**

| 格式 | 处理 |
|------|------|
| 文本文件 | 行号显示、行尾检测 |
| PDF | 页面范围提取、Token 预算 |
| 图片 | 智能降采样、Token 限制 |
| Jupyter Notebook | Cell 解析、输出展示 |
| 二进制 | 检测并拒绝 |
| 设备文件 | 阻止 (`/dev/zero` 等) |

---

### FileEditTool (21KB)

**安全特性:**
- 文件大小限制: 1 GiB
- 编辑期间文件修改检测 (防冲突)
- 行尾格式保留 (CRLF vs LF)
- Git diff 追踪支持撤销
- 文件编辑历史: `fileHistoryTrackEdit()`

---

## 完整工具清单

### 文件操作

| 工具 | 功能 |
|------|------|
| FileReadTool | 读取文件 (多格式) |
| FileWriteTool | 创建/覆写文件 |
| FileEditTool | 精确字符串替换编辑 |
| GlobTool | 文件模式搜索 |
| GrepTool | 内容搜索 (ripgrep) |
| NotebookEditTool | Jupyter 编辑 |

### 执行

| 工具 | 功能 |
|------|------|
| BashTool | Shell 命令执行 |
| AgentTool | 子 Agent 生成 |
| SkillTool | 技能执行 |
| REPLTool | REPL 交互 |

### 网络

| 工具 | 功能 |
|------|------|
| WebFetchTool | URL 内容获取 |
| WebSearchTool | 网络搜索 |

### 协议

| 工具 | 功能 |
|------|------|
| MCPTool | MCP 服务器工具调用 |
| LSPTool | LSP 集成 |

### 任务管理

| 工具 | 功能 |
|------|------|
| TaskCreateTool | 创建任务 |
| TaskUpdateTool | 更新任务 |
| TaskListTool | 列出任务 |
| TaskStopTool | 停止任务 |
| TaskOutputTool | 获取任务输出 |

### 交互

| 工具 | 功能 |
|------|------|
| AskUserQuestion | 向用户提问 |
| SendMessageTool | Agent 间消息传递 |

### 工作流

| 工具 | 功能 |
|------|------|
| EnterPlanModeTool | 进入规划模式 |
| ExitPlanModeTool | 退出规划模式 |
| EnterWorktreeTool | 创建 Git worktree 隔离 |
| ExitWorktreeTool | 退出 worktree |
| CronCreateTool | 创建定时触发器 |
| SleepTool | 主动模式等待 |

### 团队

| 工具 | 功能 |
|------|------|
| TeamCreateTool | 创建团队 Agent |
| TeamDeleteTool | 删除团队 Agent |
| SyntheticOutputTool | 结构化输出生成 |

---

## 工具结果大小管理

这是一个巧妙的设计:

```
工具返回结果
  → 检查大小 > maxResultSizeChars ?
    → Yes: 保存到 ~/.claude/tool-results/
           API 只收到摘要预览 + 文件路径
           内容哈希去重
    → No:  直接发送给 API
  → 会话级内容替换预算追踪
```

> **核心问题:** LLM 上下文窗口有限，但工具可能返回巨大结果。大型工具结果存盘，只给模型摘要——完美解决。
