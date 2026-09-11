# skillgrp

互動式 **agent skill 群組管理**。把 `npx skills` 裝好的 skills 編成群組，平常收在 vault 裡不佔
agent 的 context，需要時一個指令載入並同步 symlink 到各家 AI agent。

Node 單檔、零依賴。

## 為什麼

`npx skills` 會把 skill 實體放在 `~/.agents/skills/`，再 symlink 到 `~/.claude/skills`、
`~/.codex/skills` 等各家 agent 目錄 —— 一次同步到所有工具。但它是全有全無：裝了就永遠在線，
每個 session 的 skill 描述清單都掛著，不同情境的 skill 還會互相誤觸發。

skillgrp 加上一層「群組開關」，實體只有一份，靠搬移 + symlink 控制可見性。

## 佈局

```
~/.agents/skills/<skill>          已載入（npx skills 認得的位置）
~/.agents/vault/<群組>/<skill>    已編組但未載入
~/.agents/skill-groups.json       群組定義（成員名單的唯一來源）
~/.<agent>/skills/<skill>         各家 agent 的相對 symlink
```

- **載入** = vault 搬回 `skills/` + 連結各家 agent
- **卸載** = 拔掉各家 agent 連結 + `skills/` 搬回 vault

實體永遠只有一份，不複製。未編組的 skill 一律常駐在線。

## 安裝

```bash
cp skillgrp ~/.local/bin/skillgrp && chmod +x ~/.local/bin/skillgrp
```

或在本目錄 `npm link`。需要 Node >= 18。

## 用法

直接執行進入互動介面：

```bash
skillgrp
```

```
群組 (~/.agents)
  ○ 未載入  blog (3)
      ○ blog-poster
      ○ blog-reviewer
      ○ feynman-blog

未編組 (常駐在線)
  ● figma-work
  ● test-api

同步對象：~/.claude  ~/.codex

skillgrp
      ❯ 載入 / 卸載群組
        建立群組
        編輯群組（成員 / 改名 / 刪除）
        重新整理檢視
        離開
↑↓ 移動  Enter 確認  Ctrl-C 離開
```

- **建立群組** — 輸入名稱 → 從「未編組」多選（空白鍵勾選、`a` 全選/全不選）→ 問要不要立即卸載（預設是）
- **編輯群組** — 增減成員 / 改名（vault 目錄跟著改）/ 刪除（成員搬回並重新連結，不刪 skill 本身）

### 子指令

| 指令 | 說明 |
| --- | --- |
| `skillgrp ls` | 列出群組與狀態 |
| `skillgrp load <群組>...` | 載入 |
| `skillgrp unload <群組>...` | 卸載 |
| `skillgrp sync` | 重新連結所有已載入的 skill 到各家 agent |
| `skillgrp update` | 全部載入 → `npx skills update` → 還原原本狀態 |
| `skillgrp create <群組> <skill>...` | 非互動建立群組（建立後仍是常駐，不會自動卸載） |
| `skillgrp rename <舊名> <新名>` | 群組改名 |
| `skillgrp delete <群組>`（別名 `rm`） | 刪除群組，成員搬回 `skills/` 並載入，不刪 skill 本身 |
| `skillgrp add <群組> <skill>...` | 加入成員 |
| `skillgrp drop <群組> <skill>...` | 移除成員 |
| `skillgrp doctor` | 健康檢查：孤兒 vault 目錄、群組成員遺失、skill 沒連好所有 agent |
| `skillgrp ls --json` | 機器可讀的狀態輸出，方便串接其他腳本 |
| `skillgrp --version` / `-v` | 顯示版本號 |

以上非互動指令跟互動介面共用同一套邏輯，適合寫進 dotfiles 初始化腳本或 CI。

> 切換後需**開新的 agent session** 才會生效（skill 清單在 session 啟動時掃描）。

## 跟 `npx skills` 的相處

卸載中的 skill 不在 `~/.agents/skills`，`npx skills update` 會找不到它們。本工具完全不碰
`.skill-lock.json`，改用 `skillgrp update` 包一層：先載入全部 → 跑 update → 再還原開關狀態。

`add` / `remove` 照舊用 `npx skills`，新裝的會出現在「未編組」。

## 設定

`~/.agents/skill-groups.json` 的 `agentDirs` 可覆寫同步對象（相對於 `$HOME` 的目錄，可以是多層路徑）：

```json
{
  "version": 1,
  "agentDirs": [".claude", ".codex"],
  "groups": { "blog": { "skills": ["blog-poster", "blog-reviewer"] } }
}
```

預設會嘗試 `.claude` `.codex` `.cursor` `.gemini` `.config/opencode` `.codeium/windsurf`，不存在的自動略過。
其他 agent 需要的話，照 [`npx skills` 官方對照表](https://github.com/vercel-labs/skills#supported-agents) 的
Global Path 欄位自己加進 `agentDirs`。

部分 agent 的 skills 目錄不是 `~/.<agent>/skills`，而是巢狀更深，例如 Antigravity CLI 是
`~/.gemini/antigravity-cli/skills`。`agentDirs` 的每一項會直接接上 `skills`，所以巢狀路徑照樣
可以寫，該目錄要先存在（`mkdir -p`）才會被偵測到：

```json
{
  "version": 1,
  "agentDirs": [
    ".claude", ".codex", ".cursor", ".gemini", ".config/opencode", ".codeium/windsurf",
    ".gemini/antigravity-cli"
  ],
  "groups": {
    "blog": { "skills": ["blog-poster", "blog-reviewer", "blog-workflow", "feynman-blog"] },
    "third-party": { "skills": ["herdr", "opencli-browser", "opencli-explorer"] }
  }
}
```

環境變數 `SKILLGRP_HOME` 可指向別的 HOME，方便在沙箱試跑。

## 安全性

- 只搬 skill 目錄、只動 symlink，不刪任何 skill 內容
- 遇到非 symlink 的實體檔案會跳過不覆蓋
- 跨檔案系統搬移有 EXDEV fallback（copy + remove）
