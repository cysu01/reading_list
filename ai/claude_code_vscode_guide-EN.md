# Claude Code in VS Code — A Practitioner's Report

> Scope: how to drive Claude Code from inside VS Code, the commands worth memorizing, how skills and `CLAUDE.md` are structured, and how all of it composes into an agentic workflow for a small project. Aimed at a developer who already knows Claude Code's terminal mode and wants the VS Code-native picture.

---

## 1. The two interfaces (and why this matters)

When you install the **Claude Code extension for VS Code**, you get two cooperating surfaces. They share the same auth, settings, conversation history, MCP servers, and `~/.claude/` configuration, but they expose different feature subsets.

| | **Extension (Spark panel)** | **CLI in integrated terminal** |
|---|---|---|
| Launch | Spark icon (top-right of editor), Activity Bar, status bar, or `Cmd/Ctrl+Shift+P` → "Claude Code: Open in New Tab" | `` Cmd/Ctrl+` `` then `claude` |
| Diffs | Native side-by-side diff with accept/reject | Same — CLI uses VS Code's diff viewer via the built-in `ide` MCP server |
| Slash commands | A subset; type `/` to see what's available | Full set |
| MCP server config | Add servers via CLI; manage existing with `/mcp` | Full CRUD |
| `!` bash shortcut, tab completion | Not available | Available |
| Checkpoints, plugins | Yes | Yes |

The rule of thumb: **use the extension for inline diffs, multi-tab conversations, and @-mentions; drop into the terminal when you need a command, flag, or MCP operation the panel doesn't expose.** Conversations cross-pollinate — `claude --resume` in the integrated terminal opens a picker that includes panel sessions.

### Prerequisites

- VS Code 1.98.0 or higher
- An Anthropic account (or Bedrock/Vertex/Foundry credentials if you route through a provider)
- Install via `Cmd/Ctrl+Shift+X` → search "Claude Code" → Install, or the marketplace URI `vscode:extension/anthropic.claude-code`

### Where Claude lives in the window

You can drag the panel to:

- **Secondary sidebar** (right): keeps Claude visible while you code — best default
- **Primary sidebar** (left): if you want it grouped with Explorer/Search
- **Editor area**: opens as a tab; good for side conversations

Run `Cmd/Ctrl+Shift+Esc` to open a new conversation as an editor tab. `Cmd/Ctrl+Esc` toggles focus between the editor and Claude's prompt box.

---

## 2. Essential workflow inside the extension

### Sending context

- **Selected text is automatic.** Highlight code and Claude sees it. The footer of the prompt box shows the line count.
- **`Option/Alt+K`** inserts an explicit `@file.ts#5-10` reference from your selection.
- **`@filename`** with fuzzy matching to attach a file or folder (trailing slash for folders): `@src/auth/` or `@auth` (matches `auth.js`, `AuthService.ts`, etc.).
- **`@terminal:name`** pipes a terminal pane's output into the prompt — useful for stack traces and command output you don't want to copy-paste.
- **`@browser`** (requires Claude in Chrome extension) drives a browser tab for E2E flows and console-log debugging.
- **Shift-drag files** into the prompt box to attach them, or paste images directly.

### Permission modes (the throttle)

Click the mode indicator at the bottom of the prompt box, or set `claudeCode.initialPermissionMode` in settings:

| Mode | Behavior | When to use |
|---|---|---|
| `default` | Asks before each file write or Bash command | Default; safest |
| `plan` | Claude reads and proposes a plan but won't touch files | Exploration, complex refactors, anything multi-file |
| `acceptEdits` | Auto-accepts file edits, still asks for Bash | Cruising on a known scope |
| `auto` | Classifier model approves routine actions, blocks risky ones (scope escalation, unknown infra, hostile-content-driven actions) | Long unattended runs; requires Team/Enterprise/API plan and Sonnet/Opus 4.6 |
| `bypassPermissions` | No prompts at all | **Only in sandboxes with no internet** |

Plan mode is the big one for an agentic workflow — VS Code renders the plan as a markdown document you can edit inline before approving. Press `Ctrl+G` to open the plan in the editor and add comments.

### Multiple conversations

Open multiple tabs/windows for parallel tasks (researcher in one, implementer in another, reviewer in a third). Each maintains independent history. A small dot on the Spark icon indicates state: blue = permission request pending, orange = response ready in a hidden tab.

For truly parallel work on the same repo, use git worktrees:

```bash
claude --worktree feature-auth
```

Each worktree has its own files and branch; instances don't interfere with each other.

### Checkpoints (rewind)

Every Claude action is automatically checkpointed. Hover any message in the panel to reveal the rewind button. Three options:

- **Fork conversation from here** — branch the chat, keep code as-is
- **Rewind code to here** — revert files, keep conversation
- **Fork and rewind** — both

This is not a git replacement (external changes aren't tracked, neither are most Bash side effects), but it makes "try something risky, undo if it doesn't pan out" cheap.

---

## 3. The commands that earn their keyboard cycles

Type `/` in the prompt box for the live menu. Below are the ones that matter day to day, grouped by purpose. `<arg>` is required, `[arg]` optional.

### Project setup and memory

| Command | What it does |
|---|---|
| `/init` | Scans the repo and generates a starter `CLAUDE.md` with detected build, test, and convention info. Set `CLAUDE_CODE_NEW_INIT=1` for an interactive flow that also proposes skills, hooks, and personal memory files. |
| `/memory` | Lists every memory file loaded this session (`CLAUDE.md`, `CLAUDE.local.md`, all `.claude/rules/*.md`), toggles auto memory, opens the auto-memory folder. Open any file to edit it. |
| `/skills` | Lists available skills (bundled, user, project, plugin). |
| `/agents` | Manage subagent configurations. |
| `/hooks` | Inspect configured hooks for tool events. |
| `/permissions` | Open the allow/ask/deny rule editor; view recent auto-mode denials. Alias: `/allowed-tools`. |
| `/mcp` | Manage MCP server connections and OAuth. |
| `/plugin` | Manage Claude Code plugins (skill/hook/agent/MCP bundles). |
| `/config` | Open the settings UI (theme, model, output style). Alias: `/settings`. |

### During a session

| Command | What it does |
|---|---|
| `/plan [description]` | Drop straight into plan mode, optionally with the task. Example: `/plan refactor auth to use OAuth2`. |
| `/clear` | Reset conversation context. Aliases: `/reset`, `/new`. Use between unrelated tasks — bloated context degrades performance. |
| `/compact [focus]` | Summarize history to free context, optionally with focus instructions. |
| `/context` | Visualize current context usage as a grid; flags context-heavy tools and memory bloat. |
| `/rewind` | Open the rewind menu (also `Esc Esc`). Restore conversation, code, both, or summarize from a selected message. Alias: `/checkpoint`. |
| `/branch [name]` | Fork the conversation at the current message. Alias: `/fork`. |
| `/btw <question>` | Ask a side question without polluting conversation history — answer appears in a dismissible overlay. |
| `/diff` | Open interactive diff viewer; navigate uncommitted changes and per-turn diffs. |
| `/copy [N]` | Copy the last (or Nth-latest) assistant response; with code blocks, opens a picker. |

### Model and effort

| Command | What it does |
|---|---|
| `/model [name]` | Change model; left/right arrows adjust effort level. Takes effect immediately. |
| `/effort low\|medium\|high\|max\|auto` | Set reasoning effort. `low/medium/high` persist across sessions; `max` is one-shot and requires Opus 4.6. |
| `/fast on\|off` | Toggle fast mode. |

### Investigation and review

| Command | What it does |
|---|---|
| `/security-review` | Scan pending branch changes for injection, auth, and data-exposure issues. |
| `/pr-comments [PR]` | Fetch PR comments via `gh` CLI (auto-detects current branch). |
| `/insights` | Generate a report on your Claude Code session patterns and friction points. |

### Bundled skills (appear alongside slash commands)

These are prompt-based playbooks that ship with Claude Code. They differ from built-in commands because they orchestrate work using Claude's tools instead of running fixed logic.

| Skill | Purpose |
|---|---|
| `/batch <instruction>` | Decompose a large change into 5–30 units, spawn one background agent per unit in its own worktree, each opens a PR. Requires a git repo. Example: `/batch migrate src/ from Mocha to Vitest`. |
| `/simplify [focus]` | Spawn three parallel review agents over recently changed files, aggregate findings, apply fixes. Optionally focus: `/simplify focus on memory efficiency`. |
| `/debug [description]` | Enable debug logging and analyze the session debug log. |
| `/loop [interval] <prompt>` | Run a prompt periodically while the session is open. Example: `/loop 5m check if the deploy finished`. |
| `/claude-api` | Load Claude API and Agent SDK reference for your project's language. |

For everything else — `/help` lists the full menu; `/doctor` diagnoses installation issues.

---

## 4. Skills: the unit of repeatable Claude behavior

A skill is a directory containing a `SKILL.md` file. Claude loads the description into context at startup and pulls in the full body either when you invoke it with `/skill-name` or when it decides the description matches what you're asking for.

### Where skills live (precedence: top wins)

| Scope | Path | Applies to |
|---|---|---|
| Enterprise (managed) | Set via managed settings | All users in org |
| Personal | `~/.claude/skills/<name>/SKILL.md` | All your projects |
| Project | `.claude/skills/<name>/SKILL.md` | This project only |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | Wherever plugin is enabled |

Plugin skills use a `plugin-name:skill-name` namespace and can't collide with other levels.

> **Note on commands vs skills.** Custom slash commands have been merged into skills. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work identically. Skills are recommended because they support supporting files, frontmatter controls, and automatic invocation.

### Anatomy of a skill

```text
my-skill/
├── SKILL.md          # required entrypoint
├── reference.md      # optional — Claude loads on demand
├── examples.md       # optional — loaded on demand
└── scripts/
    └── helper.py     # optional — executable, not loaded into context
```

Keep `SKILL.md` under ~500 lines. Move heavy reference material into separate files and link to them from `SKILL.md` so Claude loads them only when needed.

### Frontmatter reference

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

All fields are optional. Only `description` is recommended.

### String substitutions

| Token | Meaning |
|---|---|
| `$ARGUMENTS` | Everything passed after the skill name |
| `$ARGUMENTS[N]` or `$N` | Nth positional arg, zero-indexed |
| `${CLAUDE_SESSION_ID}` | Current session ID — handy for logging |
| `${CLAUDE_SKILL_DIR}` | Skill's own directory — use this in scripts so paths don't break when the working dir changes |

### Dynamic context injection — the `` !`command` `` pattern

This is one of the most underused features. Place `` !`shell command` `` anywhere in the skill body. The command runs **before** Claude sees the skill, and its output replaces the placeholder. Claude receives the rendered prompt, not the command itself.

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

### Invocation control matrix

| Frontmatter | You can invoke | Claude can auto-invoke | When loaded into context |
|---|---|---|---|
| (default) | Yes | Yes | Description always in context; body loads on invoke |
| `disable-model-invocation: true` | Yes | No | Description not in context; body loads on your invoke |
| `user-invocable: false` | No | Yes | Description always in context; body loads on auto-invoke |

Use `disable-model-invocation: true` for anything with side effects you want explicit timing for: `/deploy`, `/release`, `/send-slack-message`.

---

## 5. CLAUDE.md and the memory system

Each session starts with a fresh context window. Two complementary mechanisms carry knowledge across sessions:

- **`CLAUDE.md` files** — you write them; persistent instructions
- **Auto memory** — Claude writes them; learnings and patterns

Both load every session and are treated as context (not enforced configuration). The more specific and concise, the more reliably Claude follows them.

### Where CLAUDE.md files go (load order: broadest first)

| Scope | Path | Use for |
|---|---|---|
| Managed policy | OS-specific (e.g. `/Library/Application Support/ClaudeCode/CLAUDE.md` on macOS) | Org-wide standards, compliance |
| User | `~/.claude/CLAUDE.md` | Personal preferences across all projects |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team-shared, committed to source control |
| Local | `./CLAUDE.local.md` | Personal project-specific; add to `.gitignore` |

Files are concatenated (not overridden), walked up from your CWD to filesystem root. Nested `CLAUDE.md` in subdirectories load **on demand** when Claude reads files in those directories.

### Writing effective instructions

The single biggest predictor of adherence is **specificity** and **size**. Target under 200 lines per file.

| ✅ Include | ❌ Exclude |
|---|---|
| Build/test commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions |
| Repository etiquette (branch naming, PR conventions) | Long tutorials or API docs (link to them) |
| Architectural decisions specific to this project | File-by-file codebase descriptions |
| Env vars, dev environment quirks, gotchas | Self-evident practices ("write clean code") |

Pattern that works: when you correct Claude on the same thing twice, codify it in `CLAUDE.md`. When you see a rule getting ignored, the file is probably too long — prune ruthlessly. The test for each line is "would removing this cause Claude to make mistakes?" If no, cut it.

### `.claude/rules/` for path-scoped instructions

For larger projects, split instructions across files in `.claude/rules/`. Rules without frontmatter load unconditionally (same as `.claude/CLAUDE.md`). Add a `paths` field and the rule loads only when Claude touches matching files:

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

Glob patterns support extension matching (`**/*.ts`), directory scoping (`src/**/*`), and brace expansion (`src/**/*.{ts,tsx}`).

### Imports

`@path/to/file` syntax expands a file inline at load time. Relative paths resolve from the file containing the import; absolute paths and `~/` also work. Max depth 5 hops.

```markdown
See @README.md for project overview and @package.json for npm scripts.

# Workflow
- Git: @docs/git-instructions.md
- Personal: @~/.claude/my-overrides.md
```

External imports trigger a one-time approval dialog the first time Claude encounters them.

### `AGENTS.md` interop

Claude Code reads `CLAUDE.md`, not `AGENTS.md`. If your repo uses `AGENTS.md` for other agents, either symlink or import:

```markdown
@AGENTS.md

## Claude Code specifics
Use plan mode for changes under `src/billing/`.
```

Running `/init` in a repo with an existing `AGENTS.md` (or `.cursorrules`, `.windsurfrules`) merges relevant content into the generated `CLAUDE.md`.

### Auto memory

Off the shelf, Claude saves notes to `~/.claude/projects/<project>/memory/` as it works — build quirks, debugging discoveries, preferences you've expressed. Toggle via `/memory` or `autoMemoryEnabled` in settings; disable globally with `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. The first 200 lines (or 25 KB) of `MEMORY.md` load every session; topic files load on demand.

Auto memory is machine-local and shared across worktrees of the same repo.

---

## 6. An agentic walkthrough: building a small project

Concrete example: a Go CLI that scrapes RSS feeds, deduplicates entries against a SQLite cache, and pushes summaries to a Slack channel. End-to-end with tests, lint, and CI.

The goal here isn't the project — it's to show how the pieces compose. Substitute any small project; the choreography is the same.

### Step 0 — Lay the foundation (once, then forget)

In a fresh repo, in the integrated terminal:

```bash
claude
```

Then:

```
/init
```

Let Claude scan the repo and propose a `CLAUDE.md`. Review it in the diff viewer; trim anything that's just describing the language ("Go uses gofmt" — Claude knows). Commit.

Add a project rule for the data layer:

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

Add two skills you'll use repeatedly:

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

Optional but high-leverage: a `PostToolUse` hook on `Edit` that runs `gofumpt` on the touched file automatically. Configure in `.claude/settings.json` or ask Claude to write it for you: *"add a hook that runs gofumpt on any Go file Claude edits."*

### Step 1 — Explore (Plan mode, read only)

Switch the panel to **Plan mode**. In a new conversation:

```
/plan I want to add an RSS feed fetcher with a SQLite-backed dedup cache.
Look through the repo and tell me:
1. What's already here that I can reuse
2. Where new code should live to match existing conventions
3. What external dependencies are already in go.mod that solve any of this
```

Claude reads files, doesn't touch anything. The plan opens as a markdown doc in the editor. Press `Ctrl+G` to edit it directly — add constraints, remove out-of-scope items, push back on assumptions.

If the codebase is large enough that exploration would burn context, delegate to a subagent instead:

```
Use a subagent to investigate how we currently fetch external HTTP resources
and what testing patterns we use for things that hit the network. Report back
with file references and the recommended pattern.
```

The subagent runs in its own context window and returns a summary. Your main conversation stays clean.

### Step 2 — Plan

Still in plan mode:

```
Propose an implementation plan for:
- internal/feeds: RSS fetcher with retry + timeout
- internal/cache: SQLite dedup cache keyed by GUID + content hash
- cmd/scrape: CLI entrypoint

For each package, list the files, the exported API, and the test cases.
Highlight any decisions you're making that I should weigh in on.
```

This produces a reviewable document. Edit the markdown plan to lock in the API surface. Switch back to **default** or **acceptEdits** mode when you're ready.

### Step 3 — Implement

```
/new-feature RSS feed fetcher with SQLite dedup cache, plus the cmd/scrape CLI
```

The skill runs through Explore → Plan → Implement → Verify → Commit. Because it's tagged `disable-model-invocation: true`, Claude won't trigger it on its own — you control timing.

While it runs:
- Watch the diff viewer. Reject anything that looks off and explain why; Claude course-corrects.
- If you see Claude spinning on a wrong approach, press `Esc` once to stop, then redirect. If you've corrected twice on the same thing, `/clear` and restart with a tighter prompt.
- Use `Esc Esc` (or `/rewind`) to roll back to a checkpoint if a change goes sideways.

### Step 4 — Review (fresh context, different lens)

Open a new tab (`Cmd/Ctrl+Shift+Esc`). Fresh context, no implementation bias:

```
Review the diff on this branch for:
1. Race conditions around the cache
2. Error handling — anything swallowed or logged-and-ignored
3. Test gaps — what would I be embarrassed to ship without

Use Read and Grep only. Don't write anything.
```

Or just run `/simplify`, which spawns three parallel review agents and applies fixes.

For a security pass:

```
/security-review
```

### Step 5 — Ship

Back in the implementation tab:

```
Commit the changes with a clear message and open a PR. In the PR description,
include: a one-line summary, the new commands users get, and a note about
the new SQLite file that gets created at first run.
```

Claude uses `gh` to create the PR. If you have GitHub Actions configured, you can also use Claude Code's review action — but that's a separate setup.

### When to scale up

Most small projects fit in one conversation. Reach for more machinery when:

- **A change spans many files independently.** `/batch` decomposes into worktrees and parallelizes. Good for migrations: `/batch migrate all handlers in internal/api to use the new context middleware`.
- **You want continuous verification.** Add hooks: `PostToolUse` on `Edit` to run formatters, `Stop` to run tests, `PreToolUse` to block writes to protected paths. Hooks are deterministic where `CLAUDE.md` is advisory.
- **A workflow is reused across projects.** Promote a project skill to your personal `~/.claude/skills/` or package it as a plugin.

---

## 7. Settings worth knowing

### Extension settings (VS Code-scoped)

Open with `Cmd/Ctrl+,` → Extensions → Claude Code.

| Setting | Default | Note |
|---|---|---|
| `claudeCode.selectedModel` | `default` | Per-conversation override with `/model` |
| `claudeCode.initialPermissionMode` | `default` | Set to `plan` if you want plan-first by default |
| `claudeCode.useTerminal` | `false` | Forces CLI-style interface in panel |
| `claudeCode.preferredLocation` | `panel` | `sidebar` or `panel` |
| `claudeCode.autosave` | `true` | Save files before Claude reads/writes |
| `claudeCode.respectGitIgnore` | `true` | Skip `.gitignore` patterns in file searches |
| `claudeCode.useCtrlEnterToSend` | `false` | Switches send key to Ctrl/Cmd+Enter |

### Claude Code settings (shared with CLI)

Live in `~/.claude/settings.json` and `.claude/settings.json` (project) / `.claude/settings.local.json` (gitignored, personal). Used for permissions, hooks, MCP servers, env vars, plugin enablement.

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

Including the `$schema` line gives you autocomplete and validation inside VS Code.

---

## 8. Failure patterns to avoid

These show up over and over; recognizing them is half the fix.

- **The kitchen-sink session.** One task → unrelated tangent → back to original task. Context is now full of noise. *Fix:* `/clear` between unrelated tasks.
- **Correction loop.** Claude does X wrong, you correct, still wrong, correct again. The context is polluted with failed approaches. *Fix:* after two failed corrections, `/clear` and rewrite the prompt incorporating what you learned.
- **Over-specified `CLAUDE.md`.** File is long enough that important rules get lost. *Fix:* prune. Convert "must always happen" rules into hooks.
- **Trust without verify.** Plausible-looking implementation that doesn't handle the edge case. *Fix:* always give Claude something concrete to check against — tests, a script, a screenshot. If you can't verify, don't merge.
- **Infinite exploration.** "Investigate X" without scope. Claude reads hundreds of files. *Fix:* delegate to a subagent (separate context), or scope the prompt — "investigate X by reading only the files matching `src/auth/**`."

---

## 9. Useful references

- VS Code extension docs: https://code.claude.com/docs/en/vs-code.md
- Built-in commands: https://code.claude.com/docs/en/commands.md
- Skills: https://code.claude.com/docs/en/skills.md
- Memory and `CLAUDE.md`: https://code.claude.com/docs/en/memory.md
- Best practices: https://code.claude.com/docs/en/best-practices.md
- Hooks: https://code.claude.com/docs/en/hooks.md
- Subagents: https://code.claude.com/docs/en/sub-agents.md
- MCP: https://code.claude.com/docs/en/mcp.md
- Full docs map: https://code.claude.com/docs/en/claude_code_docs_map.md