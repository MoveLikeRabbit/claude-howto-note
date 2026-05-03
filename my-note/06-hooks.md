# 第六课：钩子系统（Hooks）

---

## 一、Hooks 是什么

事件触发的自动化脚本——Claude 做某件事时，预先配置的脚本自动跑起来。

类比 git hooks（pre-commit），但触发点是 Claude 的行为。

---

## 二、四种 Hook 类型

| 类型 | 执行什么 | 典型用途 |
|------|---------|---------|
| **command** | Shell 脚本 | 验证、格式化、安全检查（最常用） |
| **http** | POST 到 webhook URL | 通知外部系统 |
| **prompt** | LLM 评估一段提示词 | 智能判断任务是否完成 |
| **agent** | 启动子代理做复杂检查 | 多步骤验证 |

---

## 三、退出码——核心通信机制

| 退出码 | 含义 | Claude 的反应 |
|--------|------|--------------|
| **0** | 成功 | 读取 stdout 的 JSON，继续执行 |
| **2** | **阻断错误** | stderr 内容作为错误显示，**停止工具调用** |
| 其他非 0 | 非阻断警告 | 继续执行，stderr 只在 verbose 模式显示 |

> 只有退出码 2 才能真正拦截 Claude 的操作。

---

## 四、stdin / stdout 是什么

每个程序天生就有三个管道，不用创建直接用：

| 管道 | 方向 | Hook 里的作用 |
|------|------|--------------|
| **stdin** | 流进来 | Claude → 脚本，传工具名/输入参数等 JSON |
| **stdout** | 流出去 | 脚本 → Claude，传决策结果（用 `print()` 写） |
| **stderr** | 流出去（错误专用） | 退出码 2 时，内容显示给 Claude 看 |

```python
import json, sys

data = json.load(sys.stdin)        # 读 stdin（Claude 传来的 JSON）
print(json.dumps({"continue": True}))  # 写 stdout（Claude 自动读取）
print("出错了", file=sys.stderr)   # 写 stderr（退出码2时显示）
```

---

## 五、PreToolUse 的 stdin JSON 结构

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "rm -rf /tmp" },
  "cwd": "/my/project",
  "transcript_path": "/path/to/transcript.jsonl"
}
```

---

## 六、⚠️ PreToolUse 如何修改工具输入（易错点）

**不是写临时文件，而是在 stdout 返回含 `updatedInput` 的 JSON：**

```python
import json, sys

data = json.load(sys.stdin)

# 修改路径示例
modified = dict(data["tool_input"])
modified["file_path"] = "/safe/" + modified.get("file_path", "")

result = {
    "hookSpecificOutput": {
        "hookEventName": "PreToolUse",
        "updatedInput": modified           # ← 这里放修改后的参数
    }
}
print(json.dumps(result))  # 写到 stdout，退出码必须是 0
sys.exit(0)
```

**Hook 和 Claude 的唯一通信渠道是 stdin/stdout，不经过文件。**

---

## 七、⚠️ CLAUDE_ENV_FILE 只在 SessionStart 可用（易错点）

`CLAUDE_ENV_FILE` 允许把环境变量持久化到当前会话，**只有 SessionStart 事件支持**（以及 `CwdChanged`、`FileChanged`）：

```bash
#!/bin/bash
# SessionStart hook 脚本
if [ -n "$CLAUDE_ENV_FILE" ]; then
    echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

PreToolUse 不支持——环境变量要在会话初始化时注入才有意义，工具执行时注入太晚了。

---

## 八、⚠️ 子代理 frontmatter 里的 Stop Hook 自动转换（易错点）

在子代理的 AGENT.md frontmatter 里定义 `Stop` Hook，它会**自动转换为 `SubagentStop`**：

```yaml
---
name: code-review-agent
hooks:
  Stop:                          # ← 写 Stop
    - hooks:
        - type: prompt
          prompt: "验证代码审查是否完整"
  # 实际生效的是 SubagentStop——只在这个子代理停止时触发
  # 不会影响主会话的 Stop 事件
---
```

记忆口诀：**子代理的 Stop = 主会话的 SubagentStop**。

---

## 九、配置结构速查

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
            "timeout": 60,
            "once": true
          }
        ]
      }
    ]
  }
}
```

| 字段 | 说明 |
|------|------|
| `matcher` | 匹配工具名，支持正则：`"Write\|Edit"`、`"mcp__github__.*"`、`"*"` |
| `type` | command / http / prompt / agent |
| `timeout` | 超时秒数，默认 60 |
| `once` | `true` = 整个会话只跑一次（组件 Hook 专用） |

**配置文件位置决定生效范围：**

| 文件 | 范围 |
|------|------|
| `~/.claude/settings.json` | 所有项目 |
| `项目/.claude/settings.json` | 当前项目（可进 git） |
| `项目/.claude/settings.local.json` | 当前项目（不进 git，个人用） |

---

## 十、重要事件速查（28 个总计）

| 事件 | 能否阻断 | 最常用场景 |
|------|---------|----------|
| **PreToolUse** | ✅ | 拦截危险命令、修改输入 |
| **PostToolUse** | ❌ | 自动格式化、安全扫描 |
| **UserPromptSubmit** | ✅ | 过滤危险提示词 |
| **Stop** | ✅ | 检查任务是否真完成 |
| **SessionStart** | ❌ | 初始化环境变量（`CLAUDE_ENV_FILE`） |
| **SubagentStop** | ✅ | 验证子代理结果 |

---

## 十一、调试

```bash
claude --debug          # 看 hook 执行详细日志
# Ctrl+O 进入 verbose 模式

# 单独测试脚本：
echo '{"tool_name":"Bash","tool_input":{"command":"ls"}}' | python3 .claude/hooks/validate.py
echo $?                 # 查看退出码
```
