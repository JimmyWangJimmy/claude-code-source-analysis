# 拆解 Claude Code

**从 Claude Code 51 万行泄露源码中，提炼 AI Agent 设计方法与提效配置。**

---

2026 年 3 月 31 日，Claude Code 的 npm 包中暴露了 source map，指向完整的未混淆 TypeScript 源码——1,902 个文件，512,000 行代码。这本书把源码拆开，提炼出能直接用的东西。

| 数字 | |
|------|-|
| **14 章** | 从使用技巧到架构拆解到多 Agent 设计方法论 |
| **13 张 SVG** | 架构图、决策链、隔离矩阵等可视化 |
| **29 条行动建议** | 覆盖 Prompt / 架构 / 安全 / 性能 / 记忆 |
| **一键复制配置** | settings.json + CLAUDE.md + Hook + 初始化脚本 |

---

### 三种读法

**拿来就用** → 直接翻 [附录：一键复制配置集](chapters/14-copy-paste-configs.md)，复制粘贴立刻生效

**日常提效** → 从 [第 1 章：高效使用手册](chapters/01-power-user-guide.md) 开始，快捷键、隐藏参数、权限配置

**系统学习** → 看 [导读](chapters/00-guide.md) 了解全书结构，按顺序读完 14 章

---

### 声明

本书仅用于教育与安全研究。原始源码版权归 Anthropic 所有。
