# 第九課：進階功能（Advanced Features）

> 本課涵蓋規劃模式、擴展思考、自動模式、後台任務、排程、權限模式、無頭模式、會話管理、鍵盤快捷鍵等大量特性。

---

## 一、權限模式總覽（Permission Modes）

用 `Shift+Tab` 在 CLI 中循環切換，共 **6 個模式**：

| 模式 | 行為 |
|------|------|
| `default` | 只讀文件；其他操作都需詢問 |
| `acceptEdits` | 自動接受文件修改；命令仍需確認 |
| `plan` | 只讀（研究模式），不執行任何修改 |
| `auto` | 所有操作 + 後台安全分類器審查（Research Preview） |
| `bypassPermissions` | 無任何權限檢查（危險！） |
| `dontAsk` | 只執行預批准工具；其他全部拒絕 |

```bash
# CLI 啟動時指定
claude --permission-mode plan

# 設置默認值
{ "permissions": { "defaultMode": "plan" } }
```

---

## 二、Planning Mode（規劃模式）

兩階段工作流：**規劃 → 批准 → 執行**

```bash
/plan 實現用戶認證系統    # 斜杠命令

claude --permission-mode plan  # CLI 標誌
```

**特殊功能**：
- `Ctrl+G` — 在外部編輯器中編輯當前計劃
- `opusplan` 模型別名 — 用 Opus 做規劃，Sonnet 做執行

```bash
claude --model opusplan "設計並實現新 API"
```

> v2.1.112：計劃文件以 prompt 內容命名，不再是隨機詞語。

**適用場景**：複雜多文件重構、新功能、架構調整  
**不適用**：簡單 bug 修復、格式調整、單文件編輯

---

## 三、Ultraplan（雲端計劃起草）

把規劃任務交給雲端 Claude Code 會話，本地終端保持空閒。

```bash
/ultraplan 把認證服務從 session 遷移到 JWT
```

| 狀態 | 含義 |
|------|------|
| `◇ ultraplan` | 正在研究代碼庫、起草計劃 |
| `◇ ultraplan needs your input` | 需要澄清，打開鏈接回覆 |
| `◆ ultraplan ready` | 計劃已就緒，可在瀏覽器查看 |

計劃完成後可選：**雲端執行**（開 PR）或 **Teleport 回終端執行**。

> ⚠️ Remote Control 與 Ultraplan 不能同時活躍（共用同一界面）。

---

## 四、Extended Thinking（擴展思考）

深度推理，讓 Claude 在回答前花更多時間思考。

```bash
Option+T / Alt+T   # 切換開關
/effort high       # 設置努力等級
--effort high      # CLI 標誌
```

**Opus 4.7 的努力等級**：

| 等級 | 符號 |
|------|------|
| low | ○ |
| medium | ◐ |
| high | ● |
| xhigh | 默認（Opus 4.7 啟動後） |
| max | 僅 Opus 4.7 |

```bash
export CLAUDE_CODE_EFFORT_LEVEL=xhigh
export MAX_THINKING_TOKENS=16000
```

> 在 prompt 中寫 **`ultrathink`** 關鍵字可激活深度推理模式。  
> `Ctrl+O` — 切換詳細輸出（查看推理過程）。

---

## 五、Auto Mode（自動模式）

後台安全分類器逐一審查每個操作，自主執行但攔截危險行為。

**要求**：Team/Enterprise/API 方案 + Sonnet 4.6 或 Opus 4.7（不支持 Pro/Max 以外的計劃，不支持 Bedrock/Vertex/Foundry）

**默認攔截**：`curl | bash`、發送敏感數據、生產部署、`rm -rf`、IAM 更改、force push to main

**默認允許**：本地文件操作、聲明的依賴安裝（npm install 等）、只讀 HTTP、推送到當前分支

```json
{
  "autoMode": {
    "allow": ["$defaults", "Bash(gh pr list:*)"],
    "soft_deny": ["$defaults", "Bash(kubectl delete:*)"]
  }
}
```

> ⚠️ `"$defaults"` token（v2.1.118+）：用於**追加**規則而不是替換內建規則。

**Fallback**：連續 3 次或累計 20 次攔截後，自動退回詢問用戶。

---

## 六、Background Tasks（後台任務）

長時間操作不阻塞對話，繼續工作直到收到通知。

```bash
/task list            # 查看所有任務
/task status bg-1234  # 查看進度
/task show bg-1234    # 查看輸出
/task cancel bg-1234  # 取消任務
```

---

## 七、Monitor Tool（事件驅動監控）

監聽後台命令的 stdout，有匹配事件時喚醒 Claude 響應，**零 token 消耗等待**。

```bash
# 流式過濾（長期監聽）
tail -f /var/log/app.log | grep --line-buffered "ERROR"
```

> ⚠️ **必須用 `grep --line-buffered`**，否則 grep 以 4KB 緩衝輸出，低流量時延遲可達數分鐘。

---

## 八、Scheduled Tasks（排程任務）

會話內排程（Claude Code 運行時有效）：

```bash
/loop 5m check if the deployment finished
/loop check build status every 30 minutes

# 一次性提醒
remind me at 3pm to push the release branch
```

| 限制 | 值 |
|------|----|
| 每會話最多排程數 | 50 |
| 重複任務自動過期 | 3 天 |
| 未接收的觸發 | 不補發 |

```bash
export CLAUDE_CODE_DISABLE_CRON=1   # 禁用排程
```

**雲端排程**（跨重啟持久化）：
```
/schedule daily at 9am run the test suite and report failures
```

---

## 九、Headless / Print Mode（無頭模式）

非交互式執行，適用於自動化和 CI/CD：

```bash
claude -p "Run all tests"
cat error.log | claude -p "Analyze these errors"

# 常用標誌
claude -p --max-turns 5 "refactor this module"
claude -p --output-format json "analyze this codebase"
claude -p --no-session-persistence "one-off analysis"
```

---

## 十、Session Management（會話管理）

| 命令 | 說明 |
|------|------|
| `/rename auth-refactor` | 命名當前會話 |
| `/fork` | 分叉當前會話，探索另一條路 |
| `/recap` | 手動觸發回顧摘要 |
| `claude -c` | 繼續最近的對話 |
| `claude -r "auth-refactor"` | 按名稱恢復會話 |

**Session Recap**（v2.1.108）：回到會話時顯示簡短摘要。
```bash
CLAUDE_CODE_ENABLE_AWAY_SUMMARY=1 claude   # 強制開啟
```

---

## 十一、⚠️ 鍵盤快捷鍵（重要）

| 快捷鍵 | 功能 |
|--------|------|
| `Ctrl+C` | 取消當前輸入/生成 |
| `Ctrl+D` | 退出 Claude Code |
| `Ctrl+G` | 在外部編輯器打開當前計劃 |
| `Ctrl+L` | 清屏 |
| `Ctrl+O` | 切換詳細輸出（查看推理） |
| `Ctrl+R` | 搜索歷史 |
| `Ctrl+T` | 切換任務列表視圖 |
| `Ctrl+B` | 後台運行中的任務 |
| `Esc+Esc` | 回退（Rewind） |
| `Shift+Tab` / `Alt+M` | 循環切換權限模式 |
| `Option+P` / `Alt+P` | 切換模型 |
| `Option+T` / `Alt+T` | 切換擴展思考 |

**自定義快捷鍵**：`/keybindings` → 編輯 `~/.claude/keybindings.json`

---

## 十二、TUI Mode（全屏模式）

```bash
/tui          # 會話中切換
claude --tui  # 直接以 TUI 啟動
```

無閃爍全屏渲染，適合 tmux / iTerm2 分屏。

`/focus` — 專注視圖（只顯示最相關輸出）

---

## 十三、其他進階功能速覽

| 功能 | 命令/標誌 | 說明 |
|------|---------|------|
| Voice Dictation | `/voice` | 按住說話，20 種語言 STT |
| Channels | `claude --channels telegram,discord` | MCP 服務器推送外部消息到會話 |
| Chrome Integration | `/chrome` 或 `claude --chrome` | 連接瀏覽器，實時調試/自動化 |
| Remote Control | `/remote-control` | 手機/瀏覽器遠程繼續本地會話 |
| Web Sessions | `claude --remote "task"` | 在瀏覽器運行 Claude Code |
| Teleport | `/teleport` | 將雲端會話轉移回本地終端 |
| Desktop App | `/desktop` | 移交至桌面應用（可視 diff） |
| Task List | `Ctrl+T` | 跨上下文壓縮的持久任務列表 |
| Git Worktrees | `claude -w` / `claude --worktree` | 隔離分支並行工作 |
| Sandboxing | `/sandbox` 或 `claude --sandbox` | OS 級文件系統/網絡隔離 |

---

## 十四、Git Worktrees 要點

```bash
claude --worktree   # 在隔離 worktree 中啟動

# Worktree 位置：<repo>/.claude/worktrees/<name>
```

**Sparse Checkout（Monorepo）**：
```json
{ "worktree": { "sparsePaths": ["packages/my-package", "shared/"] } }
```

無修改則自動清理。

---

## 十五、Sandboxing 配置

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["/Users/me/project"],
      "denyRead": ["/Users/me/.ssh", "/Users/me/.aws"]
    },
    "network": {
      "allowedDomains": ["*.example.com"],
      "deniedDomains": ["evil.example.com"]
    }
  }
}
```

> `deniedDomains`（v2.1.113+）：可在允許通配符的前提下精確屏蔽特定主機。

---

## 十六、Managed Settings（企業管控）

| 平台 | 方式 |
|------|------|
| macOS | MDM plist 文件 |
| Windows | Windows Registry |
| 跨平台 | `managed-settings.d/` 目錄（v2.1.83+，按字母順序合併） |

常用管控項：`disableBypassPermissionsMode`、`availableModels`、`allowedChannelPlugins`

---

## 十七、Agent Teams（實驗性）

```bash
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
```

- **Team Lead** 協調，**Teammates** 各自獨立工作，共享任務列表
- 顯示模式：`in-process`（默認）/ `tmux`（各一個分屏）/ `auto`

---

## 十八、常用環境變量速查

```bash
export CLAUDE_CODE_EFFORT_LEVEL=xhigh
export MAX_THINKING_TOKENS=16000
export CLAUDE_CODE_DISABLE_CRON=1
export CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false
export CLAUDE_CODE_TASK_LIST_ID=my-project-sprint
export ENABLE_PROMPT_CACHING_1H=1     # 1 小時 prompt cache（默認 5 分鐘）
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
export CLAUDE_CODE_SIMPLE=true        # 等效 --bare 標誌
```
