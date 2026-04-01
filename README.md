# Claude Code 源码深度分析

> **基于 2026-03-31 通过 npm source map 泄露的 Claude Code TypeScript 源码快照**

---

## 这本书是什么

2026 年 3 月 31 日，安全研究者 [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 发现 Claude Code 的 npm 发布包中包含了 `.map` source map 文件，指向 Anthropic R2 存储桶中的未混淆 TypeScript 源码。这意味着 Claude Code 的完整源码对公众可访问。

本书是对这份源码的**系统性深度分析**，覆盖：

- 整体架构与运行循环
- System Prompt 的动态组装与策略
- 40+ 工具的设计模式
- 6 层权限决策链与 Bash AST 安全分析
- 4 种多智能体协作模式
- 持久化记忆与上下文压缩
- IDE 桥接与远程控制
- 40+ Feature Flags 与未发布功能
- 15 个关键设计模式
- 29 条可行动洞察

## 数字一览

| 指标 | 数值 |
|------|------|
| 源文件数 | ~1,902 |
| 代码行数 | 512,000+ |
| 工具实现 | 40+ |
| 斜杠命令 | 60+ |
| Feature Flags | 40+ |
| 语言 | TypeScript |
| 运行时 | Bun |
| 终端 UI | React + Ink |

## 适合谁读

- **AI Agent 开发者** — 学习生产级 Agent 的工程实践
- **安全研究者** — 研究 Agent 安全的多层防御设计
- **产品经理** — 了解顶级 AI 编码工具的功能全景
- **架构师** — 参考多智能体协作、插件、记忆系统的设计
- **学生/研究者** — 学习大规模 TypeScript 项目的工程模式

## 声明

- 本分析仅用于**教育与安全研究**目的
- 原始 Claude Code 源码版权归 **Anthropic** 所有
- 本书不隶属于、不被 Anthropic 授权或维护
