# 第一课：斜杠命令与技能（Slash Commands & Skills）

---

## 一、斜杠命令的四种类型

| 类型 | 来源 | 示例 |
|------|------|------|
| 内置命令 | Claude Code 自带 | `/help`、`/clear`、`/model` |
| Skills | 用户创建 | `/optimize`、`/pr` |
| 插件命令 | 已安装插件 | `/plugin-name:command` |
| MCP 提示命令 | MCP 服务器 | `/mcp__github__list_prs` |

---

## 二、高频内置命令

| 命令 | 作用 |
|------|------|
| `/help` | 查看帮助 |
| `/clear` | 清空对话（别名 `/new`、`/reset`） |
| `/model` | 切换模型 |
| `/compact` | 压缩上下文，释放 token |
| `/memory` | 编辑 CLAUDE.md |
| `/plan` | 进入规划模式 |
| `/rewind` | 回滚代码和/或对话（别名 `/undo`） |
| `/skills` | 查看已加载的技能列表 |
| `/diff` | 查看未提交的变更 |
| `/usage` | 查看用量和费用 |

---

## 三、自动触发机制与 Frontmatter 字段

Skill 的自动触发靠 `description` 字段——Claude 在对话中识别上下文，与 description 匹配时自动调用，无需用户手动输入 `/`。

| 字段 | 作用 |
|------|------|
| `description` | 告诉 Claude **何时**触发（触发的核心） |
| `disable-model-invocation: false` | 授予 Claude 调用权限（非自动触发本身） |
| `user-invocable: false` | 从 `/` 菜单隐藏，只允许 Claude 自动调用 |

---

## 四、动态内容语法

```yaml
# 接收用户参数
$ARGUMENTS        # 所有参数作为一个字符串：/optimize a b → "a b"
$0、$1、$2...     # 按位置分割：/optimize a b → $0="a", $1="b"

# 注入 shell 命令输出（调用时实时执行）
!`git status`
!`pwd`

# 注入文件内容
@src/config.json
```

---

## 五、SKILL.md 完整示例

```yaml
---
name: optimize
# 命令名，用户输入 /optimize 触发

description: Analyze code for performance issues and suggest optimizations. Use when reviewing code for bottlenecks, memory leaks, or algorithm improvements.
# 自动触发的核心：描述越具体，Claude 判断越准确
# 格式建议："做什么。Use when [触发条件]。"

argument-hint: [file-path or paste code]
# 在 / 菜单中显示的参数提示，帮助用户知道该传什么

allowed-tools: Read, Bash(find *), Bash(wc *)
# 列出的工具无需每次弹出权限确认
# 格式：工具名(可选glob过滤)，多个用逗号分隔

disable-model-invocation: false
# false（默认）：Claude 可以自动调用此技能
# true：只允许用户手动 /optimize 触发，适合有副作用的命令（如部署）

user-invocable: true
# true（默认）：出现在 / 菜单中，用户可手动调用
# false：从菜单隐藏，仅 Claude 在后台自动调用

# context: fork
# 取消注释后，此技能在隔离子代理中运行，不污染主对话上下文
---

# Code Optimization

## Context

- Current directory: !`pwd`
# !`command` 语法：调用时实时执行 shell 命令，将输出注入提示词

- File to analyze: $ARGUMENTS
# $ARGUMENTS：接收用户传入的所有参数

## Review for the following issues in order of priority:

1. **Performance bottlenecks** - O(n²) operations, inefficient loops
2. **Memory leaks** - unreleased resources, circular references
3. **Algorithm improvements** - better algorithms or data structures
4. **Caching opportunities** - repeated computations
5. **Concurrency issues** - race conditions or threading problems

## Output Format

For each issue found:
- **Severity**: Critical / High / Medium / Low
- **Location**: file and line number
- **Explanation**: why it's a problem
- **Fix**: code example showing the improvement
```
