# 基于 Claude Code 构建自定义 Agent、Team 和 Agent 编排工具 — 可行性调研报告

## 结论摘要

| 维度 | 可行性 | 难度 | 说明 |
|------|--------|------|------|
| 自定义 Agent | ✅ **高** | ⭐ 低 | 原生支持 Markdown/JSON 定义，无需改代码 |
| Team (多Agent协作) | ✅ **高** | ⭐⭐ 中 | 已有完整 Swarm 框架，支持 tmux/iTerm2/进程内 |
| Agent 编排工具 | ⚠️ **中** | ⭐⭐⭐ 高 | 需深度定制，且受限于 Anthropic API 绑定和许可证 |
| 完全独立产品 | ⚠️ **有限** | ⭐⭐⭐⭐ 极高 | 强绑定 Anthropic 基础设施，无 LICENSE.md，法律风险 |

---

## 一、自定义 Agent 系统（✅ 开箱即用）

### 1.1 Agent 定义方式

代码库已提供 **两种** 原生自定义 Agent 方式，无需修改源码：

#### 方式一：Markdown 文件定义

在 `~/.claude/agents/` 或 `.claude/agents/` 目录下创建 `.md` 文件：

```markdown
---
name: my-researcher
description: 专门用于代码库调研和分析的Agent
model: claude-sonnet-4-6
tools: [Read, Glob, Grep, Bash]
disallowedTools: [Write, Edit]
permissionMode: acceptEdits
maxTurns: 50
effort: high
color: blue
background: false
memory: project
isolation: worktree
mcpServers:
  - "github"
  - { "custom-server": { "command": "npx", "args": ["my-mcp-server"] } }
hooks:
  PreToolUse:
    - matcher: "Bash(rm *)"
      hooks:
        - type: command
          command: "echo 'Blocked destructive command'"
skills:
  - commit
  - review-pr
---

你是一个代码库研究专家。你的任务是...
（这里写 system prompt）
```

**支持的 frontmatter 字段（完整列表）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Agent 唯一标识（必填） |
| `description` | string | 何时使用此 Agent（必填） |
| `model` | string | 模型选择，支持 `inherit` 继承父级 |
| `tools` | string[] | 允许使用的工具列表 |
| `disallowedTools` | string[] | 禁止使用的工具列表 |
| `permissionMode` | enum | 权限模式：`default`/`acceptEdits`/`bypassPermissions` 等 |
| `maxTurns` | number | 最大轮次限制 |
| `effort` | string/number | 推理深度：`low`/`medium`/`high`/`max` 或数字 |
| `color` | string | UI 显示颜色 |
| `background` | boolean | 是否后台运行 |
| `memory` | enum | 持久化记忆：`user`/`project`/`local` |
| `isolation` | enum | 隔离模式：`worktree`（独立 git 工作树） |
| `mcpServers` | array | Agent 专属 MCP 服务器 |
| `hooks` | object | Agent 专属 Hooks |
| `skills` | string[] | 预加载的技能 |
| `initialPrompt` | string | 首轮预注入提示词 |

> 源码参考：`src/tools/AgentTool/loadAgentsDir.ts:106-133`

#### 方式二：JSON 定义（通过 settings.json 或 Feature Flag）

```json
{
  "my-agent": {
    "description": "何时使用此Agent的说明",
    "prompt": "你是一个...",
    "tools": ["Read", "Bash", "Grep"],
    "model": "claude-sonnet-4-6",
    "effort": "high",
    "maxTurns": 30,
    "permissionMode": "acceptEdits",
    "background": true,
    "memory": "project",
    "isolation": "worktree"
  }
}
```

> 源码参考：`src/tools/AgentTool/loadAgentsDir.ts:73-98` (AgentJsonSchema)

### 1.2 Agent 加载优先级

```
内置 Agent (built-in)
  └─ 插件 Agent (plugin)
       └─ 用户 Agent (~/.claude/agents/)
            └─ 项目 Agent (.claude/agents/)
                 └─ Flag Agent (flagSettings)
                      └─ 策略 Agent (policySettings)   ← 后加载的覆盖前面同名的
```

> 源码参考：`src/tools/AgentTool/loadAgentsDir.ts:193-221` (getActiveAgentsFromList)

### 1.3 内置 Agent 参考

| Agent | 用途 | 源码 |
|-------|------|------|
| `general-purpose` | 通用多步骤任务 | `built-in/generalPurposeAgent.ts` |
| `Explore` | 快速代码库探索 | `built-in/exploreAgent.ts` |
| `Plan` | 架构设计和实现规划 | `built-in/planAgent.ts` |
| `verification` | 实现正确性验证 | `built-in/verificationAgent.ts` |
| `claude-code-guide` | Claude Code 使用指南 | `built-in/claudeCodeGuideAgent.ts` |
| `statusline-setup` | 状态栏配置 | `built-in/statuslineSetup.ts` |

---

## 二、Team / Swarm 多 Agent 协作（✅ 已有完整框架）

### 2.1 Swarm 架构

代码库已实现完整的 **多 Agent 协作框架（Swarm）**：

```
┌───────────────────────────────────────────────────────────────┐
│                      Team Lead (队长)                          │
│                                                               │
│  · 创建团队 (spawnTeam)                                        │
│  · 分配任务给 Teammate                                         │
│  · 通过 Mailbox 收发消息                                       │
│  · 管理 Teammate 生命周期                                      │
│  · 汇总结果                                                   │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │Teammate A│  │Teammate B│  │Teammate C│  │Teammate D│     │
│  │researcher│  │ tester   │  │ coder    │  │ reviewer │     │
│  │  (blue)  │  │ (green)  │  │  (red)   │  │(yellow)  │     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘     │
│       │              │              │              │           │
│       └──────────────┴──────┬───────┴──────────────┘           │
│                             │                                  │
│                    ┌────────▼────────┐                         │
│                    │  Mailbox 系统    │                         │
│                    │ (文件级消息传递)  │                         │
│                    └─────────────────┘                         │
└───────────────────────────────────────────────────────────────┘
```

> 源码参考：`src/utils/swarm/` 目录

### 2.2 执行后端

支持 **三种** Teammate 执行后端：

| 后端 | 说明 | 场景 |
|------|------|------|
| **tmux** | 通过 tmux 窗格管理 | 服务器/终端环境 |
| **iTerm2** | 利用 iTerm2 原生分屏 | macOS |
| **in-process** | 同一进程内 AsyncLocalStorage 隔离 | 轻量级/编程使用 |

> 源码参考：`src/utils/swarm/backends/types.ts:8-9`

### 2.3 Team 文件结构

```typescript
type TeamFile = {
  name: string              // 团队名
  description?: string      // 团队描述
  createdAt: number
  leadAgentId: string       // 队长 ID
  leadSessionId?: string    // 队长会话 ID
  teamAllowedPaths?: TeamAllowedPath[]  // 团队共享权限路径
  members: Array<{
    agentId: string
    name: string            // 角色名（如 "researcher"）
    agentType?: string
    model?: string
    prompt?: string         // 分配给该成员的任务
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    isActive?: boolean
    tmuxPaneId: string
    cwd: string
    worktreePath?: string   // 独立 git worktree
    backendType?: string
    mode?: string           // 权限模式
  }>
}
```

> 源码参考：`src/utils/swarm/teamHelpers.ts:64-80`

### 2.4 通信机制：Mailbox

Teammate 之间通过 **文件级 Mailbox** 通信：

```
~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

- 支持消息发送/读取/标记已读
- 使用文件锁防并发
- 消息结构：`{ from, text, timestamp, read, color, summary }`
- 工具：`SendMessageTool` 用于 Agent 间发消息

> 源码参考：`src/utils/teammateMailbox.ts`

### 2.5 Teammate 生命周期

```
spawn ──► 运行任务 ──► 空闲通知队长 ──► 等待新任务 / 被终止
  │                         │
  ├── 权限同步 (从队长继承)   ├── 通过 Stop Hook 通知
  ├── 工作树隔离 (可选)       ├── 写入 Mailbox
  └── Plan Mode (可选)       └── 更新 TeamFile
```

> 源码参考：`src/utils/swarm/teammateInit.ts`、`src/utils/swarm/inProcessRunner.ts`

---

## 三、扩展性分析

### 3.1 插件系统

```typescript
type BuiltinPluginDefinition = {
  name: string
  description: string
  version?: string
  skills?: BundledSkillDefinition[]     // 提供技能
  hooks?: HooksSettings                  // 提供 Hooks
  mcpServers?: Record<string, McpServerConfig>  // 提供 MCP 服务器
  isAvailable?: () => boolean            // 平台可用性检查
  defaultEnabled?: boolean
}
```

插件可以从 **Marketplace** 加载：GitHub repo、Git URL 等。

> 源码参考：`src/types/plugin.ts`、`src/plugins/builtinPlugins.ts`

### 3.2 Skills 系统

技能（Skill）是可复用的提示词模板，支持：
- Markdown 文件定义（带 frontmatter）
- 工具限制、模型覆盖、执行上下文
- 从 `~/.claude/skills/` 或 `.claude/skills/` 加载
- 可通过插件分发

> 源码参考：`src/skills/loadSkillsDir.ts`

### 3.3 Hooks 系统

14+ Hook 事件，4 种 Hook 类型：

| Hook 类型 | 说明 |
|-----------|------|
| `command` | 执行 Shell 命令 |
| `prompt` | LLM 评估提示词 |
| `http` | POST 到外部端点 |
| `agent` | 启动验证 Agent |

关键事件：`PreToolUse`、`PostToolUse`、`SessionStart`、`SessionEnd`、`SubagentStart`、`SubagentStop`、`TaskCreated`、`TaskCompleted` 等。

> 源码参考：`src/schemas/hooks.ts`、`src/utils/hooks.ts`

### 3.4 MCP（Model Context Protocol）扩展

通过 `.mcp.json` 添加自定义工具服务器，支持 stdio/SSE/HTTP/WebSocket 传输。

> 源码参考：`src/services/mcp/`

---

## 四、关键限制与风险

### 4.1 Anthropic API 强绑定

```
以下模块强依赖 Anthropic 基础设施：

┌─────────────────────────────────────────────────┐
│  紧耦合层（难以替换）                              │
│                                                 │
│  · @anthropic-ai/sdk — API 调用核心              │
│  · OAuth 认证 — 绑定 Anthropic 账户体系            │
│  · Bootstrap API — 启动时获取配置                  │
│  · GrowthBook Feature Flags — A/B 测试            │
│  · Sentry 错误追踪 — 绑定 Anthropic 项目           │
│  · 分析遥测 — 发送到 Anthropic 服务器              │
│  · 模型选择 — 硬编码 Claude 系列模型               │
│  · 订阅/计费 — 绑定 Anthropic 计费系统             │
│  · MCP 官方注册表 — api.anthropic.com              │
└─────────────────────────────────────────────────┘
```

### 4.2 许可证风险

- `package.json` 标注 `"license": "SEE LICENSE IN LICENSE.md"`，但 **LICENSE.md 文件不存在**
- 原始仓库为 Anthropic 私有仓库
- 当前代码是从 source map **逆向重建** 的（README 中明确说明）
- **法律风险高**：未经 Anthropic 授权，基于此代码构建商业产品可能存在侵权

### 4.3 代码完整性

- 部分模块从 source map 无法完全恢复，使用了 shim 替代
- 某些私有/原生集成使用降级回退
- 行为可能与原始实现有差异

---

## 五、构建方案建议

### 方案 A：轻量定制（推荐 ✅）

**不修改源码**，利用原生扩展点：

```
┌──────────────────────────────────────────┐
│           你的 Agent 编排系统              │
│                                          │
│  ┌─────────────────────────────────────┐ │
│  │  ~/.claude/agents/                  │ │
│  │  ├── researcher.md    (调研Agent)    │ │
│  │  ├── coder.md         (编码Agent)    │ │
│  │  ├── reviewer.md      (审查Agent)    │ │
│  │  ├── tester.md        (测试Agent)    │ │
│  │  └── coordinator.md   (编排Agent)    │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  ┌─────────────────────────────────────┐ │
│  │  ~/.claude/skills/                  │ │
│  │  ├── deploy.md        (部署技能)     │ │
│  │  ├── code-review.md   (代码审查)     │ │
│  │  └── ...                            │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  ┌─────────────────────────────────────┐ │
│  │  .mcp.json                          │ │
│  │  自定义工具服务器                     │ │
│  │  (Jira、Slack、内部API等)            │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  ┌─────────────────────────────────────┐ │
│  │  ~/.claude/settings.json            │ │
│  │  · Hooks (工作流自动化)              │ │
│  │  · 权限规则                          │ │
│  │  · 插件配置                          │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  直接使用 Swarm 框架组建 Team             │
│  利用 Mailbox 实现 Agent 间通信           │
└──────────────────────────────────────────┘
```

**优势**：
- 零代码修改，跟随上游更新
- 利用现有 Swarm 框架
- 通过 MCP 扩展无限工具能力

**劣势**：
- 受限于 Claude Code 的交互模型
- 绑定 Anthropic API

---

### 方案 B：基于 Agent SDK 构建（⚠️ 中等复杂度）

利用 `@anthropic-ai/claude-agent-sdk` 编程接口：

```typescript
// 伪代码示意
import { QueryEngine } from '@anthropic-ai/claude-agent-sdk'

// 创建自定义 Agent
const researcher = new QueryEngine({
  model: 'claude-sonnet-4-6',
  systemPrompt: '你是一个研究专家...',
  tools: [ReadTool, GrepTool, WebSearchTool],
  maxTurns: 50,
})

// 创建编排器
const orchestrator = {
  async run(task: string) {
    const plan = await planner.query(task)
    const results = await Promise.all(
      plan.subtasks.map(t => researcher.query(t))
    )
    return await synthesizer.query(results)
  }
}
```

**优势**：
- 编程式控制，灵活度高
- 可嵌入到自己的应用中
- SDK 有正式发布和文档

**劣势**：
- 需要理解 SDK API
- 仍绑定 Anthropic API
- SDK 功能可能不如 CLI 完整

---

### 方案 C：深度 Fork（❌ 不推荐）

Fork 源码做深度修改：

**风险**：
- 许可证不明确，法律风险高
- 代码从 source map 逆向重建，完整性存疑
- 维护成本极高（206K+ 行代码）
- 强耦合 Anthropic 基础设施，替换工作量巨大

---

## 六、实际可复用的架构模式

即使不直接 Fork 代码，以下架构模式值得参考和借鉴：

### 6.1 Agent 定义模式
```
Markdown frontmatter + 正文内容 = Agent 定义
  ├── frontmatter → 配置（模型、工具、权限、hooks）
  └── content → 系统提示词
```

### 6.2 多 Agent 通信模式
```
文件级 Mailbox + 锁机制
  ├── 每个 Agent 一个 inbox 文件
  ├── 文件锁防并发写入
  └── 轮询或 Hook 触发读取
```

### 6.3 工具权限模式
```
三级权限：allowList / denyList / askList
  ├── 静态规则（配置文件）
  ├── 动态分类器（auto-mode）
  └── 交互式确认（ask）
```

### 6.4 上下文压缩模式
```
Token 预算管理 + 自动压缩
  ├── 工具结果摘要
  ├── 历史消息裁剪
  └── 分段压缩
```

### 6.5 隔离执行模式
```
Git Worktree 隔离
  ├── 每个 Agent 独立工作树
  ├── 避免文件冲突
  └── 完成后合并或丢弃
```

---

## 七、推荐路径

```
                        你想做什么？
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        定制 Agent     构建 Agent      构建独立
        工作流         编排平台        AI 产品
              │             │             │
              ▼             ▼             ▼
         方案 A          方案 B       使用 Agent
       (配置定制)     (SDK 编程)     SDK + 自研
              │             │             │
              ▼             ▼             ▼
        · Agent .md    · QueryEngine  · 参考架构模式
        · Skills       · 自定义编排    · 自研通信机制
        · MCP 服务器    · 嵌入应用      · 自研工具系统
        · Hooks        · API 集成      · 多模型支持
        · Swarm Team                   · 独立许可证
```

**如果目标是团队内部使用**：方案 A 最快、最安全
**如果目标是构建 SaaS 产品**：方案 B（Agent SDK）+ 自研编排层
**如果需要多模型支持**：必须自研，但可借鉴上述架构模式
