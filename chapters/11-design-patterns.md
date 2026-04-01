# 第十一章：关键设计模式

本章梳理 Claude Code 源码中解决 AI Agent 特有难题的设计模式。

---

## 模式 1: Feature-Gated 死代码消除

```typescript
const module = feature('FEATURE_NAME')
  ? require('./path.js')
  : null
```

**问题**: 大量实验性功能需要共存于同一代码库
**方案**: 编译期通过 Bun bundler 识别 feature flag 值，移除未启用分支
**效果**: 生产产物不包含实验代码，无运行时开销

---

## 模式 2: Context Object (ToolUseContext)

```typescript
// 不是: function callTool(options, abort, state, perms, ui) {}
// 而是: function callTool(context: ToolUseContext) {}
```

**问题**: 工具执行需要大量上下文参数
**方案**: 单一上下文对象，浅拷贝即可创建子 Agent 上下文
**优势**: 克隆简单、扩展容易、覆盖灵活

---

## 模式 3: 大结果磁盘持久化

**问题**: LLM 上下文窗口有限，工具可能返回巨大结果
**方案**: 大结果存盘 + 哈希去重，只给模型摘要
**关键**: 模型可以在后续请求中引用磁盘文件

---

## 模式 4: 并行预取

```typescript
// main.tsx — 在任何 import 之前触发
startMdmRawRead()
startKeychainPrefetch()
```

**问题**: CLI 启动时间敏感
**方案**: 可并行的 I/O 操作提前到 import 解析之前
**效果**: 模块求值与网络 IO 同时进行

---

## 模式 5: Memoized System Prompt Sections

```typescript
systemPromptSection('key', () => computeExpensiveContent())
```

**问题**: System prompt 每次 API 调用都需要，部分内容计算昂贵
**方案**: 按段缓存，显式清除时失效
**边界**: `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 分隔可缓存 vs 必须更新

---

## 模式 6: 懒加载重型模块

被懒加载的模块:
- OpenTelemetry + gRPC (遥测)
- React/Ink 组件 (UI)
- 分析/Analytics
- 音频捕获 (voice)

**效果**: 大幅减少 CLI 启动时间

---

## 模式 7: 权限拒绝追踪与降级

```
连续拒绝计数器
  → 超过阈值 N
    → 不再逐一询问
    → 提示用户授予批量权限
```

**问题**: 用户权限疲劳
**方案**: 检测连续拒绝模式，升级为批量权限请求
**要点**: 粒度过细的权限控制可能比没有权限更糟糕

---

## 模式 8: Fork Agent Prompt Cache 共享

```
父 Agent Prompt = P
Fork Agent Prompt = P (字节级相同)
  → Anthropic API prompt cache 命中!
```

**问题**: 子 Agent API 调用重复浪费 prompt Token
**方案**: Fork Agent 保持 prompt 与父完全一致
**要点**: 不是优化传输，而是利用 API 侧缓存机制

---

## 模式 9: 流式工具执行器

```
StreamingToolExecutor
  → 接收模型流式输出
  → 识别 tool_use blocks
  → 并发启动工具执行
  → 聚合结果
```

**问题**: 工具执行是顺序阻塞的
**方案**: 在模型流式响应过程中就开始执行工具
**效果**: 减少用户等待时间

---

## 模式 10: 多层权限决策链

```
Deny → Allow → Ask → Hook → Classifier → Default
```

**问题**: 权限决策不是简单的 yes/no
**方案**: 6 层决策链，每层有不同来源和优先级
**优势**: 高优先级快速短路，ML 分类器作为过滤器

---

## 模式 11: 命令语义分类

```
搜索: grep, rg, find     → 安全, UI 折叠
读取: cat, head, tail     → 安全, UI 折叠
中立: echo, printf        → 管道中忽略
危险: rm, dd, chmod       → 需要权限
```

**问题**: 不是所有 Bash 命令同等危险
**方案**: 基于 AST 的语义分析
**效果**: 安全命令折叠 UI，危险命令触发检查

---

## 模式 12: Git 状态快照

**问题**: 频繁获取 Git 状态开销大且导致缓存失效
**方案**: 快照一次，会话内不更新
**权衡**: 牺牲实时性换取性能和缓存命中率

---

## 模式 13: buildTool() 工厂

```typescript
const myTool = buildTool({
  name: 'MyTool',
  inputSchema: z.object({ ... }),
  call: async (input, context) => { ... },
  // 其他方法使用默认实现
})
```

**问题**: 40+ 工具都有大量通用代码
**方案**: 工厂函数提供合理默认值

---

## 模式 14: 分析类型安全

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
```

**问题**: 分析事件可能包含 PII
**方案**: 用 TypeScript 类型名作为"已审查"签名

---

## 模式 15: Hybrid Transport 降级

```
WebSocket (全双工) → SSE (单向推送) → HTTP 轮询
```

**问题**: 不同网络环境的连接能力不同
**方案**: 渐进降级 + 指数退避 + 随机抖动
