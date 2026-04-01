# 第六章 权限与安全系统

> Claude Code 的权限与安全子系统实现了多层防御，是整个代码库中复杂度最高的部分之一。

## 权限模式

### 外部可用模式

| 模式 | 行为 |
|------|------|
| `default` | 标准提示模式 — 不确定时询问用户 |
| `acceptEdits` | 自动接受文件编辑 |
| `plan` | 规划模式 — 特殊处理 |
| `bypassPermissions` | 绕过所有权限检查 (危险) |
| `dontAsk` | 不询问，直接拒绝 |

### 内部/Feature-Gated 模式

| 模式 | 行为 |
|------|------|
| `auto` | ML 分类器自动批准 (Feature: `TRANSCRIPT_CLASSIFIER`) |
| `bubble` | 权限冒泡到父 Agent (子 Agent 用) |

---

## 6 层权限决策链

```
工具调用请求
  │
  ▼
[1] Deny 规则检查 (最高优先级)
  │ → 命中 deny → 立即拒绝
  ▼
[2] Allow 规则检查
  │ → 命中 allow → 立即允许
  ▼
[3] Ask 规则检查
  │ → 命中 ask → 跳到用户提示
  ▼
[4] Hook 执行 (executePermissionRequestHooks)
  │ → Hook 返回 allow/deny → 采纳
  ▼
[5] ML 分类器自动批准 (如启用)
  │ → 分类器判断安全 → 自动允许
  │ → 分类器不确定 → 继续
  ▼
[6] 用户交互提示 / 自动拒绝
  │ → 根据模式决定
  ▼
结果: allow / deny
```

---

## 权限规则系统

### 规则格式

```
"ToolName(content)"     # 工具名 + 内容匹配
"bash git*"             # 通配符匹配
"file /tmp/**"          # 路径通配符
```

### 规则来源 (按优先级)

| 来源 | 说明 |
|------|------|
| `hooks` | 用户在 hooks 设置中定义 |
| `settings` | 通过 CLI config 持久化 |
| `managed` | Anthropic 管理的安全策略 |
| `plugin` | 插件提供的规则 |
| `session` | 会话级临时规则 |
| `command` | 命令级规则 |
| `cliArg` | CLI 参数规则 |

### Hook 模式

```typescript
{
  if_conditions: /* 工具/输入的模式匹配 */,
  then_behavior: 'allow' | 'deny' | 'ask',
  permanent: true | false  // 是否保存到设置
}
```

---

## Bash 安全系统 (105KB)

这是安全系统中复杂度最高的部分。

### 危险模式检测

通过 Bash AST 解析 (`utils/bash/ast.ts`) 进行模式匹配:

| 危险类别 | 示例模式 |
|----------|---------|
| 文件系统破坏 | `rm -rf /`, `dd if=/dev/zero` |
| 凭证窃取 | 访问 `.env`, credentials 文件 |
| 数据外泄 | `curl` 发送敏感数据 |
| 系统破坏 | `chmod 777`, `kill -9` |
| 环境修改 | 修改 PATH, 安装全局包 |

### 命令语义分类

```
搜索命令: grep, rg, find, ag       → 安全, UI 可折叠
读取命令: cat, head, tail, less     → 安全, UI 可折叠
列表命令: ls, dir, tree             → 安全, UI 可折叠
语义中立: echo, printf              → 管道中忽略
修改命令: sed, awk                  → 需要验证
危险命令: rm, dd, chmod             → 需要权限
```

### Sed 命令验证

专门的 sed 解析器验证:
- 是否为破坏性编辑
- 目标文件是否在允许范围内
- 编辑模式是否安全

---

## 自动模式分类器

### 2 阶段决策 (Feature: TRANSCRIPT_CLASSIFIER)

```
阶段 1 (快速):
  → 基于 tool_use block 的快速分类
  → 高置信度 → 直接决策

阶段 2 (思考):
  → XML + 思维链推理
  → 完整上下文分析
  → 返回 allow/deny + 置信度 (high/medium/low)
```

---

## 拒绝追踪与降级

```
记录连续拒绝次数
  → 超过阈值 N
    → 不再逐一询问
    → 提示用户授予批量权限
```

> **分析:** 这是对"权限疲劳"问题的应对——频繁弹权限确认让用户烦躁。检测到连续拒绝模式后，系统升级为批量权限请求。

---

## 权限处理器

| 处理器 | 场景 |
|--------|------|
| `interactiveHandler` | REPL 模式: UI 对话框 |
| `coordinatorHandler` | 后台 Agent: 无 UI 时自动拒绝 |
| `swarmWorkerHandler` | 多 Agent 团队协作 |

---

## 安全设计要点

1. **防御纵深**: 不是单点检查，而是 6 层决策链
2. **AST 级分析**: 不是正则匹配命令字符串，而是解析 Bash AST
3. **权限非二元**: 不是简单的 allow/deny，支持 ask、分类器、Hook
4. **授权不扩散**: 一次授权不等于永久授权
5. **子 Agent 权限隔离**: 子 Agent 获得受限的工具集
6. **分类器兜底**: ML 模型作为人工审批前的过滤器
