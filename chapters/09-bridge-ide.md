# 第九章：IDE 桥接与远程控制

## Bridge 系统概述

Bridge 是 Claude Code CLI 与 IDE 扩展 (VS Code / JetBrains) 之间的双向通信层。

### 架构

```
┌─────────────────┐        ┌─────────────────┐
│   VS Code /     │        │   Claude Code   │
│   JetBrains     │◄─────►│   CLI           │
│   Extension     │  WS    │                 │
│                 │ + REST  │                 │
└─────────────────┘        └─────────────────┘
```

---

## 核心组件

| 文件 | 职责 |
|------|------|
| `bridgeMain.ts` | Bridge 主循环，指数退避重连 |
| `bridgeApi.ts` | OAuth 认证 API 客户端，401 自动重试 |
| `bridgeMessaging.ts` | 消息协议定义与处理 |
| `bridgePermissionCallbacks.ts` | 权限回调桥接 |
| `bridgeConfig.ts` | Bridge 配置管理 |
| `bridgeUI.ts` | UI 层接口 |
| `bridgePointer.ts` | 指针/光标同步 |
| `replBridge.ts` | REPL 会话桥接 |
| `sessionRunner.ts` | 会话生成与并发管理 |
| `jwtUtils.ts` | JWT 认证工具 |
| `trustedDevice.ts` | 可信设备令牌 |

---

## 多层认证

| 层 | 机制 | 用途 |
|----|------|------|
| 1 | Bearer Token (OAuth) | API 调用，401 自动刷新 |
| 2 | Trusted Device Token | 可信设备免重复认证 |
| 3 | Session Ingress Token | WebSocket 连接认证 |
| 4 | JWT | 短时令牌用于敏感操作 |

---

## 会话管理

### Worker 类型

| 类型 | 说明 |
|------|------|
| `claude_code` | CLI Worker |
| `claude_code_assistant` | Assistant Tab Worker |

### Spawn 模式

| 模式 | 行为 |
|------|------|
| `single-session` | 单会话模式 |
| `worktree` | Git worktree 隔离 |
| `same-dir` | 同目录复用 |

### 并发控制
- GrowthBook Gate: `tengu_ccr_bridge_multi_session`
- 默认上限: **32 个并发会话**
- 可配置超时: 默认 24 小时

---

## 通信传输层

| 传输 | 用途 |
|------|------|
| `WebSocketTransport.ts` | WebSocket 双向通信 |
| `SSETransport.ts` | Server-Sent Events 单向推送 |
| `HybridTransport.ts` | 混合传输 (WS + SSE 降级) |
| `SerialBatchEventUploader.ts` | 批量事件上传 |
| `WorkerStateUploader.ts` | Worker 状态同步 |
| `ccrClient.ts` | CCR 客户端 |

### 传输降级策略

```
优先: WebSocket (全双工)
  ↓ 失败
降级: SSE (单向推送 + HTTP 请求)
  ↓ 失败
兜底: HTTP 轮询 (指数退避 + 随机抖动防惊群)
```

---

## FlushGate 模式

确保消息发送完成:

```
开始操作 → 获取 FlushGate
  → 执行操作
  → 等待所有消息 flush 完成
  → 释放 FlushGate
```

用于保证关键操作 (如提交、推送) 的消息不丢失。

---

## 设计要点

1. **混合传输降级** — WebSocket → SSE → 轮询，适应各种网络环境
2. **指数退避重连** — 断网时不会对自身服务器造成过大压力
3. **多层认证** — OAuth + 设备信任 + 会话令牌 + JWT，纵深防御
4. **32 并发会话** — 支持团队场景
5. **FlushGate** — 保证消息不丢失的可靠模式
