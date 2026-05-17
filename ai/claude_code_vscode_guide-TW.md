# 在 VS Code 中使用 Claude Code — 實戰指南

> 範疇：如何從 VS Code 內部驅動 Claude Code、值得記下來的指令、skill 與 `CLAUDE.md` 的結構，以及這一切如何組合成一個適用於小型專案的 agentic 工作流程。對象是已熟悉 Claude Code 終端機模式、希望理解 VS Code 原生用法的開發者。

---

## 1. 兩種介面（以及為何要分清楚）

裝上 **Claude Code 的 VS Code 擴充功能**之後，你會得到兩個互通的介面。它們共用同一份 auth、設定、對話歷史、MCP 伺服器與 `~/.claude/` 設定，但暴露的功能子集不同。

| | **擴充功能（Spark 面板）** | **整合式終端機裡的 CLI** |
|---|---|---|
| 啟動 | 編輯器右上角的 Spark 圖示、活動列、狀態列，或 `Cmd/Ctrl+Shift+P` → "Claude Code: Open in New Tab" | `` Cmd/Ctrl+` `` 然後 `claude` |
| 差異檢視 | 原生並排 diff，可接受／拒絕 | 相同 — CLI 透過內建的 `ide` MCP 伺服器使用 VS Code 的 diff viewer |
| Slash 指令 | 子集；輸入 `/` 看可用清單 | 完整集合 |
| MCP 伺服器設定 | 用 CLI 加伺服器；以 `/mcp` 管理現有 | 完整 CRUD |
| `!` bash 捷徑、Tab 補完 | 無 | 有 |
| Checkpoint、Plugin | 有 | 有 |

經驗法則：**用擴充功能做 inline diff、多分頁對話與 @-mention；面板沒暴露的指令、參數或 MCP 操作就切到終端機。** 兩邊對話彼此互通 — 在整合式終端機跑 `claude --resume` 會列出包括面板對話在內的歷史選單。

### 前置條件

- VS Code 1.98.0 或以上
- Anthropic 帳號（若透過 Bedrock／Vertex／Foundry 路由，則需該供應商憑證）
- 安裝方式：`Cmd/Ctrl+Shift+X` → 搜尋 "Claude Code" → Install，或 marketplace URI `vscode:extension/anthropic.claude-code`

### Claude 在視窗中的位置

面板可拖到：

- **次要側邊欄**（右側）：寫程式時 Claude 保持可見 — 預設首選
- **主要側邊欄**（左側）：希望與 Explorer／Search 群組在一起
- **編輯器區域**：以分頁開啟；適合做側邊對話

按 `Cmd/Ctrl+Shift+Esc` 在新編輯器分頁裡開啟新對話。`Cmd/Ctrl+Esc` 在編輯器與 Claude 提示框間切換焦點。

---

## 2. 擴充功能內的核心工作流程

### 傳入 context

- **選取的文字會自動帶入。** 反白程式碼，Claude 就看得到。提示框底部會顯示行數。
- **`Option/Alt+K`** 把選取轉成明確的 `@file.ts#5-10` 參照。
- **`@filename`** 以模糊比對附上檔案或資料夾（資料夾請加結尾斜線）：`@src/auth/` 或 `@auth`（會比對到 `auth.js`、`AuthService.ts` 等）。
- **`@terminal:name`** 把終端機分頁的輸出灌進提示 — 適合不想複製貼上的 stack trace 或指令輸出。
- **`@browser`**（需安裝 Claude in Chrome 擴充功能）操控一個瀏覽器分頁，可做 E2E 流程與 console-log debug。
- **Shift 拖曳檔案**到提示框可附加檔案，或直接貼上圖片。

### 權限模式（油門）

點提示框底部的模式指示器，或在設定裡指定 `claudeCode.initialPermissionMode`：

| 模式 | 行為 | 何時用 |
|---|---|---|
| `default` | 每次寫檔或 Bash 都會詢問 | 預設；最安全 |
| `plan` | Claude 只讀檔並提出計劃，不會改動任何檔案 | 探索、複雜重構、任何跨多檔的工作 |
| `acceptEdits` | 自動接受檔案編輯，Bash 仍會詢問 | 已知範圍下的順暢推進 |
| `auto` | 由分類模型自動核可例行動作，攔下風險動作（範疇擴張、未知基礎設施、被惡意內容驅動的動作） | 長時間無人值守；需 Team／Enterprise／API 方案與 Sonnet/Opus 4.6 |
| `bypassPermissions` | 完全不詢問 | **僅在無網際網路的沙箱裡使用** |

Plan 模式是 agentic 工作流程的關鍵 — VS Code 會把計劃渲染成一份可直接內聯編輯的 markdown 文件。按 `Ctrl+G` 把計劃以編輯器開啟並加上註解。

### 多重對話

開多個分頁／視窗以平行進行（一個做研究、一個實作、一個審查）。每個維護獨立歷史。Spark 圖示上的小點代表狀態：藍色 = 等你決定權限，橘色 = 隱藏分頁有回覆已就緒。

要在同一個 repo 上真正平行工作，請用 git worktree：

```bash
claude --worktree feature-auth
```

每個 worktree 有自己的檔案與分支；各個實例彼此不互相干擾。

### Checkpoint（回溯）

Claude 每個動作都會自動建立 checkpoint。把游標停在面板裡的任一訊息上會出現回溯按鈕。三種選項：

- **Fork conversation from here** — 從這裡分叉對話，程式碼保持原狀
- **Rewind code to here** — 還原檔案，對話保留
- **Fork and rewind** — 兩者皆做

這不是 git 的替代品（外部變更與多數 Bash 副作用都沒追蹤），但讓「試個有風險的做法，不行就退回」變得很便宜。

---

## 3. 真正值得記住的指令

在提示框輸入 `/` 看即時選單。下面是日常會用到的，依用途分組。`<arg>` 為必填、`[arg]` 選填。

### 專案啟動與記憶

| 指令 | 作用 |
|---|---|
| `/init` | 掃描 repo 並產生入門級 `CLAUDE.md`，含偵測到的 build、test、慣例資訊。設 `CLAUDE_CODE_NEW_INIT=1` 啟用互動式流程，會額外提議 skill、hook 與個人記憶檔。 |
| `/memory` | 列出本次 session 載入的所有記憶檔（`CLAUDE.md`、`CLAUDE.local.md`、所有 `.claude/rules/*.md`）、切換 auto memory、開啟 auto-memory 資料夾。點任何檔案即可編輯。 |
| `/skills` | 列出可用 skill（bundled、user、project、plugin）。 |
| `/agents` | 管理子代理（subagent）設定。 |
| `/hooks` | 檢視已設定、對應工具事件的 hook。 |
| `/permissions` | 開啟 allow／ask／deny 規則編輯器；檢視 auto 模式近期的拒絕事件。別名：`/allowed-tools`。 |
| `/mcp` | 管理 MCP 伺服器連線與 OAuth。 |
| `/plugin` | 管理 Claude Code 套件（skill／hook／agent／MCP 組合包）。 |
| `/config` | 開啟設定 UI（主題、模型、輸出風格）。別名：`/settings`。 |

### Session 進行中

| 指令 | 作用 |
|---|---|
| `/plan [description]` | 直接進入 plan 模式，可一併帶上任務。範例：`/plan refactor auth to use OAuth2`。 |
| `/clear` | 重置對話 context。別名：`/reset`、`/new`。在不相關任務間使用 — context 太肥會降低表現。 |
| `/compact [focus]` | 摘要歷史以釋出 context，可加上焦點指示。 |
| `/context` | 用網格視覺化目前 context 用量；標出耗用 context 的工具與肥大的記憶。 |
| `/rewind` | 開啟回溯選單（也可按 `Esc Esc`）。可只回溯對話、只回溯程式碼、兩者皆回溯，或從選定訊息開始做摘要。別名：`/checkpoint`。 |
| `/branch [name]` | 在目前訊息處分叉對話。別名：`/fork`。 |
| `/btw <question>` | 在不污染對話歷史的前提下問個 side question — 答案會出現在可關閉的浮動框。 |
| `/diff` | 開啟互動 diff viewer；瀏覽尚未提交的變更與每一輪的 diff。 |
| `/copy [N]` | 複製最近一則（或第 N 則）助理回覆；含程式碼區塊時會跳出 picker。 |

### 模型與思考強度

| 指令 | 作用 |
|---|---|
| `/model [name]` | 切換模型；左右方向鍵調整 effort 等級。立即生效。 |
| `/effort low\|medium\|high\|max\|auto` | 設定推理強度。`low/medium/high` 跨 session 持續；`max` 為單次、且需 Opus 4.6。 |
| `/fast on\|off` | 切換 fast 模式。 |

### 調查與審查

| 指令 | 作用 |
|---|---|
| `/security-review` | 掃描分支待提交變更，找 injection、認證、資料外洩問題。 |
| `/pr-comments [PR]` | 透過 `gh` CLI 抓 PR comment（會自動偵測目前分支）。 |
| `/insights` | 對你的 Claude Code session 模式與摩擦點產出報告。 |

### Bundled skill（與 slash 指令一同出現）

這些是 Claude Code 預載的 prompt-based playbook。與內建指令不同，它們以 Claude 的工具編排動作，而非執行固定邏輯。

| Skill | 用途 |
|---|---|
| `/batch <instruction>` | 把一個大變更拆成 5–30 個單元，每個單元在自己的 worktree 裡跑一個背景 agent，各自開 PR。需在 git repo 中。範例：`/batch migrate src/ from Mocha to Vitest`。 |
| `/simplify [focus]` | 對近期變更檔案開三個平行 review agent，彙整發現後套用修正。可指定焦點：`/simplify focus on memory efficiency`。 |
| `/debug [description]` | 開啟 debug 記錄並分析該 session 的 debug log。 |
| `/loop [interval] <prompt>` | 在 session 開著時定期跑某個 prompt。範例：`/loop 5m check if the deploy finished`。 |
| `/claude-api` | 載入 Claude API 與 Agent SDK 對應你專案語言的參考。 |

其餘 — `/help` 列出完整選單；`/doctor` 診斷安裝問題。

---

## 4. Skill：可重複使用的 Claude 行為單元

一個 skill 是一個含 `SKILL.md` 的資料夾。Claude 在啟動時把 description 放進 context，並在你用 `/skill-name` 觸發、或它判斷 description 切合你目前要做的事時，再把完整 body 拉進來。

### Skill 的存放位置（覆蓋優先序：上者勝出）

| 範圍 | 路徑 | 適用 |
|---|---|---|
| 企業（受管理） | 由 managed settings 指定 | 組織內所有使用者 |
| 個人 | `~/.claude/skills/<name>/SKILL.md` | 你的所有專案 |
| 專案 | `.claude/skills/<name>/SKILL.md` | 僅此專案 |
| 套件 | `<plugin>/skills/<name>/SKILL.md` | 套件啟用之處 |

Plugin skill 使用 `plugin-name:skill-name` 命名空間，不會與其他層級衝突。

> **指令 vs skill 註記.** 自訂 slash 指令已併入 skill。`.claude/commands/deploy.md` 與 `.claude/skills/deploy/SKILL.md` 都會建出 `/deploy` 且行為一致。建議使用 skill，因為它支援附屬檔案、frontmatter 控制與自動觸發。

### Skill 的結構

```text
my-skill/
├── SKILL.md          # required entrypoint
├── reference.md      # optional — Claude loads on demand
├── examples.md       # optional — loaded on demand
└── scripts/
    └── helper.py     # optional — executable, not loaded into context
```

`SKILL.md` 控制在 ~500 行以內。把大量參考資料移到獨立檔案，在 `SKILL.md` 連結到它們，Claude 才會按需載入。

### Frontmatter 參考

```yaml
---
name: fix-issue                      # becomes /fix-issue; lowercase, hyphens, digits, max 64 chars
description: Fix a GitHub issue end-to-end   # how Claude decides when to invoke
argument-hint: [issue-number]        # shown in autocomplete
disable-model-invocation: true       # only you can trigger it (no auto-invoke)
user-invocable: false                # hide from / menu (background knowledge only)
allowed-tools: Read, Grep, Bash(gh *) # tools auto-approved while skill is active
model: opus                          # override session model
effort: high                         # override session effort level
context: fork                        # run in a forked subagent
agent: Explore                       # which subagent type to use with context: fork
paths:                               # only auto-load when working with matching files
  - "src/api/**/*.ts"
shell: bash                          # bash (default) or powershell
---
```

所有欄位皆選填。最值得寫的是 `description`。

### 字串替換

| Token | 意義 |
|---|---|
| `$ARGUMENTS` | skill 名稱之後的所有內容 |
| `$ARGUMENTS[N]` 或 `$N` | 第 N 個位置參數（從 0 起算） |
| `${CLAUDE_SESSION_ID}` | 目前 session ID — 適合做記錄 |
| `${CLAUDE_SKILL_DIR}` | Skill 自身的目錄 — 在腳本裡用它，工作目錄變動時路徑不會壞 |

### 動態 context 注入 — `` !`command` `` 模式

這是最被低估的功能之一。在 skill 內文任何位置放 `` !`shell command` ``。該指令會在 Claude 看到 skill **之前**就執行，輸出取代該佔位符。Claude 看到的是 render 後的 prompt，不是該指令本身。

```yaml
---
name: pr-summary
description: Summarize the current pull request
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## PR context
- Diff: !`gh pr diff`
- Comments: !`gh pr view --comments`
- Files changed: !`gh pr diff --name-only`

## Your task
Summarize the PR for a reviewer. Highlight risk areas and suggest test coverage gaps.
```

### 觸發控制矩陣

| Frontmatter | 你可觸發 | Claude 可自動觸發 | 何時載入 context |
|---|---|---|---|
| （預設） | 可 | 可 | description 一直在 context；body 觸發時載入 |
| `disable-model-invocation: true` | 可 | 不可 | description 不在 context；body 在你觸發時載入 |
| `user-invocable: false` | 不可 | 可 | description 一直在 context；body 於自動觸發時載入 |

任何你想自行掌握時機的副作用動作，請設 `disable-model-invocation: true`：`/deploy`、`/release`、`/send-slack-message`。

---

## 5. CLAUDE.md 與記憶系統

每次 session 都從全新 context window 開始。有兩個互補機制把知識帶過 session：

- **`CLAUDE.md` 檔** — 你寫的；持續指示
- **Auto memory** — Claude 寫的；學到的東西與模式

兩者每次 session 都會載入，被當作 context（而非強制設定）。**越具體、越精簡，Claude 就越穩定遵循。**

### CLAUDE.md 的位置（載入順序：由廣至窄）

| 範圍 | 路徑 | 用途 |
|---|---|---|
| 受管理政策 | OS 限定（macOS 為 `/Library/Application Support/ClaudeCode/CLAUDE.md`） | 全組織標準、合規 |
| 使用者 | `~/.claude/CLAUDE.md` | 跨所有專案的個人偏好 |
| 專案 | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 團隊共享、提交版控 |
| 本地 | `./CLAUDE.local.md` | 個人專案專屬；加入 `.gitignore` |

檔案是串接而非覆蓋，從你的 CWD 一路往檔案系統根目錄走。子目錄裡的 `CLAUDE.md` 在 Claude 讀到該目錄檔案時**按需載入**。

### 寫出有效的指示

是否被遵循，最大的預測因子是**具體性**與**篇幅**。目標：每份檔案 200 行以內。

| ✅ 寫進去 | ❌ 不要寫 |
|---|---|
| Claude 猜不到的 build／test 指令 | 看程式碼就能知道的事 |
| 與預設不同的程式碼風格規則 | 標準語言慣例 |
| Repo 禮節（分支命名、PR 慣例） | 長教學或 API 文件（連結過去就好） |
| 該專案特有的架構決策 | 逐檔的程式碼描述 |
| 環境變數、開發環境怪僻、地雷 | 自明的實踐（「寫乾淨的程式碼」） |

實用模式：當你對 Claude 修正同一件事兩次，就把它寫進 `CLAUDE.md`。若你發現某條規則一直被忽略，多半是檔案太長 — 狠下心修剪。每一行的試金石是「拿掉這行 Claude 會出錯嗎？」如果不會，砍掉。

### 用 `.claude/rules/` 做路徑範疇限制

對大型專案，把指示分散到 `.claude/rules/` 多個檔。沒有 frontmatter 的規則無條件載入（同 `.claude/CLAUDE.md`）。加上 `paths` 欄位，該規則只在 Claude 動到匹配檔時才載入：

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "src/handlers/**/*.ts"
---

# API Development Rules

- Every endpoint validates input with the shared `validate()` helper before any business logic
- Errors return the standard envelope: `{ error: { code, message, details? } }`
- Annotate handlers with `@openapi` JSDoc comments — these feed the spec generator
```

Glob 樣式支援副檔名比對（`**/*.ts`）、目錄範圍（`src/**/*`）與大括號展開（`src/**/*.{ts,tsx}`）。

### Import

`@path/to/file` 語法在載入時把該檔案內容內聯展開。相對路徑相對於含 import 的檔案；絕對路徑與 `~/` 也可用。最大深度 5 層。

```markdown
See @README.md for project overview and @package.json for npm scripts.

# Workflow
- Git: @docs/git-instructions.md
- Personal: @~/.claude/my-overrides.md
```

外部 import 首次被觸及時會跳出一次性同意對話框。

### 與 `AGENTS.md` 互通

Claude Code 讀 `CLAUDE.md`，不讀 `AGENTS.md`。如果你的 repo 為其他 agent 維護 `AGENTS.md`，可用 symlink 或 import：

```markdown
@AGENTS.md

## Claude Code specifics
Use plan mode for changes under `src/billing/`.
```

在已有 `AGENTS.md`（或 `.cursorrules`、`.windsurfrules`）的 repo 跑 `/init`，會把相關內容合併進產出的 `CLAUDE.md`。

### Auto memory

預設下，Claude 會把工作中學到的東西寫到 `~/.claude/projects/<project>/memory/` — build 怪事、debug 心得、你曾表達過的偏好。用 `/memory` 切換或設定 `autoMemoryEnabled`；要全域停用設 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`。`MEMORY.md` 前 200 行（或 25 KB）每次 session 都會載入；topic 檔則按需載入。

Auto memory 為本機儲存，並在同一 repo 的多個 worktree 間共享。

---

## 6. Agentic 走過一遍：建一個小專案

具體例子：一個 Go CLI，會抓 RSS feed、用 SQLite 快取做去重、把摘要推到 Slack 頻道。端到端含測試、lint 與 CI。

這裡重點不在專案本身 — 而在示範各個元件如何組合。換成任何小專案，編排都一樣。

### 步驟 0 — 打地基（一次性，做完就忘）

在新 repo 的整合式終端機裡：

```bash
claude
```

然後：

```
/init
```

讓 Claude 掃 repo 並提出 `CLAUDE.md`。在 diff viewer 裡看一遍；把只是在描述語言本身的部分剃掉（「Go 用 gofmt」 — Claude 知道）。提交。

為資料層加一個專案規則：

```markdown
# .claude/rules/storage.md
---
paths:
  - "internal/storage/**/*.go"
  - "internal/cache/**/*.go"
---

# Storage rules
- All SQLite access goes through `internal/storage.DB`; never open new connections
- Migrations live in `internal/storage/migrations/` and are embedded with `//go:embed`
- Use `context.Context` on every query; no `context.Background()` in package code
```

再加兩個你會反覆用的 skill：

```markdown
# .claude/skills/run-checks/SKILL.md
---
name: run-checks
description: Run the full local verification pipeline — formatter, vet, lint, race tests, and coverage. Use after any non-trivial change.
allowed-tools: Bash(go *), Bash(golangci-lint *), Bash(gofumpt *)
---

Run the verification pipeline:

1. `gofumpt -l -w .` — formatter must produce zero diffs
2. `go vet ./...`
3. `golangci-lint run`
4. `go test -race -count=1 -coverprofile=coverage.out ./...`
5. Report coverage of any package below 70%

If any step fails, stop and report the failure with the exact command and output. Do not attempt fixes unless asked.
```

```markdown
# .claude/skills/new-feature/SKILL.md
---
name: new-feature
description: Implement a small feature from spec to PR using plan-first workflow.
argument-hint: <one-line feature description>
disable-model-invocation: true
---

Implement: $ARGUMENTS

Phase 1 — Explore (read only):
1. Find relevant existing code with Grep/Glob
2. Identify the package(s) that need changes
3. Identify the existing patterns to follow (similar handlers, similar tests)

Phase 2 — Plan:
4. Propose a plan with file-level granularity and a list of test cases
5. Wait for approval before any writes

Phase 3 — Implement:
6. Write tests first if a clear contract exists
7. Implement the feature
8. Invoke /run-checks
9. If checks pass, write a commit message and open a PR with `gh pr create`

Do not skip phases. Surface any ambiguity as a question before planning.
```

選用但很划算：在 `Edit` 上掛一個 `PostToolUse` hook，自動對被改過的檔跑 `gofumpt`。在 `.claude/settings.json` 設定，或請 Claude 幫你寫一個：*"add a hook that runs gofumpt on any Go file Claude edits."*

### 步驟 1 — 探索（Plan 模式，只讀）

把面板切到 **Plan 模式**。在新對話裡：

```
/plan I want to add an RSS feed fetcher with a SQLite-backed dedup cache.
Look through the repo and tell me:
1. What's already here that I can reuse
2. Where new code should live to match existing conventions
3. What external dependencies are already in go.mod that solve any of this
```

Claude 讀檔，但不動任何東西。計劃會以 markdown 文件在編輯器開啟。按 `Ctrl+G` 直接編輯 — 加上限制、移除超範圍項、推回有疑慮的假設。

如果 codebase 大到探索會吃掉 context，請改成委派給子代理：

```
Use a subagent to investigate how we currently fetch external HTTP resources
and what testing patterns we use for things that hit the network. Report back
with file references and the recommended pattern.
```

子代理在自己的 context window 裡跑，回傳摘要。你主對話保持乾淨。

### 步驟 2 — Plan

仍在 plan 模式：

```
Propose an implementation plan for:
- internal/feeds: RSS fetcher with retry + timeout
- internal/cache: SQLite dedup cache keyed by GUID + content hash
- cmd/scrape: CLI entrypoint

For each package, list the files, the exported API, and the test cases.
Highlight any decisions you're making that I should weigh in on.
```

產出一份可審閱的文件。直接編輯 markdown 計劃把 API surface 鎖死。準備好之後，切回 **default** 或 **acceptEdits** 模式。

### 步驟 3 — 實作

```
/new-feature RSS feed fetcher with SQLite dedup cache, plus the cmd/scrape CLI
```

該 skill 會跑完 Explore → Plan → Implement → Verify → Commit。因為它標了 `disable-model-invocation: true`，Claude 不會自己觸發 — 時機你掌握。

執行期間：
- 盯著 diff viewer。看到不對勁就拒絕並說明原因；Claude 會修正方向。
- 看到 Claude 卡在錯誤路徑時按一次 `Esc` 停下，然後重導。如果同一件事修正兩次仍不對，`/clear` 重來，用更精確的 prompt 啟動。
- 變更走偏時用 `Esc Esc`（或 `/rewind`）回到先前 checkpoint。

### 步驟 4 — 審查（全新 context、不同視角）

開新分頁（`Cmd/Ctrl+Shift+Esc`）。全新 context，無實作偏見：

```
Review the diff on this branch for:
1. Race conditions around the cache
2. Error handling — anything swallowed or logged-and-ignored
3. Test gaps — what would I be embarrassed to ship without

Use Read and Grep only. Don't write anything.
```

或直接跑 `/simplify`，它會開三個平行 review agent 並套用修正。

要做安全審查：

```
/security-review
```

### 步驟 5 — 出貨

回到實作分頁：

```
Commit the changes with a clear message and open a PR. In the PR description,
include: a one-line summary, the new commands users get, and a note about
the new SQLite file that gets created at first run.
```

Claude 會用 `gh` 開 PR。如果你的 repo 有設 GitHub Actions，也可以用 Claude Code 的 review action — 但那是另一套設定。

### 何時該加碼

多數小專案一個對話就夠。下列情境再考慮加碼：

- **變更跨多個彼此獨立的檔。** `/batch` 拆成 worktree 平行跑。適合 migration：`/batch migrate all handlers in internal/api to use the new context middleware`。
- **要持續性驗證。** 加 hook：`Edit` 後 `PostToolUse` 跑 formatter、`Stop` 跑測試、`PreToolUse` 擋下對保護路徑的寫入。Hook 是 deterministic，`CLAUDE.md` 只是建議。
- **某個工作流程要跨專案重用。** 把 project skill 升級到個人 `~/.claude/skills/`，或打包成 plugin。

---

## 7. 值得知道的設定

### 擴充功能設定（VS Code 範圍）

以 `Cmd/Ctrl+,` 開啟 → Extensions → Claude Code。

| 設定 | 預設 | 註 |
|---|---|---|
| `claudeCode.selectedModel` | `default` | 對話內可用 `/model` 覆寫 |
| `claudeCode.initialPermissionMode` | `default` | 若希望預設先 plan，設 `plan` |
| `claudeCode.useTerminal` | `false` | 強制在面板用 CLI 風格介面 |
| `claudeCode.preferredLocation` | `panel` | `sidebar` 或 `panel` |
| `claudeCode.autosave` | `true` | Claude 讀寫前先存檔 |
| `claudeCode.respectGitIgnore` | `true` | 檔案搜尋時跳過 `.gitignore` 樣式 |
| `claudeCode.useCtrlEnterToSend` | `false` | 把送出鍵改為 Ctrl/Cmd+Enter |

### Claude Code 設定（與 CLI 共用）

存於 `~/.claude/settings.json` 與 `.claude/settings.json`（專案）／`.claude/settings.local.json`（gitignored、個人）。用於權限、hook、MCP 伺服器、env 變數、plugin 啟用。

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": ["Bash(go test*)", "Bash(go build*)", "Bash(gofumpt*)"],
    "deny": ["Write(.env*)", "Bash(rm -rf*)"]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": { "tool": "Edit", "file": "**/*.go" },
        "command": "gofumpt -w $CLAUDE_FILE_PATHS"
      }
    ]
  }
}
```

加上 `$schema` 那行，VS Code 內就能補完與驗證。

---

## 8. 該避開的失敗模式

下面這些反覆出現；認得出來就解決一半。

- **混雜成一鍋的 session。** 某任務 → 跑去做不相干的事 → 回到原任務。Context 已滿是雜訊。*解法：*不相關任務之間 `/clear`。
- **修正迴圈。** Claude 做錯一次、你修正、還是錯、再修正。Context 充斥失敗嘗試。*解法：*連續兩次修正失敗就 `/clear`，把學到的東西寫進 prompt 重來。
- **`CLAUDE.md` 過度規範。** 檔案長到重要規則被淹沒。*解法：*修剪。把「永遠都要做」的規則改成 hook。
- **信任而不驗證。** 看起來合理但沒處理 edge case 的實作。*解法：*永遠給 Claude 一個具體比對標的 — 測試、腳本、截圖。無法驗證就不要 merge。
- **無止境的探索。** 「調查 X」沒劃範圍。Claude 讀了幾百個檔。*解法：*委派給子代理（獨立 context），或把 prompt 限縮 —「只讀 `src/auth/**` 範圍下的檔來調查 X」。

---

## 9. 參考連結

- VS Code 擴充功能文件：https://code.claude.com/docs/en/vs-code.md
- 內建指令：https://code.claude.com/docs/en/commands.md
- Skill：https://code.claude.com/docs/en/skills.md
- 記憶與 `CLAUDE.md`：https://code.claude.com/docs/en/memory.md
- 最佳實踐：https://code.claude.com/docs/en/best-practices.md
- Hook：https://code.claude.com/docs/en/hooks.md
- 子代理：https://code.claude.com/docs/en/sub-agents.md
- MCP：https://code.claude.com/docs/en/mcp.md
- 文件地圖：https://code.claude.com/docs/en/claude_code_docs_map.md
