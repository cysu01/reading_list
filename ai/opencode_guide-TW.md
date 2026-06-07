# OpenCode — 實務工程師導覽報告

> 範圍：如何駕馭 **OpenCode**（由 SST 團隊開發的開源終端機 AI 編碼代理）、值得記住的指令與快捷鍵、agent／command／skill 與 `AGENTS.md` 的結構，以及這些元件如何組合成一套適用於小型專案的 agentic 工作流程。本文設定的讀者是已經熟悉 Claude Code 或 Aider 之類編碼代理、想快速掌握 OpenCode 原生樣貌的開發者。評測版本為 **v1.16.2（2026 年 6 月）**。
>
> 這個領域有少數事實仍有爭議或變動極快——Claude 訂閱制認證、可能的組織改名、以及設定目錄的單複數命名。這些都會在內文中明確標註，而非含糊帶過。

---

## 1. OpenCode 的形態（以及為什麼重要）

OpenCode 不是一個你直接對話的單一執行檔，而是一套**主從式（client/server）系統**。一個無頭（headless）的 server 持有 agent 迴圈、工具執行、session 與 HTTP + SSE API；各種 client 連上它。這個架構選擇正是 OpenCode 與 Claude Code、Aider 的根本差異，並且影響到下游的一切：你可以把引擎跑在遠端機器上、用本機 TUI 連上去，用 SDK 驅動它，或是把一個即時 session 的分享連結交給隊友。

| | **它是什麼** | **備註** |
|---|---|---|
| Server／引擎 | TypeScript 跑在 **Bun** 上；agent 迴圈、工具、MCP、session、HTTP+SSE API | 可透過 `opencode serve` 獨立執行 |
| TUI client | 以 **Go** 撰寫；預設前端 | 直接輸入 `opencode` 得到的就是它 |
| Web client | 由 `opencode web` 提供的瀏覽器 UI；也用來呈現分享的 session | 同一引擎、不同前端 |
| IDE 擴充套件 | VS Code／Cursor／Windsurf／VSCodium | 在整合終端機中執行 `opencode` 時自動安裝 |
| SDK | `@opencode-ai/sdk`（TypeScript，以 Stainless 產生） | 覆蓋 server API 的型別安全 client |
| ACP | `opencode acp` 暴露 **Agent Client Protocol** | 讓支援 ACP 的編輯器（如 Zed）連入 |

與顯而易見的替代方案相比：

- **vs Claude Code**——不綁定單一廠商；**MIT 授權、開源**；引擎可與前端拆分；以 `AGENTS.md` 作為主要指令檔（並以 `CLAUDE.md` 作為備援）。代價是你得自行組裝 provider／model 方案，而非取得一套已整理好的堆疊。
- **vs Aider**——具備完整 agentic 工具迴圈、subagent、MCP 與真正的 TUI，而不是以 git commit 為中心的 REPL。
- **vs Cursor**——以終端機／CLI 為先，而非分叉一個編輯器，不過它也提供 IDE 擴充套件以取得行內體驗。

設計上即 provider 無關：它建構在 Vercel **AI SDK** 與 **models.dev** 登錄之上，宣稱支援 **75+ providers**（Anthropic、OpenAI、Gemini、DeepSeek、Qwen、Bedrock，以及任何 OpenAI 相容端點，含 Ollama、llama.cpp、LM Studio）。

> **組織名稱注意事項。** 標準 repo 為 `github.com/sst/opencode`（MIT）。多個次級來源指出維護組織已改名為 **「Anomaly Innovations」／`anomalyco`**（Homebrew tap 與 Docker image 都使用 `anomalyco`）。請以 `sst/opencode` 為主要識別；引用改名一事前先行查證。

### 先決條件

- 一個現代終端機模擬器——WezTerm、Alacritty、Ghostty、Kitty 是建議組合，以確保正確繪製。
- 一把 LLM provider API key 或受支援的訂閱方案。

### 安裝

```bash
curl -fsSL https://opencode.ai/install | bash      # 主要安裝器
npm install -g opencode-ai                          # npm——套件名稱為 opencode-ai
brew install anomalyco/tap/opencode                 # Homebrew（tap 在 anomalyco 下）
sudo pacman -S opencode                             # Arch
choco install opencode                              # Windows／Chocolatey
docker run -it --rm ghcr.io/anomalyco/opencode      # Docker
```

`cd` 進專案後執行 `opencode`。首次執行時，`/init` 會掃描 repo 並產生或更新 `AGENTS.md`。維護：`opencode upgrade [target]`（`--method/-m`）與 `opencode uninstall`（`--keep-config/-c`、`--keep-data/-d`、`--dry-run`、`--force/-f`）。

---

## 2. CLI 介面（Claude Code 使用者最容易低估的部分）

因為引擎是無頭的，`opencode` CLI 是一等公民介面，而非只是啟動器。你實際會用到的子命令：

| 指令 | 作用 |
|---|---|
| `opencode` / `opencode tui` | 啟動 TUI。旗標：`--continue/-c`、`--session/-s`、`--fork`、`--prompt`、`--model/-m`、`--agent`、`--port`、`--hostname`、`--mdns`、`--cors`。 |
| `opencode run` | **非互動／無頭** 的 prompt 執行——腳本與 CI 的進入點。旗標含 `--continue/-c`、`--session/-s`、`--fork`、`--share`、`--model/-m`、`--agent`、`--file/-f`、`--format`、`--dangerously-skip-permissions`、`--replay`。 |
| `opencode serve` | 無頭 HTTP/SSE server，**無 UI**。`--port`、`--hostname`、`--mdns`、`--cors`。 |
| `opencode web` | 無頭 server **附** web 介面。旗標同 `serve`。 |
| `opencode attach` | 將 TUI 連上已在執行的後端。`--dir`、`--continue/-c`、`--session/-s`、`--fork`、`--password/-p`、`--username/-u`。 |
| `opencode auth login` | 認證 provider。`--provider/-p`、`--method/-m`。另有 `auth list/ls`、`auth logout`。 |
| `opencode models [provider]` | 列出可用模型。`--refresh`、`--verbose`。 |
| `opencode agent create` / `agent list` | 建立鷹架與列出 agent。 |
| `opencode mcp ...` | `add`、`list/ls`、`auth [name]`、`logout [name]`、`debug <name>`。 |
| `opencode session list` / `session delete <id>` | 管理 session。`--max-count/-n`、`--format`。 |
| `opencode export [id]` / `import <file>` | 匯出 session（`--sanitize`）；匯入 JSON 或分享 URL。 |
| `opencode stats` | 使用量統計。`--days`、`--tools`、`--models`、`--project`。 |
| `opencode github install` / `github run` / `opencode pr <number>` | GitHub Actions 整合；把 PR 拉進 session。 |
| `opencode plugin <module>` / `opencode generate` / `opencode db [query]` | plugin 安裝、OpenAPI 產生、原始 session DB 存取。 |

值得知道的全域旗標：`--help/-h`、`--version/-v`、`--print-logs`、`--log-level`、`--pure`（忽略所有設定／環境探索）。

經驗法則：**互動工作就待在 TUI；只要你想把一個 prompt 放進腳本、git hook 或 CI，就改用 `opencode run`；當引擎應比任何單一前端更長命時，就用 `serve`／`attach`。**

---

## 3. TUI 內的核心工作流程

### Leader 鍵

OpenCode 的 TUI 以 leader 鍵驅動。**預設 leader 為 `ctrl+x`**（逾時可設定，預設 2000 ms），設定於 `tui.json` 的 `keybinds` 下。值得記住的綁定：

| 按鍵 | 動作 |
|---|---|
| `ctrl+x l` | 列出／恢復 session |
| `ctrl+x n` | 新 session |
| `ctrl+x m` | 切換模型（`/models`） |
| `ctrl+x t` | 切換主題 |
| `ctrl+x e` | 開啟 `$EDITOR` 編寫 prompt |
| `ctrl+x x` | 將對話匯出為 Markdown |
| `ctrl+x c` | 壓縮／摘要 session |
| `ctrl+x u` / `ctrl+x r` | 復原／重做 |
| `ctrl+x q` | 離開 |
| `Tab` | **循環切換主要 agent**（Build ↔ Plan）——`switch_agent` 綁定 |
| `ctrl+p` | 指令選盤（`command_list`） |
| `ctrl+t` | 切換 reasoning／thinking 顯示 |

### 送出脈絡

- **`@`**——模糊檔案搜尋；將檔案內容插入為脈絡。
- **`` !`command` ``**——執行 shell 指令並把其輸出注入 prompt。與 Claude Code 的動態注入是同一套模式。
- **影像拖放**——支援；設定於 `attachment.image` 下。
- **`/editor`**（或 `ctrl+x e`）——讓你進入 `$EDITOR` 編寫長 prompt。用 `export EDITOR="code --wait"` 設定即可在 VS Code 中編寫。

### 權限模式：Plan vs Build

OpenCode 的「模式」如今就是你以 `Tab` 循環切換的**主要 agent**：

| Agent | 行為 | 使用時機 |
|---|---|---|
| **Build** | 預設。所有工具可用；edit 與 bash 依你的權限設定執行 | 實作 |
| **Plan** | 受限——`edit` 與 `bash` 預設為 `ask` | 探索、設計、任何在動手寫入前的跨檔工作 |

這乾淨地對應了「先計畫」的習慣：在 **Plan** 中探索，一旦方法定案就切到 **Build**。權限的可設定粒度遠比這兩模式摘要所示細緻——見 §7。

### Session 與分享

Session 是一等公民且存放在 server 端。`/sessions`（或 `ctrl+x l`）列出、切換、恢復。近期版本起，你可以**在 workspace／目錄之間移動 session**，並透過 API／SDK 附加**自訂中繼資料**。`opencode session list` 與 `session delete` 從 CLI 管理；`opencode export`／`import` 在機器間搬移（或從分享 URL 匯入）。

分享即時 session：

```
/share        # 在剪貼簿建立公開 URL（opncd.ai/s/<id>）
/unshare      # 撤銷存取並刪除已同步的對話
```

`share` 設定鍵控制政策：`"manual"`（預設——明確 `/share`）、`"auto"`（每個 session 都分享）或 `"disabled"`。企業可自架分享基礎設施或以 SSO 把關。

### 復原／重做作為安全網

`ctrl+x u`／`ctrl+x r`（以及 `/undo`／`/redo`）把工作樹與對話往前後捲動。這是「試個有風險的做法、不成就復原」的便利機制——不是 git 的替代品，但很廉價。

---

## 4. 指令：內建與自訂

### 內建斜線指令

`/init`、`/undo`、`/redo`、`/share`、`/unshare`、`/help`、`/connect`、`/models`、`/sessions`、`/editor`、`/export`、`/compact`。同名的自訂指令會覆蓋內建指令。

> **`/connect` vs `auth login`。** 兩者都出現在目前文件中作為 provider 設定——TUI 內用 `/connect`，CLI 用 `opencode auth login`。它們抵達同一處；用手邊現成的那個即可。

### 自訂指令

自訂指令是一個 Markdown 檔。優先序：專案勝過全域。

| 範圍 | 路徑 |
|---|---|
| 全域 | `~/.config/opencode/commands/<name>.md` |
| 專案 | `.opencode/commands/<name>.md` |
| 行內 | `opencode.json(c)` 的 `"command"` 鍵 |

> **單複數命名。** 複數（`commands/`、`agents/`）是文件記載的標準形式；單數（`command/`、`agent/`）仍會載入以向後相容。注意 `opencode agent create` 據報會寫入**單數**的 `.opencode/agent/`（issue #14410），而文件寫的是複數——所以磁碟上同時看到兩者不必意外。

Frontmatter 鍵：`description`、`agent`、`model`、`subtask`（強制指令在 subagent 中執行）、`template`（prompt；Markdown 內文本身也作為 template）。

替換語法，與你已熟悉的模式相同：

| 代符 | 意義 |
|---|---|
| `$ARGUMENTS` | 指令名稱後傳入的全部內容 |
| `$1`、`$2`、`$3` | 位置引數 |
| `` !`command` `` | 注入 shell 指令的輸出 |
| `@file` | 注入檔案內容 |

範例——一個 `.opencode/commands/test.md`：

```markdown
---
description: 為某個 package 跑測試套件並分流失敗
agent: build
---

為以下對象跑測試：$ARGUMENTS

- 目前分支：!`git branch --show-current`
- 測試設定：@vitest.config.ts

跑套件，對任何失敗回報確切指令、斷言與最小修正。勿更動無關程式碼。
```

以 `/test packages/api` 叫用。

---

## 5. Agent 與 subagent：限定行為的單位

OpenCode 把 agent 分成兩種角色：

- **主要 agent（primary）**——你直接與之對話，並以 `Tab` 循環切換。內建為 **Build**（所有工具）與 **Plan**（偏讀取，edit／bash 受限為 `ask`）。
- **Subagent**——由主要 agent 或 `@mention` 叫用。內建：**General**（完整工具）、**Explore**（唯讀程式碼調查）、**Scout**（唯讀外部／相依研究）。

這正是「無限探索燒光我的脈絡」的解方：把讀取繁重的調查委派給 **Explore** 或 **Scout**，它們在各自的脈絡中執行並回報摘要，讓主要對話保持乾淨。

### 定義一個 agent

兩種方式。`opencode.json` 中的 `"agent"` 物件，或一個以檔名為 agent 名稱的 Markdown 檔（`review.md` → `review` agent）：

| 範圍 | 路徑 |
|---|---|
| 全域 | `~/.config/opencode/agents/<name>.md` |
| 專案 | `.opencode/agents/<name>.md` |

```markdown
---
description: 唯讀審查者，獵捕競態條件與被吞掉的錯誤
mode: subagent
model: anthropic/claude-sonnet-4-5
permission:
  edit: deny
  bash: deny
temperature: 0.2
---

你審查 diff。只用 Read 與 Grep。回報：
1. 並行危害
2. 被 log-and-ignore 或默默吞掉的錯誤
3. 你會羞於上線而缺漏的測試

絕不修改檔案。
```

Agent 設定鍵：`description`（必填）、`mode`（`primary` | `subagent` | `all`）、`model`、`prompt`（路徑）、`temperature`、`top_p`、`steps`（最大迭代數）、`permission`、`disable`、`hidden`、`color`。Agent 層級權限與全域**合併，衝突時 agent 的規則勝出**。

`opencode agent create` 執行互動式鷹架（`--path`、`--description`、`--mode`、`--permissions`、`--model/-m`）：範圍 → 描述 → 產生 prompt／id → 權限 → 寫出 Markdown 檔。

---

## 6. AGENTS.md、指令與記憶

OpenCode 的持久指令方案建立在 **`AGENTS.md`** 開放慣例之上，而非專有檔案，並以 Claude Code 備援讓既有 repo 無需更動即可運作。

### 指令檔的載入位置（由廣至窄）

| 範圍 | 路徑 | 用途 |
|---|---|---|
| 專案（巢狀） | 專案根目錄**與**子目錄中的 `AGENTS.md` | 團隊共享、已提交的慣例 |
| 全域 | `~/.config/opencode/AGENTS.md` | 跨所有專案的個人偏好 |
| Claude 備援（專案） | `CLAUDE.md` | 原封不動沿用既有 Claude Code repo |
| Claude 備援（全域） | `~/.claude/CLAUDE.md` | 沿用你的個人 Claude 記憶 |

解析順序：本機檔案**自開啟目錄往上走**（先 `AGENTS.md` 再 `CLAUDE.md`）→ 全域 `AGENTS.md` → `~/.claude/CLAUDE.md`。`/init` 會產生或更新專案的 `AGENTS.md`。

除了這些知名檔名，`instructions` 設定鍵也接受明確路徑、glob，甚至遠端 URL（5 秒抓取逾時）：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["docs/guidelines.md", "packages/*/AGENTS.md"]
}
```

適用於 `CLAUDE.md` 的寫作準則在此原封不動適用：要具體、保持精簡、納入 agent 無法臆測的內容（build／test 指令、專案特定架構、陷阱），排除它能從程式碼讀出的部分。當某條規則一再被忽略，多半是檔案太長了。

---

## 7. 設定與權限

### 設定檔

- 專案：`opencode.json`（或 `.jsonc`）。
- 全域：`~/.config/opencode/opencode.json`。
- TUI 綁定／主題：`tui.json`（專案或全域）。
- Schema：`"$schema": "https://opencode.ai/config.json"`——納入以取得自動補全與驗證。
- 環境覆寫：`OPENCODE_CONFIG`（明確路徑）、`OPENCODE_CONFIG_DIR`、`OPENCODE_CONFIG_CONTENT`（行內 JSON）。

優先序（低 → 高）：遠端 `.well-known/opencode` → 全域 → `OPENCODE_CONFIG` → 專案 → `.opencode` 目錄 → 行內 `OPENCODE_CONFIG_CONTENT` → 受管／企業設定。值支援 `{env:VAR}` 與 `{file:path}` 替換。

你實際會設定的核心鍵：`model`、`small_model`（廉價的雜務模型）、`provider`、`default_agent`、`instructions`、`tools`、`permission`、`mcp`、`plugin`、`formatter`、`lsp`、`share`、`autoupdate`、`compaction`、`server`。

### 權限模型

遠比「寫入前先問」細緻。每個工具類別取 `"allow"`（自動）、`"ask"`（提示）或 `"deny"`（封鎖）之一：

`read`、`edit`（涵蓋 edit/write/patch）、`bash`、`glob`、`grep`、`task`（啟動 subagent）、`skill`、`lsp`、`question`、`webfetch`、`websearch`、`external_directory`、`doom_loop`。

預設值：多數為 `"allow"`；**`doom_loop` 與 `external_directory` 預設為 `"ask"`**；`.env` 寫入預設遭拒，而 `.env.example` 允許。比對使用 `*`／`?` glob，採**最後相符規則勝出**語意——而 bash 規則需要明確萬用字元（`"git *"` 允許帶引數；裸 `"git"` 會封鎖任何帶引數的呼叫）。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "edit": "allow",
    "bash": { "git *": "allow", "rm *": "ask", "*": "ask" },
    "external_directory": "ask"
  }
}
```

### Provider 與切換模型

`opencode auth login`（或 `/connect`）把憑證存於 `~/.local/share/opencode/auth.json`。以 `/models`（`ctrl+x m`）、CLI 的 `--model/-m provider/model`，或 `"model"` 設定鍵（如 `"anthropic/claude-sonnet-4-5"`）切換模型；`"small_model"` 設定雜務模型。自訂或本機 provider 放在 `"provider"` 下，搭配 `options.baseURL`、`models`、`apiKey`（`{env:VAR}`）與 `headers`。

> **Claude 訂閱制認證——變動中的目標。** 約 2026 年 1 月，Anthropic 加上伺服器端保護以限制第三方對 Claude 的 OAuth 存取，而 OpenCode 據報在 2026 年 2 月移除其 Claude OAuth 程式碼。Claude Pro/Max 登入如今大多存在於**社群 plugin**（如 `opencode-anthropic-auth`、`opencode-claude-code-auth`），並有 ToS／封號風險的回報。若你打算以 Claude 訂閱（而非 API key）驅動 OpenCode，**請先確認當前狀態**——這是本報告中最易過時的單一事實。Anthropic API key 認證與所有非 Anthropic provider 不受影響。

---

## 8. MCP、plugin 與 skill

### MCP server

設定於頂層 `"mcp"` 鍵下，以 server 名稱為鍵：

```json
{
  "mcp": {
    "my-local": {
      "type": "local",
      "command": ["bunx", "some-mcp-server"],
      "environment": { "TOKEN": "{env:TOKEN}" },
      "enabled": true
    },
    "my-remote": {
      "type": "remote",
      "url": "https://example.com/mcp",
      "headers": { "Authorization": "Bearer {env:TOKEN}" },
      "enabled": true
    }
  }
}
```

也可透過 `tools` glob 批次停用 server（`"my-mcp*": false`）。

> **文件不一致。** CLI 參考列有 `opencode mcp add`，而 MCP 內文說設定是檔案式、無 add 指令。CLI 指令是較新的加入；指令與手動編輯設定兩者皆可行。

### Plugin 與 hook

Plugin 是讓 OpenCode 在「指令僅為建議」處變得具決定性的方式。它們存放於 `.opencode/plugins/`（專案）、`~/.config/opencode/plugins/`（全域），或作為列於 `"plugin": [...]` 的 npm 套件。Plugin 是一個接收 `{ project, client, $, directory, worktree }`（`client` 是 SDK、`$` 是 Bun 的 shell）並回傳 hook 處理器的函式：

```js
export const Formatter = async ({ $ }) => ({
  "file.edited": async ({ path }) => {
    if (path.endsWith(".go")) await $`gofumpt -w ${path}`
  },
})
```

Hook 事件涵蓋整個生命週期：`tool.execute.before`／`tool.execute.after`、`file.edited`、`file.watcher.updated`、`permission.asked`／`permission.replied`、`session.created`／`compacted`／`idle`／`error`、`lsp.client.diagnostics`、`command.executed`、`tui.toast.show` 等。Plugin 也可定義**自訂工具**（Zod schema + `execute`）並以名稱覆蓋內建工具。把 `package.json` 放進 `.opencode/`，OpenCode 會替你跑 `bun install`。

### Formatter 與 LSP——自動整合

`"formatter": true` 與 `"lsp": true`（或物件設定）會接上程式碼格式化與語言伺服器；**LSP 診斷會回饋進 agent 迴圈**，使模型看到自己引入的型別錯誤。`watcher.ignore` 控制檔案監看器追蹤哪些路徑。

### Skill

OpenCode 實作了源自 Anthropic 的 **Agent Skills** 開放標準（skill 探索 + 檔案式載入於 v1.16.0 落地）。一個 skill 是含 `SKILL.md` 的目錄。搜尋位置包含 `.opencode/skills/<name>/SKILL.md` 與 `~/.config/opencode/skills/<name>/SKILL.md`，加上 Claude 相容（`.claude/skills/...`、`~/.claude/skills/...`）與 `.agents/skills/...` 路徑——因此 Claude Code 的 skill 可原封不動沿用。

Frontmatter：`name`（必填，`^[a-z0-9]+(-[a-z0-9]+)*$`，須與目錄相符）、`description`（必填），以及選填的 `license`、`compatibility`、`metadata`。Agent 透過 `skill` 工具叫用 skill（`skill({ name: "git-release" })`），受 `skill` 權限管轄（可用 `internal-*` 之類萬用字元樣式）。

> 部分第三方指南引用 `~/.opencode/skills/` 作為個人 skill；官方路徑是 `~/.config/opencode/skills/`。請以官方為準。

---

## 9. agentic 實戰演練：打造一個小專案

具體例子：一支 Go CLI，抓取 RSS feed、對 SQLite 快取做去重、把摘要推到 Slack。重點不在專案本身——而在 OpenCode 的元件如何組合。換成任何小專案，編排都成立。

### 步驟 0——奠定基礎（一次）

在乾淨的 repo 裡：

```bash
opencode
```

接著 `/init` 掃描並產生 `AGENTS.md`。檢視並修剪——刪掉只是複述語言的內容（「Go 用 gofmt」；模型知道）。提交。

加上一份讓安全路徑成為預設、並接上格式化的專案設定：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "formatter": true,
  "lsp": true,
  "permission": {
    "edit": "allow",
    "bash": { "go *": "allow", "golangci-lint *": "allow", "rm *": "ask", "*": "ask" }
  }
}
```

加上一個你會重複使用的 `run-checks` 指令——`.opencode/commands/run-checks.md`：

```markdown
---
description: 跑完整的本機驗證流程
agent: build
---

依序執行，在第一個失敗處停下並回報確切指令 + 輸出：

1. `gofumpt -l -w .` —— 必須產生零 diff
2. `go vet ./...`
3. `golangci-lint run`
4. `go test -race -count=1 -coverprofile=coverage.out ./...`

接著回報任何覆蓋率低於 70% 的 package。除非被要求，否則不修任何東西。
```

以及一個唯讀審查 subagent（`.opencode/agents/review.md`），如 §5 所示。

### 步驟 1——探索（Plan agent）

`Tab` 切到 **Plan** agent（edit 與 bash 受限為 `ask`），接著：

```
我想加一個 RSS 抓取器，搭配 SQLite 撐的去重快取。讀過 repo 後告訴我：
1. 已有哪些可重用
2. 新程式碼該放哪以符合慣例
3. go.mod 中哪些相依已解決部分需求
```

若程式庫大到光讀就會燒光脈絡，委派給 **Explore** subagent——它在自己的脈絡執行並回報摘要，讓你的主線保持乾淨。

### 步驟 2——計畫

仍在 Plan 中：

```
為以下提出實作計畫：
- internal/feeds:  帶重試 + 逾時的 RSS 抓取器
- internal/cache:  以 GUID + 內容雜湊為鍵的 SQLite 去重快取
- cmd/scrape:      CLI 進入點

每個 package：檔案、匯出 API、測試案例。標出任何需要我拍板的決定。
```

在對話中鎖定 API 介面，然後 `Tab` 切回 **Build**。

### 步驟 3——實作

```
實作計畫。契約清楚處先寫測試。每完成一個 package，跑 /run-checks 並修正它揭露的問題。
```

執行期間：盯著 diff 檢視器（下一個／上一個 hunk 導覽讓這很快），駁回不對的並說明原因，並用 `ctrl+x u` 回捲走偏的變更。若同一個錯誤你已修正兩次，與其和受污染的脈絡纏鬥，不如以更緊的 prompt 開新 session（`ctrl+x n`）。

### 步驟 4——審查（全新脈絡、不同視角）

開新 session（`ctrl+x n`），交給唯讀審查者：

```
@review 這個分支上的 diff。
```

一個沒有實作偏見的全新脈絡，會抓到實作者替自己合理化掉的東西。

### 步驟 5——出貨

```
以清楚訊息提交並開 PR。描述中：一行摘要、使用者得到的新指令、以及首次執行會建立一個 SQLite 檔的提醒。
```

CI 上，同樣的工作可無頭執行：

```bash
opencode run --agent review "對照 origin/main 審查 diff，找競態條件與被吞掉的錯誤"
```

### 何時擴大規模

- **持續驗證** → 把「必須永遠發生」的規則從 `AGENTS.md` 移進 **plugin** hook（`file.edited` 做格式化、`session.idle` 跑測試）。Hook 具決定性；指令僅為建議。
- **跨專案重用的工作流程** → 把專案的 command／agent／skill 提升到 `~/.config/opencode/`，或打包成 plugin／發布成 skill。
- **引擎應比前端更長命** → 在遠端機器上 `opencode serve`、從筆電 `opencode attach`；或以 `/share` 分享即時 session。

---

## 10. 應避免的失敗模式

每個 agentic 工作流程都會踩的那幾個，以 OpenCode 的語彙呈現：

- **大雜燴 session。** 一個任務漂移成岔題，脈絡塞滿雜訊。*修法：*無關任務之間用 `ctrl+x n` 開新 session；某任務確實跑很長時用 `ctrl+x c` 壓縮。
- **修正迴圈。** 兩次修正失敗代表脈絡已被死路做法污染。*修法：*開新 session，帶著學到的東西重寫 prompt。
- **塞爆的 `AGENTS.md`。** 長到重要規則被淹沒。*修法：*修剪，並把「必須永遠」的規則改成 plugin hook。
- **信任而不驗證。** 看似合理卻漏掉邊界情況的程式碼。*修法：*永遠給 agent 具體可比對的東西——`/run-checks`、一個測試、一個腳本。善用 LSP 回饋迴圈。
- **無限探索。** 「調查 X」卻無範圍，讀了上百個檔案。*修法：*委派給 **Explore**／**Scout** subagent（獨立脈絡），或把 prompt 範圍限定到某路徑。
- **權限小割傷。** 安全指令上不斷跳 `ask`。*修法：*用明確萬用字元擴大 `permission.bash` 允許清單（`"go *": "allow"`），而非動用 `--dangerously-skip-permissions`。

---

## 11. 實用參考

- 文件首頁：https://opencode.ai/docs/
- CLI 參考：https://opencode.ai/docs/cli/
- 設定：https://opencode.ai/docs/config/
- Agents：https://opencode.ai/docs/agents/
- Commands：https://opencode.ai/docs/commands/
- TUI／快捷鍵：https://opencode.ai/docs/tui/
- 權限：https://opencode.ai/docs/permissions/
- Providers：https://opencode.ai/docs/providers/
- MCP server：https://opencode.ai/docs/mcp-servers/
- Plugin：https://opencode.ai/docs/plugins/
- Skill：https://opencode.ai/docs/skills/
- 規則／`AGENTS.md`：https://opencode.ai/docs/rules/
- 分享：https://opencode.ai/docs/share/
- SDK：https://opencode.ai/docs/sdk/
- Repo 與 releases：https://github.com/sst/opencode（releases 在 `/releases`）
- 模型登錄：https://models.dev
