# 第二课：记忆系统（Memory）

---

## 一、三个核心命令

| 命令 | 作用 | 适用场景 |
|------|------|----------|
| `/init` | 分析项目结构，生成 CLAUDE.md 模板 | 新项目首次初始化（一次性） |
| `/memory` | 打开系统编辑器直接编辑记忆文件 | 批量修改、整理已有内容 |
| `@path/to/file` | 在 CLAUDE.md 中引用外部文件 | 避免重复、复用现有文档 |

> 1. 用 `/memory` 打开编辑器手动编辑
> 2. 直接对 Claude 说"记住……"，Claude 会询问写入哪个文件

---

## 二、记忆层级（优先级从高到低，共 8 层）

| 层级 | 路径 | 范围 | 进 Git？ |
|------|------|------|---------|
| 1. Managed Policy | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS） | 公司全局 | 系统级 |
| 2. Managed Drop-ins | 同目录 `managed-settings.d/`（v2.1.83+） | 公司模块化 | 系统级 |
| 3. Project Memory | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享 | ✅ 提交 |
| 4. Project Rules | `./.claude/rules/*.md` | 团队（路径限定） | ✅ 提交 |
| 5. User Memory | `~/.claude/CLAUDE.md` | **个人，所有项目** | ❌ |
| 6. User Rules | `~/.claude/rules/*.md` | 个人，所有项目 | ❌ |
| 7. Local Project Memory | `./CLAUDE.local.md` | **个人，仅当前项目** | ❌ 加 .gitignore |
| 8. Auto Memory | `~/.claude/projects/<project>/memory/` | Claude 自动写入 | ❌ |

**关键区分（易混淆）**：

| 需求 | 选哪里 | 原因 |
|------|--------|------|
| 个人偏好，对所有项目生效 | `~/.claude/CLAUDE.md`（层级 5） | User Memory |
| 个人偏好，仅对当前项目生效，不进 git | `./CLAUDE.local.md`（层级 7） | Local Project Memory |

---

## 三、Auto Memory（重点）

Claude 在会话中**自动**写入，无需手动维护。

```
~/.claude/projects/<project>/memory/
├── MEMORY.md              # 入口（启动时加载前 200 行 / 25KB）
├── debugging.md           # 主题文件（按需加载）
└── api-conventions.md     # 主题文件（按需加载）
```

**控制 Auto Memory 的方式**：通过环境变量，不是配置文件

```bash
CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 claude   # 禁用
CLAUDE_CODE_DISABLE_AUTO_MEMORY=0 claude   # 强制开启
# 不设置 = 默认开启
```

**自定义路径**（v2.1.74+，只能写在用户级或 local 设置里）：

```jsonc
// ~/.claude/settings.json
{ "autoMemoryDirectory": "/path/to/custom/memory" }
```

---

## 四、模块化规则（Rules）+ 路径限定

在 `.claude/rules/` 下创建文件，支持子目录和符号链接：

```
.claude/rules/
├── code-style.md
└── api/
    └── validation.md      # 子目录递归发现
```

**路径限定**：在文件顶部加 YAML Frontmatter，规则只对指定路径生效：

```markdown
---
paths: src/api/**/*.ts
---

# API 规则（仅对 src/api/ 下的 .ts 文件有效）
- 必须使用 Zod 做 schema 校验
```

---

## 五、@import 语法

```markdown
@README.md                          # 相对路径
@docs/architecture.md
@~/.claude/my-instructions.md       # 绝对路径
```

- 最多 5 层递归嵌套
- 首次引用外部路径触发权限确认弹窗
- 代码块内的 `@` 不会被解析（可以安全地写示例）

---

## 六、多目录加载（--add-dir）

想让 Claude 同时加载另一个仓库的 CLAUDE.md：

```bash
# 先设置环境变量
CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1

# 启动时加参数
claude --add-dir /path/to/other/project
```

---

## 七、排除不相关的 CLAUDE.md

大型单仓库场景，跳过不需要的 CLAUDE.md：

```jsonc
// .claude/settings.json
{
  "claudeMdExcludes": [
    "packages/legacy-app/CLAUDE.md",
    "vendors/**/CLAUDE.md"
  ]
}
```

---

## 八、最佳实践速记

**该放的**：具体可操作的规则、团队约定、常用命令、`@path` 引用现有文档

**不该放的**：API key / 密码、PII、单文件超 500 行、代码架构细节（从代码读即可）
