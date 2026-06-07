# OpenCode — A Practitioner's Report

> Scope: how to drive **OpenCode** (the open-source terminal AI coding agent by the SST team), the commands and keybindings worth memorizing, how agents, commands, skills, and `AGENTS.md` are structured, and how all of it composes into an agentic workflow for a small project. Aimed at a developer who already knows a coding agent like Claude Code or Aider and wants the OpenCode-native picture. Evaluated against **v1.16.2 (June 2026)**.
>
> A few facts in this space are contested or moving fast — Claude subscription auth, a possible org rename, and singular-vs-plural config directories. Those are flagged inline rather than smoothed over.

---

## 1. The shape of OpenCode (and why it matters)

OpenCode is not a single binary you talk to — it is a **client/server system**. A headless server holds the agent loop, tool execution, sessions, and an HTTP + SSE API; clients attach to it. That one architectural choice is the thing that separates OpenCode from Claude Code and Aider, and it shows up everywhere downstream: you can run the engine on a remote box and attach a local TUI, drive it from an SDK, or hand a teammate a share link to a live session.

| | **What it is** | **Notes** |
|---|---|---|
| Server / engine | TypeScript on **Bun**; the agent loop, tools, MCP, sessions, HTTP+SSE API | Runs standalone via `opencode serve` |
| TUI client | Written in **Go**; the default frontend | What you get when you just type `opencode` |
| Web client | Browser UI served by `opencode web`; also renders shared sessions | Same engine, different frontend |
| IDE extension | VS Code / Cursor / Windsurf / VSCodium | Auto-installs when you run `opencode` in the integrated terminal |
| SDK | `@opencode-ai/sdk` (TypeScript, generated with Stainless) | Type-safe client over the server API |
| ACP | `opencode acp` exposes the **Agent Client Protocol** | Lets ACP-capable editors (e.g. Zed) attach |

How it lands against the obvious alternatives:

- **vs Claude Code** — not tied to one vendor; **MIT-licensed and open source**; the engine is splittable from the frontend; uses `AGENTS.md` as its primary instruction file (with a `CLAUDE.md` fallback). The trade-off is that you assemble your own provider/model story instead of getting one curated stack.
- **vs Aider** — a full agentic tool loop with subagents, MCP, and a real TUI, not a git-commit-centric REPL.
- **vs Cursor** — terminal/CLI-first rather than a forked editor, though it ships an IDE extension for the inline experience.

Provider-agnostic by design: it is built on the Vercel **AI SDK** plus the **models.dev** registry, advertising **75+ providers** (Anthropic, OpenAI, Gemini, DeepSeek, Qwen, Bedrock, plus any OpenAI-compatible endpoint including Ollama, llama.cpp, and LM Studio).

> **Org-name caveat.** The canonical repo is `github.com/sst/opencode` (MIT). Several secondary sources say the maintaining org has been renamed to **"Anomaly Innovations" / `anomalyco`** (the Homebrew tap and Docker image use `anomalyco`). Treat `sst/opencode` as the primary handle; verify the rename before quoting it as fact.

### Prerequisites

- A modern terminal emulator — WezTerm, Alacritty, Ghostty, or Kitty are the recommended set for correct rendering.
- An LLM provider API key or a supported subscription.

### Installation

```bash
curl -fsSL https://opencode.ai/install | bash      # primary installer
npm install -g opencode-ai                          # npm — package name is opencode-ai
brew install anomalyco/tap/opencode                 # Homebrew (tap is under anomalyco)
sudo pacman -S opencode                             # Arch
choco install opencode                              # Windows / Chocolatey
docker run -it --rm ghcr.io/anomalyco/opencode      # Docker
```

`cd` into a project and run `opencode`. On first run, `/init` scans the repo and generates or updates `AGENTS.md`. Maintenance: `opencode upgrade [target]` (`--method/-m`) and `opencode uninstall` (`--keep-config/-c`, `--keep-data/-d`, `--dry-run`, `--force/-f`).

---

## 2. The CLI surface (the part Claude Code users underestimate)

Because the engine is headless, the `opencode` CLI is a first-class interface, not just a launcher. The subcommands you will actually reach for:

| Command | What it does |
|---|---|
| `opencode` / `opencode tui` | Launch the TUI. Flags: `--continue/-c`, `--session/-s`, `--fork`, `--prompt`, `--model/-m`, `--agent`, `--port`, `--hostname`, `--mdns`, `--cors`. |
| `opencode run` | **Non-interactive / headless** prompt run — the scripting and CI entrypoint. Flags include `--continue/-c`, `--session/-s`, `--fork`, `--share`, `--model/-m`, `--agent`, `--file/-f`, `--format`, `--dangerously-skip-permissions`, `--replay`. |
| `opencode serve` | Headless HTTP/SSE server, **no UI**. `--port`, `--hostname`, `--mdns`, `--cors`. |
| `opencode web` | Headless server **with** the web interface. Same flags as `serve`. |
| `opencode attach` | Attach a TUI to an already-running backend. `--dir`, `--continue/-c`, `--session/-s`, `--fork`, `--password/-p`, `--username/-u`. |
| `opencode auth login` | Authenticate a provider. `--provider/-p`, `--method/-m`. Also `auth list/ls`, `auth logout`. |
| `opencode models [provider]` | List available models. `--refresh`, `--verbose`. |
| `opencode agent create` / `agent list` | Scaffold and list agents. |
| `opencode mcp ...` | `add`, `list/ls`, `auth [name]`, `logout [name]`, `debug <name>`. |
| `opencode session list` / `session delete <id>` | Manage sessions. `--max-count/-n`, `--format`. |
| `opencode export [id]` / `import <file>` | Export a session (`--sanitize`); import JSON or a share URL. |
| `opencode stats` | Usage stats. `--days`, `--tools`, `--models`, `--project`. |
| `opencode github install` / `github run` / `opencode pr <number>` | GitHub Actions integration; pull a PR into a session. |
| `opencode plugin <module>` / `opencode generate` / `opencode db [query]` | Plugin install, OpenAPI generation, raw session DB access. |

Global flags worth knowing: `--help/-h`, `--version/-v`, `--print-logs`, `--log-level`, `--pure` (ignore all config/env discovery).

The rule of thumb: **live in the TUI for interactive work; reach for `opencode run` the moment you want a prompt in a script, a git hook, or CI; reach for `serve`/`attach` when the engine should outlive any single frontend.**

---

## 3. Essential workflow inside the TUI

### The leader key

OpenCode's TUI is leader-driven. The **default leader is `ctrl+x`** (configurable timeout, default 2000 ms), set under `keybinds` in `tui.json`. The bindings that earn their keystrokes:

| Keys | Action |
|---|---|
| `ctrl+x l` | List / resume sessions |
| `ctrl+x n` | New session |
| `ctrl+x m` | Switch model (`/models`) |
| `ctrl+x t` | Switch theme |
| `ctrl+x e` | Open `$EDITOR` for the prompt |
| `ctrl+x x` | Export conversation to Markdown |
| `ctrl+x c` | Compact / summarize the session |
| `ctrl+x u` / `ctrl+x r` | Undo / redo |
| `ctrl+x q` | Exit |
| `Tab` | **Cycle primary agents** (Build ↔ Plan) — the `switch_agent` keybind |
| `ctrl+p` | Command palette (`command_list`) |
| `ctrl+t` | Toggle reasoning/thinking display |

### Sending context

- **`@`** — fuzzy file search; inserts the file's content as context.
- **`` !`command` ``** — runs a shell command and injects its output into the prompt. Same pattern as Claude Code's dynamic injection.
- **Image drag-and-drop** — supported; configured under `attachment.image`.
- **`/editor`** (or `ctrl+x e`) — drops you into `$EDITOR` for composing a long prompt. Set it with `export EDITOR="code --wait"` to compose in VS Code.

### Permission modes: Plan vs Build

OpenCode's "modes" are now **primary agents** you cycle with `Tab`:

| Agent | Behavior | When to use |
|---|---|---|
| **Build** | Default. All tools allowed; edits and bash run per your permission config | Implementation |
| **Plan** | Restricted — `edit` and `bash` default to `ask` | Exploration, design, anything multi-file before you commit to writes |

This maps cleanly onto the plan-first habit: explore in **Plan**, flip to **Build** once the approach is locked. Permissions are configurable far more granularly than the two-mode summary suggests — see §7.

### Sessions and sharing

Sessions are first-class and server-side. `/sessions` (or `ctrl+x l`) lists, switches, and resumes them. As of recent releases you can **move a session between workspaces/directories** and attach **custom metadata** via the API/SDK. `opencode session list` and `session delete` manage them from the CLI; `opencode export`/`import` move them between machines (or in from a share URL).

To share a live session:

```
/share        # creates a public URL (opncd.ai/s/<id>) on your clipboard
/unshare      # revokes access and deletes the synced conversation
```

The `share` config key controls policy: `"manual"` (default — explicit `/share`), `"auto"` (share every session), or `"disabled"`. Enterprises can self-host the share infrastructure or gate it behind SSO.

### Undo / redo as a safety net

`ctrl+x u` / `ctrl+x r` (and `/undo` / `/redo`) roll the working tree and conversation back and forward. This is the "try something risky, undo if it doesn't pan out" affordance — not a git replacement, but cheap.

---

## 4. Commands: built-in and custom

### Built-in slash commands

`/init`, `/undo`, `/redo`, `/share`, `/unshare`, `/help`, `/connect`, `/models`, `/sessions`, `/editor`, `/export`, `/compact`. A custom command with the same name overrides the built-in.

> **`/connect` vs `auth login`.** Both appear in current docs for provider setup — `/connect` inside the TUI, `opencode auth login` from the CLI. They reach the same place; use whichever is in front of you.

### Custom commands

A custom command is a Markdown file. Precedence: project beats global.

| Scope | Path |
|---|---|
| Global | `~/.config/opencode/commands/<name>.md` |
| Project | `.opencode/commands/<name>.md` |
| Inline | the `"command"` key in `opencode.json(c)` |

> **Singular vs plural.** Plural (`commands/`, `agents/`) is the documented canonical form; the singular (`command/`, `agent/`) still loads for backwards-compat. Note `opencode agent create` is reported to write the **singular** `.opencode/agent/` (issue #14410) while the docs say plural — so don't be surprised to see both on disk.

Frontmatter keys: `description`, `agent`, `model`, `subtask` (force the command to run in a subagent), `template` (the prompt; the Markdown body also serves as the template).

Substitutions, mirroring the patterns you already know:

| Token | Meaning |
|---|---|
| `$ARGUMENTS` | Everything passed after the command name |
| `$1`, `$2`, `$3` | Positional arguments |
| `` !`command` `` | Inject the output of a shell command |
| `@file` | Inject a file's contents |

Example — a `.opencode/commands/test.md`:

```markdown
---
description: Run the test suite for a package and triage failures
agent: build
---

Run the tests for: $ARGUMENTS

- Current branch: !`git branch --show-current`
- Test config: @vitest.config.ts

Run the suite, and for any failure report the exact command, the assertion,
and the smallest fix. Do not change unrelated code.
```

Invoke it with `/test packages/api`.

---

## 5. Agents and subagents: the unit of scoped behavior

OpenCode splits agents into two roles:

- **Primary agents** — you talk to them directly and cycle them with `Tab`. The built-ins are **Build** (all tools) and **Plan** (read-leaning, edits/bash gated to `ask`).
- **Subagents** — invoked by a primary agent or by `@mention`. Built-ins: **General** (full tools), **Explore** (read-only codebase investigation), **Scout** (read-only external/dependency research).

This is the answer to "infinite exploration burns my context": delegate the read-heavy investigation to **Explore** or **Scout**, which run in their own context and report back a summary, keeping the primary conversation clean.

### Defining an agent

Two ways. Either the `"agent"` object in `opencode.json`, or a Markdown file where the filename is the agent name (`review.md` → the `review` agent):

| Scope | Path |
|---|---|
| Global | `~/.config/opencode/agents/<name>.md` |
| Project | `.opencode/agents/<name>.md` |

```markdown
---
description: Read-only reviewer that hunts for race conditions and swallowed errors
mode: subagent
model: anthropic/claude-sonnet-4-5
permission:
  edit: deny
  bash: deny
temperature: 0.2
---

You review diffs. Use Read and Grep only. Report:
1. Concurrency hazards
2. Errors that are logged-and-ignored or silently swallowed
3. Test gaps you would be embarrassed to ship without

Never modify files.
```

Agent config keys: `description` (required), `mode` (`primary` | `subagent` | `all`), `model`, `prompt` (path), `temperature`, `top_p`, `steps` (max iterations), `permission`, `disable`, `hidden`, `color`. Agent-level permissions **merge with global, and the agent's rules win** on conflict.

`opencode agent create` runs an interactive scaffold (`--path`, `--description`, `--mode`, `--permissions`, `--model/-m`): scope → description → generated prompt/id → permissions → writes the Markdown file.

---

## 6. AGENTS.md, instructions, and memory

OpenCode's persistent-instruction story is built on the **`AGENTS.md`** open convention rather than a proprietary file, with a Claude Code fallback so existing repos work unchanged.

### Where instruction files load (broadest first)

| Scope | Path | Use for |
|---|---|---|
| Project (nested) | `AGENTS.md` at the project root **and** in subdirectories | Team-shared, committed conventions |
| Global | `~/.config/opencode/AGENTS.md` | Personal preferences across all projects |
| Claude fallback (project) | `CLAUDE.md` | Reuse an existing Claude Code repo as-is |
| Claude fallback (global) | `~/.claude/CLAUDE.md` | Reuse your personal Claude memory |

Resolution order: local files walking **upward from the opened directory** (`AGENTS.md` then `CLAUDE.md`) → global `AGENTS.md` → `~/.claude/CLAUDE.md`. `/init` generates or updates the project `AGENTS.md`.

Beyond those well-known names, the `instructions` config key accepts explicit paths, globs, and even remote URLs (5 s fetch timeout):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": ["docs/guidelines.md", "packages/*/AGENTS.md"]
}
```

The writing standards that apply to `CLAUDE.md` apply verbatim here: be specific, keep it short, include what the agent can't guess (build/test commands, project-specific architecture, gotchas) and exclude what it can read from the code. When a rule keeps getting ignored, the file is probably too long.

---

## 7. Configuration and permissions

### The config files

- Project: `opencode.json` (or `.jsonc`).
- Global: `~/.config/opencode/opencode.json`.
- TUI keybinds/theme: `tui.json` (project or global).
- Schema: `"$schema": "https://opencode.ai/config.json"` — include it for autocomplete and validation.
- Env overrides: `OPENCODE_CONFIG` (explicit path), `OPENCODE_CONFIG_DIR`, `OPENCODE_CONFIG_CONTENT` (inline JSON).

Precedence (low → high): remote `.well-known/opencode` → global → `OPENCODE_CONFIG` → project → `.opencode` dirs → inline `OPENCODE_CONFIG_CONTENT` → managed/enterprise settings. Values support `{env:VAR}` and `{file:path}` substitution.

Core keys you will actually set: `model`, `small_model` (the cheap utility model), `provider`, `default_agent`, `instructions`, `tools`, `permission`, `mcp`, `plugin`, `formatter`, `lsp`, `share`, `autoupdate`, `compaction`, `server`.

### The permission model

Far more granular than "ask before edits." Each tool class takes one of `"allow"` (auto), `"ask"` (prompt), or `"deny"` (block):

`read`, `edit` (covers edit/write/patch), `bash`, `glob`, `grep`, `task` (launch a subagent), `skill`, `lsp`, `question`, `webfetch`, `websearch`, `external_directory`, `doom_loop`.

Defaults: most are `"allow"`; **`doom_loop` and `external_directory` default to `"ask"`**; `.env` writes are denied by default while `.env.example` is allowed. Matching uses `*`/`?` globs with **last-matching-rule-wins** semantics — and bash rules need explicit wildcards (`"git *"` permits arguments; a bare `"git"` would block anything with args).

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

### Providers and switching models

`opencode auth login` (or `/connect`) stores credentials in `~/.local/share/opencode/auth.json`. Switch models with `/models` (`ctrl+x m`), `--model/-m provider/model` on the CLI, or the `"model"` config key (e.g. `"anthropic/claude-sonnet-4-5"`); `"small_model"` sets the utility model. Custom or local providers go under `"provider"` with `options.baseURL`, `models`, `apiKey` (`{env:VAR}`), and `headers`.

> **Claude subscription auth — moving target.** Around Jan 2026 Anthropic added server-side protections restricting third-party OAuth to Claude, and OpenCode reportedly removed its Claude OAuth code in Feb 2026. Claude Pro/Max login now lives largely in **community plugins** (e.g. `opencode-anthropic-auth`, `opencode-claude-code-auth`), with reported ToS/ban risk. If you intend to drive OpenCode with a Claude subscription rather than an API key, **confirm the current state first** — this is the single most stale-prone fact in this report. Anthropic-API-key auth and every non-Anthropic provider are unaffected.

---

## 8. MCP, plugins, and skills

### MCP servers

Configured under the top-level `"mcp"` key, keyed by server name:

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

You can also disable servers in bulk via the `tools` glob (`"my-mcp*": false`).

> **Doc inconsistency.** The CLI reference lists `opencode mcp add`, while the MCP prose says configuration is file-based with no add command. The CLI command is the newer addition; both the command and hand-editing the config work.

### Plugins and hooks

Plugins are how you make OpenCode deterministic where instructions are advisory. They live in `.opencode/plugins/` (project), `~/.config/opencode/plugins/` (global), or as npm packages listed in `"plugin": [...]`. A plugin is a function receiving `{ project, client, $, directory, worktree }` (`client` is the SDK, `$` is Bun's shell) and returning hook handlers:

```js
export const Formatter = async ({ $ }) => ({
  "file.edited": async ({ path }) => {
    if (path.endsWith(".go")) await $`gofumpt -w ${path}`
  },
})
```

Hook events span the lifecycle: `tool.execute.before` / `tool.execute.after`, `file.edited`, `file.watcher.updated`, `permission.asked` / `permission.replied`, `session.created` / `compacted` / `idle` / `error`, `lsp.client.diagnostics`, `command.executed`, `tui.toast.show`, and more. Plugins can also define **custom tools** (Zod schema + `execute`) and override built-ins by name. Add a `package.json` to `.opencode/` and OpenCode runs `bun install` for you.

### Formatters and LSP — auto-integrated

`"formatter": true` and `"lsp": true` (or object configs) wire up code formatting and language servers; **LSP diagnostics feed back into the agent loop**, so the model sees type errors it introduces. `watcher.ignore` controls which paths the file watcher tracks.

### Skills

OpenCode implements the Anthropic-originated **Agent Skills** open standard (skill discovery + file-based loading landed in v1.16.0). A skill is a directory with a `SKILL.md`. Locations searched include `.opencode/skills/<name>/SKILL.md` and `~/.config/opencode/skills/<name>/SKILL.md`, plus Claude-compat (`.claude/skills/...`, `~/.claude/skills/...`) and `.agents/skills/...` paths — so Claude Code skills are reusable as-is.

Frontmatter: `name` (required, `^[a-z0-9]+(-[a-z0-9]+)*$`, must match the directory), `description` (required), and optional `license`, `compatibility`, `metadata`. Agents invoke skills via the `skill` tool (`skill({ name: "git-release" })`), governed by the `skill` permission (with wildcard patterns like `internal-*`).

> Some third-party guides cite `~/.opencode/skills/` for personal skills; the official path is `~/.config/opencode/skills/`. Prefer the official one.

---

## 9. An agentic walkthrough: building a small project

Concrete example: a Go CLI that scrapes RSS feeds, deduplicates entries against a SQLite cache, and pushes summaries to Slack. The point isn't the project — it's how the OpenCode pieces compose. Substitute any small project; the choreography holds.

### Step 0 — Lay the foundation (once)

In a fresh repo:

```bash
opencode
```

Then `/init` to scan and generate `AGENTS.md`. Review and trim it — drop anything that just restates the language ("Go uses gofmt"; the model knows). Commit.

Add a project config that makes the safe path the default and wires up formatting:

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

Add a `run-checks` command you'll reuse — `.opencode/commands/run-checks.md`:

```markdown
---
description: Run the full local verification pipeline
agent: build
---

Run, in order, stopping at the first failure and reporting the exact command + output:

1. `gofumpt -l -w .` — must produce zero diffs
2. `go vet ./...`
3. `golangci-lint run`
4. `go test -race -count=1 -coverprofile=coverage.out ./...`

Then report any package below 70% coverage. Do not fix anything unless asked.
```

And a read-only reviewer subagent (`.opencode/agents/review.md`) as shown in §5.

### Step 1 — Explore (Plan agent)

`Tab` to the **Plan** agent (edits and bash are gated to `ask`), then:

```
I want to add an RSS fetcher with a SQLite-backed dedup cache. Read the repo and tell me:
1. what's already here I can reuse
2. where new code should live to match conventions
3. which go.mod dependencies already solve part of this
```

If the codebase is big enough that reading it would burn context, delegate to the **Explore** subagent — it runs in its own context and returns a summary, keeping your main thread clean.

### Step 2 — Plan

Still in Plan:

```
Propose an implementation plan for:
- internal/feeds:  RSS fetcher with retry + timeout
- internal/cache:  SQLite dedup cache keyed by GUID + content hash
- cmd/scrape:      CLI entrypoint

For each package: the files, the exported API, and the test cases.
Flag any decision I should weigh in on.
```

Lock the API surface in conversation, then `Tab` back to **Build**.

### Step 3 — Implement

```
Implement the plan. Write tests first where the contract is clear. After each
package, run /run-checks and fix what it surfaces.
```

While it runs: watch the diff viewer (next/previous-hunk nav makes this fast), reject anything off and say why, and use `ctrl+x u` to roll back a change that goes sideways. If you've corrected the same mistake twice, start a fresh session (`ctrl+x n`) with a tighter prompt rather than fighting a polluted context.

### Step 4 — Review (fresh context, different lens)

New session (`ctrl+x n`), then hand it to the read-only reviewer:

```
@review the diff on this branch.
```

A fresh context with no implementation bias catches what the implementer rationalized away.

### Step 5 — Ship

```
Commit with a clear message and open a PR. In the description: a one-line summary,
the new commands users get, and a note that a SQLite file is created on first run.
```

For CI, the same work runs headless:

```bash
opencode run --agent review "Review the diff against origin/main for race conditions and swallowed errors"
```

### When to scale up

- **Continuous verification** → move "must always happen" rules out of `AGENTS.md` and into a **plugin** hook (`file.edited` to format, `session.idle` to test). Hooks are deterministic; instructions are advisory.
- **A workflow reused across projects** → promote a project command/agent/skill into `~/.config/opencode/`, or package it as a plugin / publish a skill.
- **The engine should outlive the frontend** → `opencode serve` on a remote box, `opencode attach` from your laptop; or share a live session with `/share`.

---

## 10. Failure patterns to avoid

The same ones that bite every agentic workflow, in OpenCode's idiom:

- **The kitchen-sink session.** One task drifts into a tangent and the context fills with noise. *Fix:* `ctrl+x n` for a fresh session between unrelated tasks; `ctrl+x c` to compact when one task legitimately runs long.
- **The correction loop.** Two failed corrections means the context is polluted with dead approaches. *Fix:* new session, rewrite the prompt with what you learned.
- **Over-stuffed `AGENTS.md`.** Long enough that important rules get lost. *Fix:* prune, and convert "must always" rules into plugin hooks.
- **Trust without verify.** Plausible code that misses the edge case. *Fix:* always give the agent something concrete to check against — `/run-checks`, a test, a script. Lean on the LSP feedback loop.
- **Infinite exploration.** "Investigate X" with no scope reads hundreds of files. *Fix:* delegate to the **Explore**/**Scout** subagents (separate context), or scope the prompt to a path.
- **Permission paper cuts.** Constant `ask` prompts on safe commands. *Fix:* widen the `permission.bash` allowlist with explicit wildcards (`"go *": "allow"`) rather than reaching for `--dangerously-skip-permissions`.

---

## 11. Useful references

- Docs home: https://opencode.ai/docs/
- CLI reference: https://opencode.ai/docs/cli/
- Config: https://opencode.ai/docs/config/
- Agents: https://opencode.ai/docs/agents/
- Commands: https://opencode.ai/docs/commands/
- TUI / keybinds: https://opencode.ai/docs/tui/
- Permissions: https://opencode.ai/docs/permissions/
- Providers: https://opencode.ai/docs/providers/
- MCP servers: https://opencode.ai/docs/mcp-servers/
- Plugins: https://opencode.ai/docs/plugins/
- Skills: https://opencode.ai/docs/skills/
- Rules / `AGENTS.md`: https://opencode.ai/docs/rules/
- Share: https://opencode.ai/docs/share/
- SDK: https://opencode.ai/docs/sdk/
- Repo & releases: https://github.com/sst/opencode (releases under `/releases`)
- Model registry: https://models.dev
