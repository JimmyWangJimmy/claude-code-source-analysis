# 附录：一键复制配置集

> **导读｜这页干什么用**
> - 以下所有配置直接复制粘贴即可使用
> - 分三档：保守版（安全优先）、日常版（平衡）、全开版（效率最大化）
> - 每段配置标注了作用和风险等级

---

## 一、settings.json（核心配置）

文件位置：`~/.claude/settings.json`（全局）或 `.claude/settings.json`（项目级）

### 保守版 — 只放行读取和搜索类操作

适合：刚开始用、对安全性要求高、团队共享项目

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(rm:-rf:*)",
      "Bash(sudo:*)",
      "Bash(chmod:777:*)"
    ]
  }
}
```

### 日常版 — 大部分场景不再弹确认

适合：个人项目、日常开发、已经熟悉 Claude Code 的行为模式

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git:*)",
      "Bash(npm:test)",
      "Bash(npm:run:*)",
      "Bash(npx:*)",
      "Bash(node:*)",
      "Bash(python:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(find:*)",
      "Bash(grep:*)",
      "Bash(rg:*)",
      "Bash(wc:*)",
      "Bash(echo:*)",
      "Bash(which:*)",
      "Bash(env:*)",
      "Bash(pwd)",
      "Bash(date)",
      "Bash(whoami)"
    ],
    "deny": [
      "Bash(rm:-rf:*)",
      "Bash(sudo:*)",
      "Bash(chmod:777:*)",
      "Bash(curl:*:POST:*)",
      "Bash(wget:*)",
      "Bash(dd:*)"
    ]
  }
}
```

### 全开版 — 效率最大化，含 Hook 自动化

适合：个人项目、信任的代码库、追求最快速度

```json
{
  "model": "sonnet",
  "effort": "auto",
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git:*)",
      "Bash(npm:*)",
      "Bash(npx:*)",
      "Bash(node:*)",
      "Bash(python:*)",
      "Bash(bun:*)",
      "Bash(pnpm:*)",
      "Bash(yarn:*)",
      "Bash(cargo:*)",
      "Bash(go:*)",
      "Bash(make:*)",
      "Bash(ls:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(find:*)",
      "Bash(grep:*)",
      "Bash(rg:*)",
      "Bash(wc:*)",
      "Bash(echo:*)",
      "Bash(which:*)",
      "Bash(env:*)",
      "Bash(pwd)",
      "Bash(date)",
      "Bash(sort:*)",
      "Bash(uniq:*)",
      "Bash(diff:*)",
      "Bash(tree:*)",
      "Bash(mkdir:*)",
      "Bash(touch:*)",
      "Bash(cp:*)",
      "Bash(mv:*)"
    ],
    "deny": [
      "Bash(rm:-rf:*)",
      "Bash(sudo:*)",
      "Bash(chmod:777:*)",
      "Bash(curl:*:POST:*)",
      "Bash(dd:*)",
      "Bash(kill:-9:*)",
      "Bash(pkill:*)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "type": "command",
        "command": "prettier --write --ignore-unknown $CHANGED_FILES 2>/dev/null || true",
        "if": "Edit(*)",
        "timeout": 10,
        "async": true
      }
    ]
  },
  "enableAutoMemory": true
}
```

---

## 二、CLAUDE.md（指令文件）

文件位置：项目根目录 `CLAUDE.md`

### 通用模板 — 前端/全栈项目

```markdown
# claude

## 项目信息
- 语言: TypeScript
- 框架: React + Node.js
- 包管理: pnpm
- 测试: vitest
- 格式化: prettier + eslint

## 工作规则
- commit message 用中文
- 不要给没改过的代码加注释或类型标注
- 修改前先读文件，理解上下文再动手
- 一次只做一件事，做完再做下一件
- 遇到报错先分析原因，不要重试相同命令

## 代码风格
- 函数式优先，避免 class
- 组件用函数组件 + hooks
- 变量命名用 camelCase，常量用 UPPER_SNAKE_CASE
- 文件命名用 kebab-case
```

### 通用模板 — Python 项目

```markdown
# claude

## 项目信息
- 语言: Python 3.11+
- 框架: FastAPI
- 包管理: poetry
- 测试: pytest
- 格式化: ruff

## 工作规则
- commit message 用中文
- 不加多余的 docstring，函数名能说明意图就行
- 修改前先读文件
- 一次只做一件事
- 报错先分析，不重试

## 代码风格
- 类型标注必须加
- 用 Pydantic model 做数据校验
- async 优先
```

### 全局偏好 — 放在 ~/CLAUDE.md

```markdown
# claude

- 回复用中文
- 不要用 emoji
- 重要决策先说方案再执行
- 不确定的事情先问，不要猜
- 不要给时间估计
```

---

## 三、环境变量（.bashrc / .zshrc）

### 日常版

```bash
# Claude Code 效率配置
export ANTHROPIC_SMALL_FAST_MODEL=claude-haiku-4-5-20251001  # 子 Agent 用便宜模型
export BASH_DEFAULT_TIMEOUT_MS=120000                         # Bash 超时 2 分钟
export BASH_MAX_TIMEOUT_MS=600000                             # 最大 10 分钟
```

### 隐私版 — 最小化网络流量

```bash
# 隐私优先配置
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1  # 关闭遥测和非必要请求
export CLAUDE_CODE_DISABLE_AUTO_MEMORY=1            # 不自动记忆
export DISABLE_AUTO_COMPACT=1                       # 手动控制压缩
```

### 企业代理版

```bash
# 企业网络配置
export https_proxy=http://proxy.company.com:8080
export no_proxy=localhost,127.0.0.1,.company.com
export CLAUDE_CODE_PROXY_RESOLVES_HOSTS=1
export NODE_EXTRA_CA_CERTS=/path/to/company-ca.pem
```

### AWS Bedrock 版

```bash
# 走 AWS Bedrock
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-west-2
export AWS_PROFILE=my-profile
```

---

## 四、Shell 别名

加到 `.bashrc` / `.zshrc` / `.bash_profile`：

```bash
# Claude Code 快捷命令
alias cc='claude'
alias ccp='claude --print'
alias ccf='claude --fast'
alias ccb='claude --bare'
alias ccr='claude --resume'
alias ccc='claude --continue'

# 常用组合
alias cc-review='claude --print "review this PR for bugs and security issues" --output-format json'
alias cc-test='claude --print "run tests and report results" --max-turns 5'
alias cc-json='claude --print --output-format json --json-schema'
```

---

## 五、Hook 配方

加到 `settings.json` 的 `"hooks"` 字段里。

### 编辑后自动格式化（Prettier）

```json
{
  "type": "command",
  "command": "prettier --write --ignore-unknown $CHANGED_FILES 2>/dev/null || true",
  "if": "Edit(*)",
  "timeout": 10,
  "async": true
}
```

### 编辑后自动格式化（Ruff / Python）

```json
{
  "type": "command",
  "command": "ruff format $CHANGED_FILES 2>/dev/null && ruff check --fix $CHANGED_FILES 2>/dev/null || true",
  "if": "Edit(*.py)",
  "timeout": 10,
  "async": true
}
```

### 编辑 src/ 后自动跑测试

```json
{
  "type": "command",
  "command": "npm test -- --bail 2>&1 | tail -20",
  "if": "Edit(src/**/*)",
  "timeout": 60,
  "async": true
}
```

### 会话结束发 Slack 通知

```json
{
  "type": "http",
  "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
  "headers": { "Content-Type": "application/json" },
  "timeout": 5
}
```

放在 `"SessionEnd"` 事件下。

---

## 六、自定义 Agent

放在 `.claude/agents/` 目录下。

### reviewer.md — 只读代码审查

```markdown
---
name: reviewer
description: 代码审查，只读不写
tools: [Read, Glob, Grep]
model: claude-sonnet-4-6
permissionMode: acceptEdits
maxTurns: 50
---
审查规则：
1. 不修改任何文件
2. 检查：安全漏洞、性能问题、可维护性、命名规范
3. 按严重程度分级：Critical / Warning / Info
4. 输出格式：表格，列出文件路径、行号、问题、建议
```

### quick-check.md — 快速健康检查

```markdown
---
name: quick-check
description: 项目快速健康检查
tools: [Read, Glob, Grep, Bash]
model: claude-haiku-4-5-20251001
maxTurns: 20
---
快速检查以下项目，控制在 5 分钟内：
1. package.json 依赖是否有已知漏洞（npm audit）
2. 是否有未处理的 TODO/FIXME/HACK
3. 是否有 .env 或密钥文件被 git 追踪
4. TypeScript 是否有 any 类型泛滥
输出一页摘要，不超过 20 行。
```

---

## 七、快捷键速查卡

打印出来贴屏幕旁边。

```
操作                快捷键
─────────────────────────────────
切换模型            Meta+P
切换 Fast 模式      Meta+O
切换思考模式        Meta+T
循环权限模式        Shift+Tab
历史搜索            Ctrl+R
暂存输入            Ctrl+S
外部编辑器写 prompt Ctrl+X Ctrl+E
粘贴图片            Alt+V (Win)
查看任务列表        Ctrl+T
查看完整对话        Ctrl+O
重绘屏幕            Ctrl+L
中断                Ctrl+C
退出                Ctrl+D (×2)
```

---

## 八、一键初始化脚本

把以下脚本保存为 `setup-claude.sh`，在新项目中运行一次：

```bash
#!/bin/bash
# Claude Code 项目初始化
# 用法: bash setup-claude.sh

echo "初始化 Claude Code 配置..."

# 创建项目级 CLAUDE.md（如果不存在）
if [ ! -f "CLAUDE.md" ]; then
cat > CLAUDE.md << 'CLAUDEMD'
# claude

## 项目信息
- 语言: （填写）
- 框架: （填写）
- 测试: （填写）

## 工作规则
- commit message 用中文
- 修改前先读文件
- 一次只做一件事
- 报错先分析，不重试
CLAUDEMD
echo "  已创建 CLAUDE.md"
fi

# 创建 .claude 目录结构
mkdir -p .claude/agents
mkdir -p .claude/skills

# 创建审查 Agent
if [ ! -f ".claude/agents/reviewer.md" ]; then
cat > .claude/agents/reviewer.md << 'AGENT'
---
name: reviewer
description: 代码审查，只读不写
tools: [Read, Glob, Grep]
model: claude-sonnet-4-6
maxTurns: 50
---
审查规则：不修改文件。检查安全漏洞、性能、可维护性。
按 Critical/Warning/Info 分级，输出表格。
AGENT
echo "  已创建 .claude/agents/reviewer.md"
fi

# 创建项目级 settings
if [ ! -f ".claude/settings.json" ]; then
cat > .claude/settings.json << 'SETTINGS'
{
  "permissions": {
    "allow": [
      "Read", "Glob", "Grep",
      "Bash(git:*)", "Bash(npm:test)", "Bash(npm:run:*)"
    ],
    "deny": [
      "Bash(rm:-rf:*)", "Bash(sudo:*)"
    ]
  }
}
SETTINGS
echo "  已创建 .claude/settings.json"
fi

echo "完成。运行 claude 开始使用。"
```

---

## 配置效果对照

| 配置项 | 效果 | 节省 |
|--------|------|------|
| 权限白名单 (日常版) | git/npm/搜索/读取不再弹确认 | 每次会话少点 20+ 次确认 |
| `ANTHROPIC_SMALL_FAST_MODEL=haiku` | 子 Agent 用 Haiku | 子 Agent 成本降 ~90% |
| PostToolUse Hook (prettier) | 编辑后自动格式化 | 不用手动跑 format |
| CLAUDE.md 项目模板 | 规则只说一次，每次会话自动加载 | 不用重复交代偏好 |
| Shell 别名 | `cc` 代替 `claude` | 每天少敲几十个字符 |
| 自定义 reviewer Agent | 一句话启动代码审查 | 不用每次写审查 prompt |
| `Meta+P` 切模型 | 简单任务切 Sonnet，复杂任务切 Opus | 按需付费，不多花 |
| `--bare` 模式 | CI/沙盒环境极速启动 | 跳过 5+ 个初始化步骤 |
