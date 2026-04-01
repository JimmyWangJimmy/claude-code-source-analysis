# 第三章 整体架构

## 入口与启动流程

### main.tsx (4,683 行)

入口文件基于 Commander.js 的 CLI 解析器 + React/Ink 渲染器初始化。

> **并行预取优化:** 启动时同时发起 MDM 设置读取、Keychain 预取、GrowthBook 初始化——在 JavaScript 模块求值的同时，网络/IO 已在并行进行。

```typescript
// main.tsx — 作为 side-effects 在其他 import 之前触发
startMdmRawRead()
startKeychainPrefetch()
```

## 核心运行循环

<img src="../assets/architecture-flow.svg" alt="核心运行循环" width="100%">

## 目录结构

```
src/
├── main.tsx                 # 入口: Commander.js CLI + React/Ink 渲染
├── QueryEngine.ts           # 核心引擎: LLM调用、流式响应、重试、Token计数
├── Tool.ts                  # 工具类型定义与基础框架
├── commands.ts              # 命令注册中心 (60+ 命令)
├── context.ts               # 系统/用户上下文收集 (git状态、CLAUDE.md)
├── cost-tracker.ts          # Token 成本跟踪
│
├── tools/                   # 40+ 工具实现
│   ├── BashTool/            # Shell 命令执行 (161KB, 最复杂)
│   ├── AgentTool/           # 子 Agent 生成 (235KB)
│   ├── FileEditTool/        # 文件编辑
│   ├── FileReadTool/        # 文件读取 (PDF/图片/Notebook)
│   ├── GlobTool/            # 文件模式匹配
│   ├── GrepTool/            # ripgrep 内容搜索
│   └── ...                  # WebFetch, MCPTool, LSPTool 等
│
├── commands/                # 60+ 斜杠命令
├── services/                # 外部服务集成 (API, MCP, OAuth, LSP)
├── bridge/                  # IDE 双向通信桥 (VS Code / JetBrains)
├── coordinator/             # 多 Agent 编排
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── memdir/                  # 持久化记忆
├── hooks/                   # React hooks + 权限钩子
├── tasks/                   # 后台任务管理
└── utils/                   # 工具函数 (权限、Bash AST、System Prompt)
```

## 关键架构决策

### 1. 选择 Bun 运行时

选择 Bun 而非 Node.js，获得:
- 更快的启动速度
- 原生 TypeScript 支持
- `bun:bundle` feature flags 实现编译期死代码消除

### 2. React + Ink 终端 UI

用 React 的组件模型来构建终端界面:
- 组件化的工具结果渲染
- 声明式的权限对话框
- 流式输出的状态管理

### 3. 懒加载策略

重型模块通过动态 `import()` 延迟加载:
- OpenTelemetry、gRPC (遥测)
- React/Ink 组件 (UI)
- 分析/Analytics 模块
- Feature-gated 子系统

### 4. Feature Flags 死代码消除

```typescript
const module = feature('FEATURE_NAME')
  ? require('./path.js')
  : null
```

Bun 编译时识别 feature flag 值，未启用的功能代码**完全不会出现**在最终产物中。

### 5. 大型工具结果的磁盘持久化

工具返回的大型结果不直接发送给 API (会撑爆上下文窗口):
1. 保存到磁盘 (`~/.claude/tool-results/`)
2. 只给 API 发送摘要预览
3. 通过内容哈希去重

---

## QueryEngine.ts 深度分析

**1,295 行**的核心引擎，负责:

| 职责 | 实现 |
|------|------|
| 消息规范化 | `normalizeMessagesForAPI()` — 清理 ANSI 码、规范化 content blocks |
| Tool Use/Result 配对 | 确保每个 tool_use 都有对应的 tool_result |
| 流式执行 | `StreamingToolExecutor` 并发执行多个工具 |
| 自动压缩 | 上下文超 token 阈值时触发 auto-compact |
| 重试逻辑 | 指数退避，区分可重试/永久错误 |
| 速率限制 | `CLAUDE_AI_LIMITS` 服务集成 |
| 会话持久化 | 消息历史、成本状态保存到磁盘 |
| 思考模式 | thinking configuration 支持 |

---

## 成本追踪系统

```typescript
// cost-tracker.ts 追踪的维度
{
  inputTokens: number      // 按模型分计
  outputTokens: number     // 按模型分计
  cacheReadTokens: number  // 缓存命中
  cacheWriteTokens: number // 缓存写入
  apiDuration: number      // API 耗时 (含/不含重试)
  totalCostUSD: number     // 总费用 (USD)
  linesAdded: number       // 文件编辑统计
  linesRemoved: number
  webSearchRequests: number
}
```

集成 OpenTelemetry meters，支持会话间恢复成本状态。
