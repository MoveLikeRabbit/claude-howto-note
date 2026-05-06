# 第八課：檢查點與回退（Checkpoints & Rewind）

---

## 一、Checkpoint 是什麼

Checkpoint 是**對話狀態的快照**，包含：

- 所有對話消息
- 文件修改記錄
- 工具使用歷史
- 會話上下文

核心價值：讓你可以大膽實驗，任何時候都能回到之前的狀態。

---

## 二、⚠️ 自動創建（不需要手動保存）

Claude Code **在每一次用戶輸入時自動創建** Checkpoint：

| 特性 | 說明 |
|------|------|
| 觸發時機 | 每次用戶發送消息 |
| 持久性 | 跨會話保存 |
| 保留時長 | 默認 30 天自動清理 |

**記憶要點**：不需要手動創建，專注於工作即可。

---

## 三、訪問 Checkpoint 的兩種方式

```bash
# 方式一：鍵盤快捷鍵
Esc + Esc          ← 打開 checkpoint 瀏覽器

# 方式二：斜杠命令
/rewind            ← 主命令
/checkpoint        ← 別名，效果相同
```

---

## 四、⚠️ 五個回退選項（Rewind Options）

打開 checkpoint 瀏覽器後，選中某個 checkpoint，會出現 5 個選項：

| # | 選項 | 效果 |
|---|------|------|
| 1 | **Restore code and conversation** | 代碼 + 對話全部還原 |
| 2 | **Restore conversation** | 只還原對話，代碼保留現狀 |
| 3 | **Restore code** | 只還原代碼，對話保留完整歷史 |
| 4 | **Summarize from here** | 從此點壓縮後續對話為摘要（釋放上下文窗口空間） |
| 5 | **Never mind** | 取消，什麼都不做 |

> 還原後，被選中 checkpoint 的原始 prompt 會自動填回輸入框，可直接重發或編輯。

---

## 五、⚠️ 限制（不追蹤的內容）

| 操作類型 | 是否追蹤 |
|----------|---------|
| Claude 修改的文件 | ✅ 追蹤 |
| Bash 命令（rm/mv/cp 等） | ❌ **不追蹤** |
| Claude Code 外部的文件修改 | ❌ **不追蹤** |

**記憶口訣**：只追蹤 Claude 自己做的文件修改，Shell 命令的副作用不在範圍內。

---

## 六、Checkpoint vs Git

| 對比項 | Git | Checkpoint |
|--------|-----|-----------|
| 範圍 | 文件系統 | 對話 + 文件 |
| 持久性 | 永久 | 基於會話（30天） |
| 粒度 | 手動 commit | 每次消息自動 |
| 速度 | 較慢 | 即時 |
| 共享 | ✅ | 有限 |

**最佳組合**：用 Checkpoint 快速實驗，用 Git commit 固化最終成果。

---

## 七、核心工作流模式

### 多方案探索
```
Checkpoint A（起點）
  ├── → 方案一 → 評估
  └── ← 回退到 A → 方案二 → 評估 → 選最優
```

### 安全重構
```
自動 Checkpoint → 開始重構 → 跑測試
  ├── 測試通過 → 繼續 → git commit
  └── 測試失敗 → 回退 → 換思路
```

### 上下文壓縮（"Summarize from here"）
```
長對話佔用大量上下文窗口
→ Esc+Esc 選早期 Checkpoint
→ 選 "Summarize from here"（可選填摘要重點）
→ 對話被壓縮為摘要，原始消息仍保留在轉錄文件中
→ 繼續工作，上下文窗口恢復空間
```

---

## 八、配置（唯一相關設置）

```json
{
  "cleanupPeriodDays": 30
}
```

`cleanupPeriodDays` 同時管理 4 個本地緩存目錄的保留時長：

| 目錄 | 內容 |
|------|------|
| Session checkpoints | Checkpoint 快照 |
| `~/.claude/tasks/` | 持久化任務列表 |
| `~/.claude/shell-snapshots/` | Shell 環境快照 |
| `~/.claude/backups/` | settings/CLAUDE.md 備份 |

（v2.1.117 起，一個設置統一控制四個目錄）

---

## 九、何時使用 Checkpoint（決策樹）

| 場景 | 建議操作 |
|------|---------|
| 想嘗試不同實現方案 | ✅ 標記當前 checkpoint，自由實驗 |
| 重構出問題想恢復 | ✅ Esc+Esc → Restore code and conversation |
| 對話太長影響質量 | ✅ Summarize from here |
| 只想撤銷代碼但保留對話 | ✅ Restore code |
| 永久保存代碼成果 | ❌ 用 git commit，不要只靠 Checkpoint |
| Bash 命令執行的後果 | ❌ Checkpoint 無法幫你，需手動處理 |

---

## 十、上下文狀態監控（進階）

推薦工具 **[cc-context-stats](https://github.com/luongnv89/cc-context-stats)**：在狀態欄顯示實時上下文用量分區。

| 顯示區域 | 含義 | 建議操作 |
|---------|------|---------|
| Plan（綠） | 安全，可規劃和寫代碼 | 正常工作 |
| Code（黃） | 避免開始新計劃 | 完成當前任務 |
| Dump（橙） | 趕快收尾 | Summarize or Rewind |

---

## 十一、常見問題排查

| 問題 | 原因 | 解法 |
|------|------|------|
| 找不到預期 Checkpoint | 被清理或磁盤空間不足 | 調高 `cleanupPeriodDays`，檢查磁盤 |
| 回退失敗 | Checkpoint 損壞 | 嘗試更早的 Checkpoint |
| Bash 副作用沒還原 | 設計如此，不追蹤 Shell 操作 | 手動處理文件系統副作用 |
