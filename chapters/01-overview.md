# 价值定位与技术栈

## 这份源码能带来什么价值?

### 1. 理解顶级 AI Agent 的工程架构

Claude Code 是目前最强大的 AI 编码 Agent 之一。源码完整揭示了:

- 从用户输入到模型调用到工具执行的**完整请求链路**
- **多智能体协作**的生产级实现 (coordinator / swarm / fork)
- **权限与安全系统**的多层设计
- **IDE 集成**的双向通信桥接方案
- **持久化记忆**跨会话的工程实现

### 2. 学习 System Prompt 工程

源码中包含了 Anthropic 精心打磨的 system prompt，展示了:

- 如何通过 prompt 塑造 Agent 行为
- 如何动态组装上下文 (git 状态、用户记忆、工具列表)
- 如何平衡安全性与实用性的 prompt 策略

### 3. 安全研究与防御参考

- 工具权限的分层决策引擎
- Bash 命令的 AST 级安全分析
- 危险模式检测与自动拦截
- 沙盒隔离机制

### 4. 产品设计灵感

- 40+ 工具的设计模式
- 60+ 斜杠命令的用户交互
- 技能系统与插件架构
- 成本追踪与 Token 预算管理

---

## 技术栈速览

| 类别 | 技术 |
|------|------|
| 运行时 | [Bun](https://bun.sh) |
| 语言 | TypeScript (strict) |
| 终端 UI | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI 解析 | [Commander.js](https://github.com/tj/commander.js) |
| Schema 校验 | [Zod v4](https://zod.dev) |
| 代码搜索 | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| 协议 | [MCP SDK](https://modelcontextprotocol.io), LSP |
| API | [Anthropic SDK](https://docs.anthropic.com) |
| 遥测 | OpenTelemetry + gRPC |
| Feature Flags | GrowthBook |
| 认证 | OAuth 2.0, JWT |

---

## 泄露的背景

[Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 于 2026 年 3 月 31 日公开指出:

> **"Claude code source code has been leaked via a map file in their npm registry!"**

npm 包中的 `.map` 文件引用了存储在 Anthropic R2 存储桶中的未混淆 TypeScript 源码，使整个 `src/` 目录可公开下载。
