# 第四课：子代理（Subagents）

---

## 一、Subagents 是什么

子代理是 Claude 可以委派任务的**专业 AI 助手**，每个子代理拥有：
- 独立的上下文窗口（不污染主对话）
- 自定义系统提示（领域专业化）
- 受控的工具访问权限

核心价值：将复杂任务分解，让专才处理专事，主代理只做协调和合成。

---

## 二、文件位置与优先级

| 优先级 | 类型 | 路径 | 范围 |
|--------|------|------|------|
| 1（最高） | CLI 定义 | `--agents` 参数（JSON） | 当次会话 |
| 2 | Project | `.claude/agents/` | 当前项目，进 git |
| 3 | User | `~/.claude/agents/` | 全部项目 |
| 4（最低） | Plugin | 插件 `agents/` 目录 | 插件作用域 |

同名时高优先级覆盖低优先级。

---

## 三、AGENT.md 关键配置字段

```yaml
---
name: code-reviewer                   # 唯一标识符（小写字母+连字符）
description: 做什么。Use PROACTIVELY  # 自动委派的核心，含触发词
tools: Read, Grep, Glob               # 省略则继承全部工具
disallowedTools: Bash                 # 明确禁止的工具
model: sonnet                         # sonnet / opus / haiku / inherit
permissionMode: default               # default / acceptEdits / bypassPermissions / plan / dontAsk / auto
maxTurns: 20                          # 最大轮次限制
memory: project                       # 持久化内存范围：user / project / local
background: true                      # 始终后台运行
isolation: worktree                   # 给予独立 git worktree
initialPrompt: "先分析代码库结构"      # 启动时自动提交的第一条消息
---

系统提示正文写在这里……
```

---

## 四、内置子代理

| 代理 | 模型 | 用途 |
|------|------|------|
| general-purpose | 继承 | 复杂多步骤任务 |
| Plan | 继承 | 规划模式下研究代码库 |
| **Explore** | **Haiku（快）** | 只读代码库探索（quick/medium/very thorough） |
| Bash | 继承 | 在独立上下文中执行 Shell 命令 |
| Claude Code Guide | Haiku | 回答 Claude Code 功能问题 |

---

## 五、调用子代理的三种方式

```
# 自动委派（Claude 根据 description 判断）
修复这个 bug 吧

# 显式指定
用 code-reviewer 子代理检查我最近的改动

# @-mention（保证调用，绕过自动判断）
@"code-reviewer (agent)" review auth module
```

**触发自动委派**：在 description 里写 `use PROACTIVELY` 或 `MUST BE USED`。

---

## 六、四个重要特性

### 持久化内存（memory）

```yaml
memory: project   # .claude/agent-memory/<name>/
```
- 子代理的 `MEMORY.md` 前 200 行自动加载到系统提示
- 跨会话积累知识，自动启用 Read/Write/Edit 工具管理记忆文件

### 后台运行（background）

```yaml
background: true
```
- `Ctrl+B`：将正在运行的任务发送到后台
- `Ctrl+F`（两次）：终止所有后台代理

### Worktree 隔离（isolation）

```yaml
isolation: worktree
```
- 子代理在独立的 git worktree + 分支上操作
- 无变更 → 自动清理；有变更 → 返回 worktree 路径和分支名给主代理

### 限制可展开的子代理

```yaml
tools: Agent(worker, researcher), Read, Bash
```
> v2.1.63 前叫 `Task(name)`，现在是 `Agent(name)`，旧语法仍兼容。

---

## 七、恢复子代理（Resumable）

```
# 第一次调用，返回 agentId: "abc123"
用 code-analyzer 子代理分析认证模块

# 后续恢复
Resume agent abc123，继续分析授权逻辑
```

恢复方式：调用 Task/Agent 工具时传入 `resume: "abc123"` 参数，**不是斜杠命令**。

---

## 八、Agent Teams（实验性）

多个 Claude Code 实例协同工作，与子代理的核心区别：

| 对比 | 子代理 | Agent Teams |
|------|--------|-------------|
| 上下文 | 每次干净的新上下文 | 每个 teammate 有自己的持久上下文 |
| 通信 | 结果只返回给主代理 | teammate 间可通过 mailbox 互发消息 |
| 适合 | 明确的单一子任务 | 需要并行+跨代理沟通的复杂工作 |

启用：`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

---

## 九、最佳实践速记

**该做的**：
- description 写明触发场景，复杂任务加 `use PROACTIVELY`
- 工具权限最小化（分析类 agent 只给 Read/Grep/Glob）
- Project 级 agent 进 git，团队共享

**不该做的**：
- 简单单步任务用子代理（增加延迟，得不偿失）
- 多个子代理职责重叠
- 忘记传递必要上下文给子代理
