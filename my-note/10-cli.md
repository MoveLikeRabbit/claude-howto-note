# 第十課：CLI 參考（CLI Reference）

---

## 一、兩種執行模式

| 模式 | 觸發方式 | 特點 |
|------|---------|------|
| **Interactive REPL** | `claude` 或 `claude "query"` | 多輪對話、Tab 補全、歷史記錄、斜杠命令 |
| **Print Mode（非互動）** | `claude -p "query"` | 單次查詢、可腳本化、可管道、支持 JSON 輸出 |

```bash
# 互動模式
claude
claude "解釋這個項目的架構"

# Print 模式（CI/CD 自動化用）
claude -p "解釋這個函數"
cat error.log | claude -p "分析這些錯誤"
claude -p "列出所有 TODO" | grep "URGENT"
```

---

## 二、⚠️ 核心標誌速查

| 標誌 | 說明 |
|------|------|
| `-p, --print` | Print 模式 |
| `-c, --continue` | 繼續最近的對話 |
| `-r, --resume <name>` | 按名稱或 ID 恢復會話 |
| `-w, --worktree` | 在隔離 git worktree 中啟動 |
| `-n, --name <title>` | 為會話設置顯示名稱 |
| `--from-pr <url>` | 恢復與 PR/MR 關聯的會話（GitHub/GitLab/Bitbucket，v2.1.119+） |
| `--remote "task"` | 在 claude.ai 創建雲端會話 |
| `--rc, --remote-control` | 啟用遠程控制 |
| `--teleport` | 將雲端會話拉回本地終端 |
| `--bare` | 最簡模式（跳過 hooks/skills/plugins/MCP/auto memory/CLAUDE.md） |
| `--fork-session` | 恢復會話時創建分叉 |
| `--max-budget-usd <N>` | Print 模式最大費用上限 |

---

## 三、模型選擇

```bash
claude --model opus "複雜架構設計"
claude --model sonnet "實現這個功能"
claude --model haiku -p "格式化這段 JSON"
claude --model opusplan "設計並實現緩存層"   # Opus 規劃 → Sonnet 執行

# 努力等級（Opus 4.7）
claude --effort xhigh "複雜任務"
/effort high                                  # 會話中切換
export CLAUDE_CODE_EFFORT_LEVEL=xhigh

# 備用模型（防過載）
claude -p --model opus --fallback-model sonnet "分析架構"
```

**Opus 4.7 努力等級**：low（○）→ medium（◐）→ high（●）→ xhigh（默認）→ max（僅 Opus 4.7）

---

## 四、系統提示自訂

| 標誌 | 行為 | 可用模式 |
|------|------|---------|
| `--system-prompt` | 替換整個默認系統提示 | 互動 + Print |
| `--system-prompt-file` | 從文件替換系統提示 | **僅 Print** |
| `--append-system-prompt` | 追加到默認系統提示 | 互動 + Print |

> ⚠️ `--system-prompt-file` 只能在 Print 模式使用。

---

## 五、⚠️ 工具與權限管理

```bash
# 限制可用工具
claude -p --tools "Bash,Edit,Read" "query"

# 無需詢問即執行的工具
claude --allowedTools "Bash(git log:*)" "Read"

# 禁用工具
claude --disallowedTools "Bash(rm -rf:*)" "Edit"

# 只讀審計（plan 模式 + 限制工具）
claude --permission-mode plan --tools "Read,Grep,Glob" "審計代碼安全"
```

> v2.1.119+：PowerShell 工具命令支持與 Bash 相同的自動批准語法：`PowerShell(Get-ChildItem:*)`

---

## 六、輸出格式

```bash
# 文本（默認）
claude -p "解釋這段代碼"

# JSON（供腳本使用）
claude -p --output-format json "列出 main.py 中的所有函數"

# 流式 JSON（實時處理）
claude -p --output-format stream-json "生成長報告"

# JSON Schema 驗證
claude -p --json-schema '{"type":"object","properties":{"bugs":{"type":"array"}}}' \
  "找出代碼中的 bug，以 JSON 返回"

# jq 管道處理
claude -p --output-format json "列出所有 API 端點" | jq '.endpoints[]'
claude -p --output-format json "檢查安全" | jq 'if .vulnerabilities | length > 0 then "UNSAFE" else "SAFE" end'
```

---

## 七、會話管理

```bash
claude -c                                    # 繼續最近對話
claude -r "feature-auth" "繼續實現登錄"      # 按名稱恢復
claude --resume feature-auth --fork-session  # 恢復並分叉（實驗不同方案）
claude --session-id "550e8400-..." "continue" # 按 UUID 恢復
```

---

## 八、Agents 配置（--agents 標誌）

```json
{
  "agent-name": {
    "description": "必填：何時調用此代理",
    "prompt": "必填：代理的系統提示",
    "tools": ["可選", "工具", "列表"],
    "model": "可選：sonnet|opus|haiku"
  }
}
```

```bash
# 從文件加載
claude --agents "$(cat ~/.claude/agents.json)" "審計認證模塊"
```

**優先級**：CLI `--agents` > 用戶級 `~/.claude/agents/` > 項目級 `.claude/agents/`

---

## 九、高價值用例

### CI/CD 集成

```yaml
# GitHub Actions
- name: Run Code Review
  run: |
    claude -p --output-format json \
      --max-turns 1 \
      "審查此 PR 的安全漏洞和代碼質量，以 JSON 返回" > review.json
```

### 批量處理

```bash
# 批量處理多個文件
for file in src/*.ts; do
  claude -p --model haiku "總結此文件：$(cat $file)" >> summaries.md
done
```

### JSON API 集成

```bash
RESULT=$(claude -p --output-format json "代碼是否安全？返回 {secure: bool, issues: []}" < code.py)
if echo "$RESULT" | jq -e '.secure == false' > /dev/null; then
  echo "發現安全問題！"
fi
```

---

## 十、Print Mode 常用標誌組合

| 用途 | 命令 |
|------|------|
| 快速代碼審查 | `cat file \| claude -p "review"` |
| 結構化輸出 | `claude -p --output-format json "query"` |
| CI/CD 集成 | `claude -p --max-turns 3 --output-format json` |
| 預算限制運行 | `claude -p --max-budget-usd 2.00 "analyze"` |
| 最小化模式 | `claude --bare "quick query"` |
| 安全自動化 | `claude --permission-mode auto --enable-auto-mode` |

---

## 十一、⚠️ 重要環境變量

| 變量 | 說明 |
|------|------|
| `ANTHROPIC_API_KEY` | API 認證密鑰 |
| `ANTHROPIC_MODEL` | 覆蓋默認模型 |
| `CLAUDE_CODE_EFFORT_LEVEL` | 努力等級（xhigh 為 Opus 4.7 默認） |
| `MAX_THINKING_TOKENS` | 擴展思考 token 預算 |
| `CLAUDE_CODE_DISABLE_CRON` | 禁用排程任務 |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | 啟用實驗性 Agent Teams |
| `CLAUDE_CODE_TASK_LIST_ID` | 跨會話共享任務列表 |
| `DISABLE_UPDATES` | 阻止所有更新路徑（含手動 `claude update`）（v2.1.118+） |
| `CLAUDE_CODE_HIDE_CWD` | 設為 `1` 時隱藏啟動 logo 的工作目錄（屏幕分享隱私）（v2.1.119+） |
| `CLAUDE_CODE_PERFORCE_MODE` | 設為 `1` 啟用 Perforce 模式（文件默認只讀） |
| `ENABLE_PROMPT_CACHING_1H` | 使用 1 小時 prompt cache（默認 5 分鐘） |
| `ENABLE_TOOL_SEARCH` | Vertex AI 上默認關閉，需顯式設為 `true` |

---

## 十二、Runtime 注意（v2.1.113+）

- CLI 現在啟動**原生平台二進制文件**（macOS/Linux/Windows），不再默認使用 JavaScript bundle
- 安裝命令不變：`npm install -g @anthropic-ai/claude-code`
- 二進制下載源（v2.1.116+）：`https://downloads.claude.ai/claude-code-releases`
- ⚠️ 企業/代理用戶：需將 `downloads.claude.ai` 加入代理白名單，否則 `claude update` 會失敗

---

## 十三、常見故障排除

| 問題 | 解法 |
|------|------|
| `claude: command not found` | `npm install -g @anthropic-ai/claude-code`；檢查 PATH |
| 認證失敗 | `export ANTHROPIC_API_KEY=your-key` |
| 找不到會話 | 用 `-c` 繼續最近會話；會話可能已過期 |
| JSON 輸出格式錯誤 | 用 `--json-schema` 強制結構；在 prompt 中明確要求 JSON |
| 工具執行被阻止 | 檢查 `--permission-mode` 和 `--allowedTools` 設置 |
| `/doctor` | 在 REPL 中輸入 `/doctor`，按 `f` 自動修復問題（v2.1.116+） |
