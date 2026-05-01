# 第三课：技能系统（Agent Skills）

---

## 一、Skills 是什么

Skills 是**可复用的文件系统能力单元**，将领域专业知识、工作流打包成 Claude 可自动加载的组件。

- Slash Commands 已并入 Skills 体系（`.claude/commands/` 仍兼容，但 Skills 优先级更高）
- 核心文件：`SKILL.md`，放在 `<scope>/skills/<name>/SKILL.md`

---

## 二、三层渐进式加载（Progressive Disclosure）

| 层级 | 何时加载 | Token 消耗 | 内容 |
|------|---------|-----------|------|
| Level 1：元数据 | **始终**（启动时） | ~100 tokens/个 | frontmatter 里的 `name` + `description` |
| Level 2：指令 | Skill 被触发时 | < 5k tokens | SKILL.md 正文 |
| Level 3：资源 | Claude 按需读取 | 无上限 | 脚本、模板、参考文档 |

> 安装再多 Skills 也不占上下文——只有被触发的才真正加载。

---

## 三、四种 Skill 位置（优先级从高到低）

| 类型 | 路径 | 范围 | 进 Git？ |
|------|------|------|---------|
| Enterprise | 系统管理路径 | 全组织 | 系统级 |
| Personal | `~/.claude/skills/<name>/SKILL.md` | 个人全部项目 | ❌ |
| Project | `.claude/skills/<name>/SKILL.md` | 团队共享 | ✅ |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | 插件作用域 | 取决于插件 |

同名时高优先级覆盖低优先级；Plugin Skills 用 `plugin-name:skill-name` 命名空间，不冲突。

---

## 四、SKILL.md 关键 Frontmatter 字段

```yaml
---
name: my-skill              # kebab-case，max 64 字符，不含 anthropic/claude
description: 做什么。Use when [触发场景关键词]。   # max 1024 字符，自动触发的核心
argument-hint: "[file] [format]"   # / 菜单中的参数提示
disable-model-invocation: true     # true = 只有用户能手动调用（适合有副作用的命令）
user-invocable: false              # false = 从 / 菜单隐藏，只允许 Claude 自动调用
allowed-tools: Read, Bash(git *)   # 白名单工具，无需每次弹权限确认
model: opus                        # 覆盖模型
context: fork                      # 在独立子代理中运行，不污染主对话
agent: Explore                     # 配合 context: fork 指定子代理类型
paths: "src/api/**/*.ts"           # 限定 Skill 只在特定路径下自动激活
---
```

---

## 五、调用控制三模式

| Frontmatter | 用户可手动调用 | Claude 可自动调用 |
|---|---|---|
| （默认） | ✅ | ✅ |
| `disable-model-invocation: true` | ✅ | ❌ |
| `user-invocable: false` | ❌ | ✅ |

**记忆口诀**：`disable-model-invocation` 禁的是 Claude，`user-invocable: false` 禁的是用户。

---

## 六、动态内容语法

```yaml
$ARGUMENTS          # 所有参数：/fix-issue 123 → "123"
$ARGUMENTS[0]/$0    # 按位置取参数
${CLAUDE_SKILL_DIR} # 当前 Skill 所在目录绝对路径
!`git log --oneline -5`  # Shell 命令输出注入（调用时实时执行）
```

Shell 注入安全开关（企业/CI 环境）：
```jsonc
{ "disableSkillShellExecution": true }
```

---

## 七、Reference Content vs Task Content

| 类型 | 作用 | 典型用法 |
|------|------|---------|
| Reference Content | 向上下文注入领域知识 | 品牌语气、代码规范、系统背景 |
| Task Content | 提供可执行的步骤指导 | `/deploy`、`/code-review`、`/commit` |

---

## 八、子代理运行（context: fork）

```yaml
---
context: fork
agent: Explore   # Explore（只读快）/ Plan / general-purpose / 自定义
---
```

好处：Skill 在独立上下文窗口执行，主对话不被污染。适合耗时的研究、分析类任务。

---

## 九、最佳实践速记

**该做的**：
- description 里写具体触发关键词（"Use when users ask to review code..."）
- 超 500 行的参考资料拆到 `references/` 子目录
- 有副作用的命令加 `disable-model-invocation: true`

**不该做的**：
- description 过于宽泛（"Helps with documents"）
- 一个 Skill 塞太多功能
- 安装来源不明的 Skills（等同于安装不明软件）
