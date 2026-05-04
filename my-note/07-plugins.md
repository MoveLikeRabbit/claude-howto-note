# 第七課：插件系統（Plugins）

---

## 一、插件是什麼

把多個 Claude Code 功能（命令、代理、MCP、Hooks 等）打包成**一條命令可安裝**的可分發單元。

核心價值：與獨立配置的對比

| 對比項 | 獨立配置 | 插件 |
|--------|---------|------|
| 安裝時間 | 2+ 小時（逐一配置） | 2 分鐘（一條命令） |
| 版本管理 | 手動 | 自動 |
| 團隊共享 | 複製文件（容易出錯） | 安裝 ID |
| 市場分發 | 無 | ✅ 有 |

---

## 二、⚠️ 目錄結構（易錯點）

```
my-plugin/
├── .claude-plugin/          ← ⚠️ 注意：是 .claude-plugin/ 不是 .claude/
│   └── plugin.json          ← manifest 文件在這裡
├── commands/                # 命令（Skills）
├── agents/                  # 子代理
├── skills/                  # Skills
├── hooks/
│   └── hooks.json           ← ⚠️ hooks 配置在 hooks/hooks.json
├── .mcp.json                # MCP 配置
├── .lsp.json                # LSP 配置
├── settings.json            # 插件默認設置（目前支持 agent key）
├── bin/                     # 執行文件（自動加入 PATH）
├── templates/
└── scripts/
```

**記憶口訣**：manifest → `.claude-plugin/plugin.json`；hooks → `hooks/hooks.json`

---

## 三、Manifest 最小結構

```json
{
  "name": "my-plugin",
  "description": "做什麼",
  "version": "1.0.0",
  "author": { "name": "Your Name" }
}
```

---

## 四、⚠️ 命令命名空間（易錯點）

插件命令用 **冒號** 分隔，不是斜杠：

```bash
/pr-review:check-security     ← ✅ 正確：插件名:命令名
/pr-review/check-security     ← ❌ 錯誤（路徑格式）
/plugin pr-review check-security  ← ❌ 錯誤（不是這樣調用）
```

---

## 五、⚠️ 安裝與測試（易錯點）

```bash
# 本地開發測試（不是 /plugin test！）
claude --plugin-dir ./my-plugin
claude --plugin-dir ./plugin-a --plugin-dir ./plugin-b  # 可重複

# 市場安裝
/plugin install plugin-name
/plugin install github:username/repo    # 從 GitHub

# CLI 等效
claude plugin install plugin-name@marketplace-name
```

**熱重載**：修改插件文件後用 `/reload-plugins` 刷新，無需重啟會話。

---

## 六、生命週期管理

```bash
/plugin enable plugin-name    # 啟用
/plugin disable plugin-name   # 停用
/plugin update plugin-name    # 更新
/plugin uninstall plugin-name # 卸載
/plugin list --installed      # 查看已安裝
claude plugin tag v0.3.0      # 打版本 tag（v2.1.118+，驗證格式並創建 git tag）
```

---

## 七、關鍵環境變量

| 變量 | 說明 |
|------|------|
| `${CLAUDE_PLUGIN_ROOT}` | 插件安裝目錄，用於 hooks/MCP 配置裡的路徑引用 |
| `${CLAUDE_PLUGIN_DATA}` | 插件持久化數據目錄（v2.1.78+，跨會話保存） |

---

## 八、插件可打包的全部組件

commands / agents / skills / hooks / MCP（.mcp.json）/ LSP（.lsp.json）/ settings.json / templates / scripts / docs / tests / themes（v2.1.118+）

---

## 九、settings.json 的 agent key

```json
{ "agent": "agents/specialist-1.md" }
```

設置插件啟用時的**主線程代理**（main thread agent）。

---

## 十、企業管控

| 設置 | 作用 |
|------|------|
| `enabledPlugins` | 默認啟用的插件白名單 |
| `deniedPlugins` | 禁止安裝的插件黑名單 |
| `strictKnownMarketplaces` | 限制允許的市場來源 |
| `blockedMarketplaces` | 屏蔽的市場（v2.1.119 起支持正則） |

**安全限制**：插件子代理的 frontmatter 不允許 `hooks`、`mcpServers`、`permissionMode`——防止提權。

---

## 十一、何時用插件（決策樹）

| 場景 | 建議 |
|------|------|
| 需要多個命令/代理/MCP 組合 | ✅ 用插件 |
| 團隊共享/企業分發 | ✅ 用插件 |
| 單一斜杠命令 | ❌ 用 Skill 就夠 |
| 單一代理 | ❌ 直接建 agent 文件 |
| 只需實時數據 | ❌ 直接用 MCP |
