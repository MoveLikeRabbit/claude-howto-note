# Claude Code 干货学习笔记

> 根据原始 10 章笔记精简、校正并重组。  
> 校对日期：2026-06-10。具体命令和实验性功能可能继续变化，使用前以官方文档和本机 `claude --help` 为准。

## 1. 先建立正确的心智模型

Claude Code 的核心能力可以分成 6 层：

| 机制 | 解决什么问题 | 典型载体 |
|---|---|---|
| Instructions / Memory | 持久保存规则、偏好和项目知识 | `CLAUDE.md`、`.claude/rules/`、Auto Memory |
| Skills | 按需加载可复用知识和工作流 | `.claude/skills/<name>/SKILL.md` |
| Subagents | 把独立子任务交给专业代理 | `.claude/agents/<name>.md` |
| MCP | 连接外部工具、服务和实时数据 | `.mcp.json`、`claude mcp add` |
| Hooks | 在固定事件点强制执行自动化 | `settings.json` 中的 `hooks` |
| Plugins | 将 Skills、Agents、Hooks、MCP 等打包分发 | `.claude-plugin/plugin.json` |

### 一句话选型

- 每次会话都要知道的规则：`CLAUDE.md`
- 仅对特定目录或文件生效的规则：`.claude/rules/`
- 可重复的多步骤操作：Skill
- 独立、复杂、适合并行的任务：Subagent
- GitHub、数据库、Notion 等外部系统：MCP
- 必须在某个事件点执行或拦截：Hook
- 需要团队安装和版本管理的一整套扩展：Plugin

---

## 2. CLAUDE.md 与记忆系统

### 2.1 四种常用指令范围

| 范围 | 路径 | 用途 |
|---|---|---|
| Organization | macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md` | 企业统一规则 |
| User | `~/.claude/CLAUDE.md` | 个人跨项目偏好 |
| Project | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享项目规则 |
| Local | `./CLAUDE.local.md` | 当前项目的个人配置，不进 Git |

规则不会互相简单覆盖，而是一起进入上下文。越接近当前工作目录的内容越靠后，因此应避免冲突。

### 2.2 CLAUDE.md 应该写什么

适合写：

- 构建、测试、Lint 命令
- 项目目录和关键模块说明
- 明确的编码规范
- 团队工作流和禁止事项
- Claude 从代码中不容易推断出的约定

不适合写：

- API Key、密码、个人隐私
- 很长的教程或一次性任务
- 可直接从代码读出的细节
- 必须百分百强制执行的安全规则

推荐控制在 200 行以内。指令要具体，例如：

```markdown
- 修改 TypeScript 后运行 `npm test`
- API handler 放在 `src/api/handlers/`
- 使用 2 空格缩进
```

不要只写“保持代码整洁”“做好测试”这类模糊要求。

### 2.3 模块化规则

```text
.claude/
├── CLAUDE.md
└── rules/
    ├── testing.md
    └── api.md
```

使用 `paths` 只在匹配文件被读取时加载规则：

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "tests/**/*.test.ts"
---

# API 规则
- 所有输入必须进行 schema 校验
- 错误响应使用统一结构
```

### 2.4 导入其他文件

```markdown
@README.md
@docs/architecture.md
@~/.claude/personal-workflow.md
```

- 相对路径相对于当前 `CLAUDE.md` 解析。
- 外部导入首次使用需要确认。
- 导入只是便于组织，不会减少上下文占用。
- 当前官方文档说明最多递归导入 4 层，不是原笔记中的 5 层。

### 2.5 Auto Memory

Auto Memory 是 Claude 自己维护的项目经验：

```text
~/.claude/projects/<project>/memory/
├── MEMORY.md
├── debugging.md
└── api-conventions.md
```

- 默认开启。
- 每次会话自动加载 `MEMORY.md` 的前 200 行或 25KB，取先达到者。
- 详细内容应拆入主题文件，Claude 按需读取。
- 同一 Git 仓库的多个 worktree 共用项目记忆。
- 使用 `/memory` 查看、编辑或关闭 Auto Memory。

配置示例：

```json
{
  "autoMemoryEnabled": false,
  "autoMemoryDirectory": "~/my-claude-memory"
}
```

环境变量关闭方式：

```bash
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 claude
```

### 2.6 高频命令

```text
/init      分析项目并生成或改进 CLAUDE.md
/memory    查看已加载的指令文件和 Auto Memory
/compact   压缩会话上下文
```

---

## 3. Skills：可复用工作流

### 3.1 目录结构

```text
.claude/skills/code-review/
├── SKILL.md
├── scripts/
├── references/
└── assets/
```

常用位置：

- 个人：`~/.claude/skills/<name>/SKILL.md`
- 项目：`.claude/skills/<name>/SKILL.md`
- 插件：`<plugin>/skills/<name>/SKILL.md`

旧的 `.claude/commands/` 仍可兼容，但新工作流优先使用 Skills。

### 3.2 渐进式加载

1. 启动时只加载名称和描述。
2. Skill 被调用或判断为相关时，加载 `SKILL.md` 正文。
3. 脚本、参考资料和资源按需加载。

因此 Skill 的 `description` 很关键，应说明“做什么”和“何时使用”。

### 3.3 推荐模板

```markdown
---
name: code-review
description: Review code for correctness, security, regressions, and missing tests. Use when the user asks to review changes or a pull request.
argument-hint: "[file, directory, or PR]"
allowed-tools: Read, Grep, Glob, Bash(git diff:*)
---

# Code Review

1. Read the changed code and surrounding contracts.
2. Prioritize bugs, security risks, regressions, and missing tests.
3. Report findings by severity with file and line references.
4. Keep summaries secondary to actionable findings.
```

### 3.4 调用控制

| 配置 | 用户手动调用 | Claude 自动调用 |
|---|---:|---:|
| 默认 | 是 | 是 |
| `disable-model-invocation: true` | 是 | 否 |
| `user-invocable: false` | 否 | 是 |

有部署、发布、删除等副作用的 Skill，应禁止模型自动调用。

### 3.5 动态内容

```text
$ARGUMENTS
$0、$1
${CLAUDE_SKILL_DIR}
!`git status`
```

Shell 注入会真正执行命令，第三方 Skill 必须先审查再安装。

### 3.6 Skill 设计原则

- 一个 Skill 只解决一类明确问题。
- 正文保持短小，把长资料放入 `references/`。
- 脚本输出应稳定、精简、可重复。
- 不要为了“可能有用”而把大量内容常驻上下文。

---

## 4. Subagents：分工与并行

### 4.1 适用场景

适合：

- 独立代码审查
- 大范围只读探索
- 测试、研究、迁移等长任务
- 可以并行且文件边界清晰的任务

不适合：

- 简单单步修改
- 强依赖主对话全部细节的任务
- 多个代理同时修改同一文件

### 4.2 常用位置与优先级

```text
当前会话 --agents
项目 .claude/agents/
用户 ~/.claude/agents/
插件 agents/
```

同名时，更高优先级定义生效。

### 4.3 推荐模板

```markdown
---
name: security-reviewer
description: Review authentication, authorization, secrets, and input validation. Use proactively for security-sensitive changes.
tools: Read, Grep, Glob, Bash(git diff:*)
model: inherit
permissionMode: plan
maxTurns: 20
---

You are a security-focused reviewer.
Report exploitable issues first, include evidence, and avoid unrelated style feedback.
```

关键思想：

- `description` 决定何时委派。
- `tools` 遵循最小权限原则。
- `permissionMode: plan` 适合只读分析。
- `isolation: worktree` 适合需要独立修改代码的代理。
- `memory: user|project|local` 可让特定代理跨会话积累经验。

### 4.4 内置代理

- `Explore`：快速、只读地探索代码库。
- `Plan`：为复杂变更研究和制定计划。
- `general-purpose`：执行复杂多步骤任务。

具体内置代理、模型映射和快捷键可能随版本变化，不需要死记。

### 4.5 Subagent 与 Agent Teams

| 机制 | 特点 | 适用任务 |
|---|---|---|
| Subagent | 由主代理委派，完成后返回结果 | 边界明确的子任务 |
| Agent Teams | 多个持续工作的成员可互相协作 | 大型并行项目 |

Agent Teams 属于较新且变化快的能力，使用前查当前官方文档，不要把实验性环境变量当永久接口。

---

## 5. MCP：连接外部世界

### 5.1 MCP 与 Memory 的区别

- Memory 保存本地、相对稳定的规则和经验。
- MCP 连接外部工具、API、数据库和实时数据。

### 5.2 传输方式

| 方式 | 使用场景 |
|---|---|
| HTTP | 推荐用于远程云服务 |
| stdio | 本地进程、脚本和需要本机访问的工具 |
| WebSocket | 需要远程双向推送的服务 |
| SSE | 已弃用，仅兼容旧服务 |

### 5.3 添加服务器

```bash
# 远程 HTTP
claude mcp add --transport http notion https://mcp.notion.com/mcp

# 本地 stdio，注意 `--` 分隔 Claude 参数与服务器命令
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub

claude mcp list
claude mcp get db
claude mcp remove db
```

### 5.4 三种作用域

| Scope | 存储位置 | 是否共享 |
|---|---|---|
| local（默认） | `~/.claude.json` 中当前项目条目 | 否 |
| project | 项目根目录 `.mcp.json` | 是，适合进 Git |
| user | `~/.claude.json` | 否，跨项目使用 |

项目级 `.mcp.json` 首次使用需要人工信任。

### 5.5 安全配置

```json
{
  "mcpServers": {
    "api": {
      "type": "http",
      "url": "${API_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

- 凭证放环境变量或 OAuth，不要提交到 Git。
- 只连接可信服务器，外部内容可能包含提示注入。
- 数据库优先使用只读、最小权限账号。
- 第三方本地 MCP server 等同于本机程序，安装前审查来源。

### 5.6 输出与 Tool Search

- MCP 工具输出超过 10,000 tokens 时会警告。
- 可用 `MAX_MCP_OUTPUT_TOKENS` 调整限制。
- Tool Search 会按需加载工具定义，减少大量 MCP 工具占用的上下文。

原笔记中的“25,000 tokens 固定截断线”“50,000 字符固定写盘线”“每个服务器描述 2KB”等过细数字不适合作为稳定知识，当前官方参考页也没有把它们列为应背规则。

---

## 6. Hooks：强制自动化

### 6.1 什么时候用 Hook

- 执行工具前拦截危险命令
- 修改文件后运行格式化或检查
- 会话启动时注入环境
- Claude 准备停止时验证任务是否完成
- 将事件通知外部系统

CLAUDE.md 是行为指导，不是强制机制；必须执行或禁止的规则应使用 Hook、权限配置或沙箱。

### 6.2 常用事件

| 事件 | 作用 |
|---|---|
| `PreToolUse` | 工具调用前检查、允许、拒绝或修改输入 |
| `PostToolUse` | 工具成功后处理 |
| `PostToolUseFailure` | 工具失败后处理 |
| `UserPromptSubmit` | 用户提示提交后、模型处理前 |
| `SessionStart` | 会话初始化 |
| `Stop` | 主代理准备结束 |
| `SubagentStop` | 子代理准备结束 |

事件列表会扩展，不必背“总共多少个”。

### 6.3 配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/validate.py\"",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

Hook 可配置在：

- `~/.claude/settings.json`
- `.claude/settings.json`
- `.claude/settings.local.json`
- 插件的 `hooks/hooks.json`

### 6.4 Command Hook 通信

- Claude 通过 `stdin` 传入 JSON。
- 脚本通过 `stdout` 返回结构化 JSON。
- 退出码 `0` 表示正常处理。
- 退出码 `2` 表示阻断错误。
- 其他非零退出码通常作为非阻断错误处理。

需要修改工具输入时，通过 `PreToolUse` 的 `hookSpecificOutput.updatedInput` 返回，不要依赖临时文件。

### 6.5 Hook 安全原则

- 所有输入都按不可信数据处理。
- 对路径和命令做白名单校验。
- 为 Hook 设置超时。
- 输出保持小而明确。
- 避免 Hook 再次触发相同 Hook 形成循环。

---

## 7. Plugins：扩展能力打包

### 7.1 典型结构

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
├── agents/
├── commands/
├── hooks/
│   └── hooks.json
├── .mcp.json
├── .lsp.json
├── scripts/
└── templates/
```

最小 manifest：

```json
{
  "name": "my-plugin",
  "description": "Project automation tools",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

### 7.2 开发和安装

```bash
# 本地加载测试
claude --plugin-dir ./my-plugin

# 会话中管理
/plugin

# 修改后重新加载
/reload-plugins
```

插件命令通常使用命名空间：

```text
/plugin-name:command-name
```

### 7.3 什么时候才需要 Plugin

用 Plugin：

- 同时包含 Skills、Agents、Hooks、MCP 等多个组件
- 需要版本管理、安装、升级和团队分发

不用 Plugin：

- 只有一个工作流：直接做 Skill
- 只有一个专业代理：直接做 Agent
- 只连接实时服务：直接配置 MCP

---

## 8. Checkpoint、Git 与会话管理

### 8.1 Checkpoint 的作用

Claude Code 会在工作过程中保存可回退状态，可通过：

```text
Esc Esc
/rewind
```

典型选项包括：

- 恢复代码和对话
- 只恢复对话
- 只恢复代码
- 从某处总结后续对话

### 8.2 重要限制

Checkpoint 主要追踪 Claude Code 文件编辑工具造成的修改：

- Shell 命令的副作用不保证可恢复。
- Claude Code 外部程序造成的修改不保证可恢复。
- 它不是永久版本控制，也不是备份系统。

最佳组合：

```text
Checkpoint：短期试错和恢复
Git commit：永久、可审查、可共享的里程碑
```

### 8.3 常用会话操作

```bash
claude -c                 # 继续最近会话
claude -r <name-or-id>    # 恢复指定会话
```

会话内常用：

```text
/rename
/fork
/compact
```

---

## 9. 权限、安全与隔离

### 9.1 常见权限模式

| 模式 | 含义 |
|---|---|
| `default` | 按默认策略询问敏感操作 |
| `acceptEdits` | 自动接受文件编辑，其他敏感操作仍受控 |
| `plan` | 只读研究和规划 |
| `dontAsk` | 未预先允许的操作直接拒绝 |
| `bypassPermissions` | 跳过权限检查，风险最高 |

某些版本或套餐可能还有额外自动模式。不要依赖固定“共有 6 种”这一数字，以 `claude --help` 为准。

```bash
claude --permission-mode plan
```

### 9.2 工具权限

```bash
claude -p \
  --tools "Read,Grep,Glob,Bash" \
  --allowedTools "Bash(git diff:*)" \
  --disallowedTools "Bash(rm -rf:*)" \
  "Review the repository"
```

原则：

- 先给最少工具，再按需要增加。
- 自动化场景显式限制工具、轮次和预算。
- 不把 `bypassPermissions` 当日常模式。

### 9.3 Worktree 与 Sandbox

```bash
claude --worktree
```

Worktree 用于隔离 Git 分支和文件修改，适合并行任务。

Sandbox 用于限制文件系统和网络访问，适合运行不完全可信的命令或扩展。二者解决的问题不同，可以组合使用。

---

## 10. CLI 高频速查

### 10.1 交互与非交互

```bash
claude                         # 交互会话
claude "解释项目架构"          # 带初始提示启动
claude -p "列出所有 TODO"      # 非交互 Print Mode
cat error.log | claude -p "分析错误"
```

### 10.2 结构化输出

```bash
claude -p --output-format json "列出 API 端点"

claude -p \
  --json-schema '{"type":"object","properties":{"bugs":{"type":"array"}}}' \
  "检查代码并返回 bugs"
```

### 10.3 自动化常用限制

```bash
claude -p \
  --max-turns 5 \
  --max-budget-usd 2 \
  --permission-mode plan \
  --output-format json \
  "审查本次改动"
```

### 10.4 模型选择

```bash
claude --model sonnet
claude --model opus
claude --model haiku
```

模型版本、默认 effort 等级和套餐支持变化很快。稳定做法是使用别名或从 `/model` 选择，不要把某一时点的完整版本号写死进长期笔记。

### 10.5 最值得记住的斜杠命令

```text
/help       查看当前版本可用命令
/model      切换模型
/clear      清空当前对话
/compact    压缩上下文
/memory     管理指令与记忆
/mcp        查看 MCP 状态和认证
/permissions 管理权限
/rewind     打开回退界面
/plugin     管理插件
/doctor     检查安装和配置问题
```

斜杠命令会随版本、插件和 MCP 配置动态变化，`/help` 比背完整列表可靠。

---

## 11. 一套推荐的实际工作流

### 新项目初始化

```text
1. 在仓库根目录启动 Claude Code。
2. 运行 /init，审查生成的 CLAUDE.md。
3. 删除泛泛描述，只保留构建命令、项目约定和关键限制。
4. 把特定目录规则拆进 .claude/rules/。
5. 将团队配置提交 Git，个人配置放 CLAUDE.local.md。
```

### 实现复杂功能

```text
1. 先用 plan 模式研究范围和风险。
2. 为独立研究任务使用 Explore 或自定义 Subagent。
3. 经确认后实施修改。
4. 运行测试、Lint 和类型检查。
5. 检查 diff，再创建 Git commit。
6. 长对话及时 /compact，关键成果不要只依赖 Checkpoint。
```

### 建立自动化

```text
重复工作流 -> Skill
固定事件检查 -> Hook
外部实时系统 -> MCP
多组件团队分发 -> Plugin
CI 执行 -> claude -p + 工具/轮次/预算限制
```

---

## 12. 原笔记的主要纠正

1. `CLAUDE.md` 导入递归深度：当前官方文档为最多 4 层，不是 5 层。
2. Auto Memory 不只靠环境变量控制，也可通过 `/memory` 或 `autoMemoryEnabled` 设置。
3. `autoMemoryDirectory` 当前可从多种 settings scope 读取，不再应写成“只能用户级或 local”。
4. MCP 当前除 HTTP、stdio、弃用的 SSE 外，还支持 WebSocket；远程服务仍优先 HTTP。
5. MCP 官方文档明确的是超过 10,000 tokens 警告和可调上限；原笔记的固定 25,000/50,000/2KB 数字不宜继续背诵。
6. Hooks 事件持续增加，记住关键事件和数据流比记“总计 28 个”可靠。
7. 权限模式、自动模式、TUI、Voice、Ultraplan、排程等属于变化较快的产品能力，不应写成永久固定接口。
8. 模型版本、effort 默认值、套餐要求和模型映射会变化；长期笔记应保留选择原则，不应绑定某个版本。
9. Subagent、Skill 和 Plugin 的字段会扩展；创建时以当前官方 schema 为准，不要把旧示例当完整字段表。
10. Checkpoint 不能替代 Git，也不能可靠撤销 Shell 或外部程序造成的副作用。

---

## 13. 官方资料

- [Claude Code 文档首页](https://code.claude.com/docs)
- [Memory / CLAUDE.md](https://code.claude.com/docs/en/memory)
- [Skills](https://code.claude.com/docs/en/skills)
- [Subagents](https://code.claude.com/docs/en/sub-agents)
- [MCP](https://code.claude.com/docs/en/mcp)
- [Hooks](https://code.claude.com/docs/en/hooks)
- [Plugins](https://code.claude.com/docs/en/plugins)
- [Checkpointing](https://code.claude.com/docs/en/checkpointing)
- [CLI Reference](https://code.claude.com/docs/en/cli-reference)

## 最终记忆口诀

```text
规则写 CLAUDE.md，局部规则放 rules；
流程封成 Skill，复杂任务交 Agent；
外部系统接 MCP，强制动作靠 Hook；
整套能力做 Plugin，短期回退用 Checkpoint；
长期成果必须进 Git，所有权限坚持最小化。
```
