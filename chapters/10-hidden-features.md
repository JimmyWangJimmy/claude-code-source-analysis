# 第十章：隐藏功能与未发布特性

> **导读｜读完这章能做什么**
> - 了解 40+ Feature Flags 揭示的产品方向
> - 找到 PROACTIVE/VOICE/ULTRATHINK 等未发布功能
> - 理解 Bun feature() 的编译期死代码消除

## Feature Flags 系统

Claude Code 通过 Bun 的 `feature()` 实现编译期 feature flags。未启用的 flag 对应的代码在构建时被**完全剥离**。

```typescript
const module = feature('FEATURE_NAME')
  ? require('./path.js')
  : null
// Bun 编译时识别 flag 值，false 分支直接删除
```

---

## 40+ Feature Flags

### Agent 相关

| Flag | 推测功能 |
|------|----------|
| `PROACTIVE` | 主动式 Agent — 不等用户指令，自主行动 |
| `KAIROS` | Assistant Tab 相关功能集 |
| `KAIROS_*` | Kairos 子功能 (GitHub webhooks 等) |
| `COORDINATOR_MODE` | 多 Worker 编排模式 |
| `FORK_SUBAGENT` | 隐式 Fork 子 Agent |
| `AGENT_TRIGGERS` | Agent 触发器系统 |

### 分类器

| Flag | 推测功能 |
|------|----------|
| `TRANSCRIPT_CLASSIFIER` | 自动模式 ML 分类器 |
| `BASH_CLASSIFIER` | Bash 命令安全分类器 |

### 实验性功能

| Flag | 推测功能 |
|------|----------|
| `ULTRAPLAN` | 增强型规划模式 |
| `ULTRATHINK` | 扩展思考模式 |
| `TORCH` | 未知 (可能是加速功能) |
| `WEB_BROWSER_TOOL` | 网页浏览器工具 |
| `VOICE_MODE` | 语音输入模式 |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索发现 |
| `CACHED_MICROCOMPACT` | 增量压缩优化 |
| `WORKFLOW_SCRIPTS` | 工作流脚本自动化 |

### 基础设施

| Flag | 推测功能 |
|------|----------|
| `BRIDGE_MODE` | IDE 桥接模式 |
| `CCR_*` | Cloud Code Runner 系列 |
| `DAEMON` | 守护进程模式 |
| `HISTORY_SNIP` | 历史压缩优化 |
| `UDS_INBOX` | Unix Domain Socket 收件箱 |
| `MONITOR_TOOL` | 监控工具 |
| `EXTRACT_MEMORIES` | 自动记忆提取 |
| `TEAMMEM` | 团队记忆同步 |

---

## 未发布命令

| 命令 | 功能 |
|------|------|
| `/proactive` | 主动模式: Agent 自主执行任务 |
| `/assistant` | Kairos assistant tab |
| `/brief` | 简短快速响应 |
| `/bughunter` | 自动化 bug 搜索 |
| `/autofix-pr` | 自动修复 PR 问题 |
| `/chrome` | Chrome 浏览器集成 |
| `/fork` | Git fork 工作流 |
| `/team` | 团队 Agent 管理 |
| `/good-claude` | 正向反馈 (可能用于 RLHF) |
| `/teleport` | 会话转移 |

---

## 值得关注的隐藏功能

### 语音输入系统 (Feature: VOICE_MODE)

```
音频捕获:
  → 原生模块: audio-capture-napi (macOS, Linux, Windows)
  → 备选: SoX rec 或 ALSA arecord (Linux)

参数:
  → 采样率: 16000 Hz, 单声道
  → 静音检测: 2秒持续, 3% 阈值

加载策略:
  → 懒加载: 首次语音按键时才加载
  → 避免启动时冻结
```

### Buddy 系统 (伴侣精灵)

终端中的虚拟伴侣/宠物系统:
- `CompanionSprite.tsx` — React 组件渲染精灵图形
- `companion.ts` — 伴侣逻辑
- `sprites.ts` — 精灵图形定义
- `useBuddyNotification.tsx` — 通知 hook

### Undercover 模式

```typescript
isUndercover()  // 抑制特定功能的函数
```

用于测试或特殊场景，控制某些功能的可见性。

### 开发通道

```
--dangerously-load-development-channels
```

允许加载未测试的 MCP 开发通道，per-entry dev flag 防止白名单绕过。

### Client Attestation

```
cch=00000  // 客户端证明头，被原生 HTTP 栈覆写
```

版本指纹识别 + 工作负载 QoS 路由提示。

---

## GrowthBook & 分析

### Analytics 事件

```
tengu_agent_memory_loaded      — Agent 记忆加载
tengu_editor_mode_changed      — 编辑器模式切换
tengu_coordinator_mode_switched — Coordinator 模式切换
tool_use                       — 工具使用
tool_error                     — 工具错误
permission_decision            — 权限决策
agent_spawn                    — Agent 生成
```

### PII 保护类型

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
// 类型名本身就是"我已检查不含代码/路径"的签名
// 用 TypeScript 类型系统防止 PII 泄露
```

---

## 设计要点

1. **Feature Flags 实现死代码消除** — 编译期移除，产物不含未启用代码
2. **Proactive 模式** — Agent 自主性的最高级别，代表未来演进方向
3. **Voice + Browser** — 向多模态 Agent 演进
4. **Buddy 精灵** — 终端工具也能有趣味性
5. **Analytics 类型安全** — TypeScript 类型系统防止 PII 泄露
