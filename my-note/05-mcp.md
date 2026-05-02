# 第五课：模型上下文协议（MCP）

---

## 一、MCP 是什么

MCP（Model Context Protocol）是 Claude 访问**外部工具、API 和实时数据**的标准化协议。

核心特征：实时访问、数据会变、连接外部服务。与 Memory 的根本区别：

| 对比 | MCP | Memory |
|------|-----|--------|
| 数据特点 | 实时、动态变化 | 持久、静态偏好 |
| 典型场景 | GitHub Issues、数据库查询、Slack 消息 | 用户偏好、项目规范、历史上下文 |
| 存储位置 | 外部服务 | 本地文件 |

---

## 二、三种传输协议

| 协议 | 推荐度 | 服务器位置 | 适用场景 |
|------|--------|-----------|---------|
| **HTTP** | ✅ 推荐 | 远程独立运行 | GitHub、Notion、Stripe 等云服务 |
| **Stdio** | 常用 | 本地进程（Claude 启动） | `npx @xxx/mcp-server` 本地包 |
| **SSE** | ❌ 已废弃 | 远程 | 旧版兼容，勿用 |

```bash
# HTTP（推荐，远程服务）
claude mcp add --transport http github https://api.github.com/mcp

# Stdio（本地进程）
claude mcp add --transport stdio database -- npx @company/db-server
```

---

## 三、MCP 范围与安全规则

| 范围 | 配置文件 | 共享 | 首次加载 |
|------|---------|------|---------|
| Local（默认） | `~/.claude.json`（按项目路径） | 仅自己 | 自动 |
| **Project** | `.mcp.json`（进 git） | 团队共享 | ⚠️ 需手动信任确认 |
| User | `~/.claude.json` | 跨所有项目 | 自动 |

> `.mcp.json` 首次使用触发安全审批弹窗——防止恶意 MCP 通过 git 注入。

---

## 四、常用 CLI 命令

```bash
claude mcp add --transport http github https://api.github.com/mcp  # 添加
claude mcp list          # 列出所有 MCP 服务器
claude mcp get github    # 查看某个服务器详情
claude mcp remove github # 删除
claude mcp reset-project-choices  # 重置项目审批记录
claude mcp add-from-claude-desktop # 从 Claude Desktop 导入
```

---

## 五、环境变量语法

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

| 语法 | 含义 |
|------|------|
| `${VAR}` | 使用环境变量，未设置则报错 |
| `${VAR:-default}` | 使用环境变量，未设置则用默认值 |

---

## 六、工具描述预算与 Tool Search

当 MCP 工具描述超过上下文窗口的 **10%**，**Tool Search 自动开启**：

- 不再把所有工具描述塞进上下文
- Claude 按需动态检索相关工具
- 需要 Sonnet 4 或 Opus 4+（Haiku 不支持）

工具描述上限：每个 MCP server **2 KB**（v2.1.84+）。

---

## 七、MCP 输出限制（三个关键数字）

| 数值 | 角色 | 触发行为 |
|------|------|---------|
| 10,000 tokens | 警告线 | 显示「输出较大」提示，继续执行 |
| **25,000 tokens** | **默认截断线** | 超出部分被截断 |
| 50,000 字符 | 磁盘持久化线 | 超出写入磁盘，不占内存 |

调整截断上限：`export MAX_MCP_OUTPUT_TOKENS=50000`

---

## 八、引用 MCP 资源

在对话中用 `@` 语法把 MCP 资源内容注入上下文：

```
@server-name:protocol://resource/path

# 例子
@database:postgres://mydb/users
@github:repo://owner/repo/README.md
```

MCP 斜杠命令（调用 MCP 暴露的 Prompt）：

```
/mcp__<server>__<prompt>
# 例子
/mcp__github__review
```

---

## 九、claude mcp serve（反向模式）

```bash
claude mcp serve
```

让 Claude Code **自身成为 MCP 服务器**，供其他应用连接——用于多代理编排：

```bash
# 在另一个 Claude 实例里把第一个当 MCP server 用
claude mcp add --transport stdio claude-agent -- claude mcp serve
```

---

## 十、企业管控（managed-mcp.json）

```json
{
  "allowedMcpServers": [{ "serverName": "github" }],
  "deniedMcpServers": [{ "serverUrl": "http://*" }]
}
```

**Denied 优先于 Allowed**——两者同时匹配时，拒绝规则生效。

---

## 十一、最佳实践速记

**该做的**：
- 凭证存环境变量，不硬编码进配置文件
- `.mcp.json` 进 git，敏感 token 不进 git
- 工具权限最小化

**不该做的**：
- 把 token 写死在 `.mcp.json` 里提交
- 用 root/admin 权限运行 MCP server
- 忽略首次审批弹窗直接信任陌生来源的 `.mcp.json`
