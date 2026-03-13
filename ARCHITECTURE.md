# gstack — Code Structure Analysis

This document is a structural analysis of the gstack codebase: how it is organized, what every module does, how the parts connect, and the design principles that hold it together.

---

## Repository layout

```
gstack/
├── browse/                  # The only compiled component — headless browser CLI
│   ├── src/                 # TypeScript source (8 files, ~1,625 lines)
│   ├── test/                # Integration tests + HTML fixtures
│   └── dist/                # Compiled binary (gitignored, ~58 MB)
├── plan-ceo-review/         # Skill: founder / CEO review mode
│   └── SKILL.md
├── plan-eng-review/         # Skill: engineering manager review mode
│   └── SKILL.md
├── review/                  # Skill: paranoid staff engineer code review
│   ├── SKILL.md
│   └── checklist.md
├── ship/                    # Skill: release engineer — automated shipping
│   └── SKILL.md
├── retro/                   # Skill: engineering manager retrospective
│   └── SKILL.md
├── SKILL.md                 # Root skill (browse) — Claude discovers this first
├── CLAUDE.md                # Developer quick-reference (commands, deploy steps)
├── README.md                # User-facing documentation
├── BROWSER.md               # Browser internals and full command reference
├── TODO.md                  # Roadmap, phased backlog
├── CHANGELOG.md             # Version history
├── VERSION                  # Current version string
├── setup                    # One-time installer: build binary + create skill symlinks
└── package.json             # Bun build scripts + runtime dependencies
```

### Two distinct layers

The repository has two layers that never mix:

| Layer | Contents | Language | Deliverable |
|-------|----------|----------|-------------|
| **Browser** | `browse/src/` | TypeScript | Compiled native binary |
| **Skills** | `*/SKILL.md` | Markdown | LLM prompt files |

The browser layer is real software — compiled, tested, benchmarked. The skill layer is pure orchestration — structured prompt files that Claude Code discovers and executes as slash commands. There is no code in the skill directories; their entire surface area is natural language.

---

## Browse binary — architecture

The browse binary is a persistent Chromium daemon with a thin CLI client.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Claude Code (or any shell)                                             │
│                                                                         │
│  $ browse goto https://staging.myapp.com                                │
│        │                                                                │
│        ▼                                                                │
│  ┌──────────┐    HTTP POST /command    ┌────────────────────────────┐   │
│  │ cli.ts   │ ──────────────────────▶  │ server.ts                  │   │
│  │ (client) │   localhost:9400+        │                            │   │
│  │          │   Authorization: Bearer  │  ┌──────────────────────┐  │   │
│  │          │ ◀──────────────────────  │  │ BrowserManager       │  │   │
│  │          │   plain text response    │  │ (browser-manager.ts) │  │   │
│  └──────────┘                          │  └──────────────────────┘  │   │
│  ~1 ms startup                         │          │                  │   │
│  (binary already compiled)             │          ▼                  │   │
│                                        │      Playwright API         │   │
│                                        │          │                  │   │
│                                        │          ▼                  │   │
│                                        │      Chromium (headless)    │   │
│                                        └────────────────────────────┘   │
│                                        persistent daemon                │
│                                        auto-starts on first call        │
│                                        auto-stops after 30 min idle     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Source map

| File | Lines | Role |
|------|-------|------|
| `cli.ts` | 249 | Entry point. Reads state file, starts server if needed, sends HTTP command, prints response. |
| `server.ts` | 268 | Bun HTTP server. Auth, routing, idle timeout, state file, log flush. |
| `browser-manager.ts` | 253 | Chromium lifecycle: launch, tab management, ref map, crash detection. |
| `snapshot.ts` | 212 | Accessibility tree → `@ref` assignment → Playwright Locator map. |
| `read-commands.ts` | 221 | Non-mutating queries: `text`, `html`, `links`, `forms`, `js`, `css`, `attrs`, etc. |
| `write-commands.ts` | 179 | Mutating operations: `goto`, `click`, `fill`, `select`, `scroll`, `wait`, etc. |
| `meta-commands.ts` | 199 | Server control (`status`, `stop`, `restart`) and visual output (`screenshot`, `pdf`, `responsive`, `diff`). |
| `buffers.ts` | 44 | In-memory ring buffers for console and network log capture. |

### Module dependency graph

```
cli.ts
  └── (HTTP) ──▶ server.ts
                   ├── browser-manager.ts
                   │     ├── buffers.ts
                   │     └── (Playwright API)
                   ├── snapshot.ts
                   │     └── browser-manager.ts
                   ├── read-commands.ts
                   │     └── browser-manager.ts
                   ├── write-commands.ts
                   │     └── browser-manager.ts
                   └── meta-commands.ts
                         ├── browser-manager.ts
                         └── snapshot.ts
```

`cli.ts` and `server.ts` are the only two executable entry points. All other modules are libraries that `server.ts` wires together. `buffers.ts` is a pure data structure — no Playwright dependency.

---

## Module deep-dives

### `cli.ts` — the thin client

The CLI's only job is to relay commands to a running server and print the result. It never touches Playwright directly.

**Startup flow:**

```
1. Read STATE_FILE (/tmp/browse-server.json)
2. If missing → spawn server.ts in background, poll /health until ready (max 8s)
3. If PID stale (server died) → remove state file, restart
4. POST /command with Bearer token + argv as JSON body
5. Print response to stdout; exit 1 on HTTP error
6. One auto-retry on connection-refused (handles rare race on startup)
```

**Key function:** `resolveServerScript()` — finds `server.ts` relative to the binary whether running in dev mode (source) or compiled mode (embedded path). This is what allows `bun run dev` and the compiled binary to share the same startup logic.

**Multi-workspace:** When `CONDUCTOR_PORT` is set, the port is derived as `CONDUCTOR_PORT - 45600`. Each workspace therefore gets a deterministically different port and its own isolated state file (`/tmp/browse-server-<port>.json`), Chromium process, and log files.

---

### `server.ts` — the daemon

The server is a single-process HTTP daemon that owns Playwright's lifecycle.

**Responsibilities:**

- Serve `POST /command` (all commands) and `GET /health`
- Validate `Authorization: Bearer <token>` on every request
- Dispatch to `handleReadCommand`, `handleWriteCommand`, or `handleMetaCommand`
- Reset the idle timer on every command; shut down after `BROWSE_IDLE_TIMEOUT` (default 30 min)
- Write state file (`port`, `token`, `pid`) with `chmod 600` on startup
- Flush console and network buffers to disk every 1 second
- Exit immediately on Chromium crash — no self-healing (fail visibly)

**Auth:** The token is a `crypto.randomUUID()` generated at startup. It is never reused across server restarts. This prevents other processes on the same machine from controlling the browser session.

**Command dispatch pattern:**

```typescript
// Every command goes through this single dispatch point:
POST /command  →  { command: string, args: string[] }
                     │
                     ├─ read commands  → handleReadCommand(bm, cmd, args)
                     ├─ write commands → handleWriteCommand(bm, cmd, args)
                     └─ meta commands  → handleMetaCommand(bm, cmd, args)
```

The three handler modules are imported once and called synchronously per request. There is no async queue — requests are handled one at a time (Bun's `serve` is single-threaded per request).

---

### `browser-manager.ts` — Chromium lifecycle

`BrowserManager` is the single class that owns the Playwright browser instance. All command handlers receive a `BrowserManager` reference and operate through it.

**State it owns:**

| Property | Type | Purpose |
|----------|------|---------|
| `browser` | `Browser` | The Playwright browser instance |
| `context` | `BrowserContext` | Shared context (cookies, storage) |
| `tabs` | `Map<string, Page>` | All open tabs keyed by ID |
| `activeTabId` | `string` | Currently focused tab |
| `refMap` | `Map<string, Locator>` | `@ref` → Playwright Locator, populated by snapshot |

**Key behaviors:**

- **Crash handling:** `browser.on('disconnected')` → `process.exit(1)` with an error log. The design principle is "fail loud, restart clean" — self-healing would mask real crashes.
- **Ref resolution:** `resolveRef(sel)` — if selector starts with `@`, look up in `refMap`; otherwise return a CSS Locator. This is the single point where `@ref` syntax is resolved, used by every command that accepts a selector.
- **Console/network wiring:** Every new `Page` is immediately wired to `buffers.ts` via `page.on('console')` and `page.on('response')`.

---

### `snapshot.ts` — the ref system

The snapshot system is the browser's key innovation for AI agents. Instead of having Claude guess CSS selectors, it assigns stable short names (`@e1`, `@e2`, ...) to every element in the accessibility tree and stores the mapping as Playwright Locators.

**Algorithm:**

```
1. page.locator(scope).ariaSnapshot()
      → YAML-like accessibility tree string

2. Parse tree line by line, assign @e1, @e2, ... to each node

3. For each ref, build a Playwright Locator:
      - getByRole(role, { name: label }) + nth(index) for disambiguation
      - Fall back to CSS-based locators for unlabeled elements

4. Store Map<string, Locator> on BrowserManager.refMap

5. Return the accessibility tree text with refs prepended to each line
```

**Why accessibility tree, not DOM?**

- No DOM mutation — works on pages with strict CSP
- No injected scripts — nothing that can trigger CSP violations or change behavior
- ~200-400 tokens vs ~3,000-5,000 for full DOM — critical for AI context budget
- Refs survive dynamic class names and style changes between renders

**Flags:**

| Flag | Effect |
|------|--------|
| `-i` | Interactive elements only (buttons, links, inputs, selects) |
| `-c` | Compact — skip empty structural wrappers |
| `-d N` | Limit tree depth to N levels |
| `-s sel` | Scope snapshot to a CSS selector |

---

### `read-commands.ts` — non-mutating queries

All commands that read state without changing the page. Each command is a function that receives `(bm: BrowserManager, args: string[])` and returns a `Promise<string>`.

| Command | What it returns |
|---------|----------------|
| `text` | Cleaned page text (no `<script>` or `<style>` content) |
| `html [sel]` | `innerHTML` of selector, or full page HTML |
| `links` | All `<a href>` as `text → href` pairs |
| `forms` | All form fields as JSON (name, type, value, options) |
| `accessibility` | Full Playwright accessibility tree |
| `js <expr>` | Result of `page.evaluate(expr)` serialized to string |
| `eval <file>` | Result of running a JS file against the page |
| `css <sel> <prop>` | Computed CSS property value |
| `attrs <sel>` | Element attributes as JSON |
| `console [--clear]` | In-memory console log buffer |
| `network [--clear]` | In-memory network request buffer |
| `cookies` | All cookies as JSON |
| `storage` | `localStorage` + `sessionStorage` as JSON |
| `storage set <k> <v>` | Set `localStorage` key (mutation, but categorized here) |
| `perf` | Navigation timing metrics |

---

### `write-commands.ts` — mutating operations

Commands that change page state. Same function signature as read commands.

| Command | Effect |
|---------|--------|
| `goto <url>` | Navigate, wait for `load`, return HTTP status code |
| `back` / `forward` / `reload` | Browser navigation |
| `click <sel>` | Click element (CSS or `@ref`) |
| `fill <sel> <val>` | Fill input/textarea |
| `select <sel> <val>` | Choose `<select>` option |
| `hover <sel>` | Hover element |
| `type <text>` | Type into currently focused element |
| `press <key>` | Press a keyboard key (`Enter`, `Tab`, `Escape`, etc.) |
| `scroll [sel]` | Scroll element into view, or scroll page to bottom |
| `wait <sel>` | Wait up to 10 s for element to appear |
| `viewport <WxH>` | Set viewport dimensions (e.g., `375x812`) |
| `cookie <str>` | Set a cookie from `name=value[; domain=...]` |
| `header <str>` | Set a persistent request header (`Name:Value`) |
| `useragent <str>` | Set the user-agent string |

All selector arguments go through `BrowserManager.resolveRef()`, so `@ref` and CSS selectors work identically everywhere.

---

### `meta-commands.ts` — server control and visual

Commands that operate on the server or produce binary/rich output rather than text.

**Tabs:**

| Command | Effect |
|---------|--------|
| `tabs` | List all tabs (ID, URL, title) |
| `tab <id>` | Switch active tab |
| `newtab [url]` | Open a new tab, optionally navigate |
| `closetab [id]` | Close tab (default: active tab) |

**Server:**

| Command | Effect |
|---------|--------|
| `status` | Print health, uptime, active tab count, port |
| `url` | Print current page URL |
| `stop` | Graceful server shutdown |
| `restart` | Kill server (CLI auto-restarts on next call) |

**Visual:**

| Command | Effect |
|---------|--------|
| `screenshot [path]` | Save PNG (default: `/tmp/browse-screenshot.png`) |
| `pdf [path]` | Save PDF |
| `responsive [prefix]` | Three screenshots: mobile (375px), tablet (768px), desktop (1280px) |
| `diff <url1> <url2>` | Text diff of page content between two URLs (uses `diff` npm package) |

**Advanced:**

| Command | Effect |
|---------|--------|
| `snapshot [flags]` | Delegates to `snapshot.ts` — kept here because it is a meta-operation on the browser state |
| `chain` | Read JSON array from stdin, execute each command sequentially, return all results |

`chain` is the batch API: `echo '[["goto","https://x.com"],["snapshot","-i"]]' | browse chain`. It avoids per-command CLI startup overhead for multi-step flows.

---

### `buffers.ts` — log capture

Two ring buffers for capturing browser events asynchronously.

```typescript
interface LogEntry    { timestamp: number; level: string; text: string }
interface NetworkEntry { timestamp: number; method: string; url: string;
                        status: number; duration: number; size: number }

const consoleBuffer: LogEntry[]     = [];  // cap: 50,000 entries
const networkBuffer: NetworkEntry[] = [];  // cap: 50,000 entries
```

**Ring buffer logic:** When the buffer reaches capacity, `shift()` removes the oldest entry before pushing the new one. This bounds memory usage without losing recent events.

**Tracking cursors:** `consoleTotalAdded` and `networkTotalAdded` are monotonically increasing counters. The server's 1-second flush timer uses these to detect new entries without re-scanning the entire buffer on each tick.

**Why in-memory first, disk second?** The `console` and `network` commands read directly from the in-memory buffers — no disk I/O in the hot path. Disk files are only for persistence across CLI invocations (e.g., `cat /tmp/browse-console.log` in a shell).

---

## Skill files — structure and patterns

Each skill directory contains only a `SKILL.md` file. Claude Code discovers these files automatically when a skill directory is symlinked into `~/.claude/skills/`.

```
SKILL.md format:
  ---
  name: <skill-name>
  version: <version>
  description: |
    <one-line description for Claude to read>
  allowed-tools:
    - Bash
    - Read
    ...
  ---

  # Workflow instructions (natural language)
```

The frontmatter is consumed by Claude Code for skill discovery and tool permission scoping. The body is the actual workflow prompt.

### Skill inventory

| Skill | Slash command | Cognitive mode | Core job |
|-------|---------------|----------------|----------|
| `browse` | `/browse` | QA engineer | Navigate, interact, screenshot, verify |
| `plan-ceo-review` | `/plan-ceo-review` | Founder / CEO | Challenge premise, find 10-star product |
| `plan-eng-review` | `/plan-eng-review` | Engineering manager | Lock architecture, data flow, diagrams |
| `review` | `/review` | Paranoid staff engineer | Find production bugs that pass CI |
| `ship` | `/ship` | Release engineer | Sync main, run tests, push, open PR |
| `retro` | `/retro` | Engineering manager | Analyze commit history, work patterns |

### Common patterns across skills

**Pre-flight system audit** (`plan-ceo-review`, `plan-eng-review`): Before any planning work, run `git log --oneline -30`, `git diff main --stat`, `git stash list` to understand the current branch state. This grounds the model in facts before it reasons.

**Two-pass review** (`review`, `ship`): A CRITICAL pass (blocking issues) followed by an INFORMATIONAL pass (non-blocking). Critical issues stop the workflow until fixed. This mirrors how safety-critical systems separate "must fix now" from "should fix eventually."

**Scope selection before execution** (`plan-ceo-review`, `plan-eng-review`): Present three options — SCOPE REDUCTION, HOLD SCOPE, BIG CHANGE — and commit fully to whichever the user picks. No silent mode-blending.

**Snapshot + ref pattern** (`browse`): `snapshot -i` first to get element refs, then `click @e3`, `fill @e4`. This is the recommended interaction loop because refs are stable, human-readable, and token-efficient.

---

## Build and test systems

### Build

```
bun build --compile browse/src/cli.ts --outfile browse/dist/browse
```

`bun --compile` bundles all TypeScript, runtime dependencies (Playwright, diff), and the Bun runtime into a single self-contained native binary. The result (~58 MB) can run on any machine with the same OS/arch without Bun installed.

The server (`server.ts`) is **not** compiled separately — the CLI spawns it by calling back into the binary or via `bun run browse/src/server.ts` in dev mode. `resolveServerScript()` in `cli.ts` handles the path resolution for both modes.

### Tests

Tests live in `browse/test/` and use Bun's built-in test runner (`bun test`).

```
browse/test/
├── commands.test.ts   # ~530 lines — integration tests for all 40+ commands
├── snapshot.test.ts   # ~200 lines — snapshot parsing and ref assignment
├── test-server.ts     # ~47 lines  — local HTTP server serving HTML fixtures
└── fixtures/
    ├── basic.html         # Standard page with headings, links, paragraphs
    ├── forms.html         # Form elements (inputs, selects, checkboxes)
    ├── snapshot.html      # Accessibility-rich page for snapshot tests
    ├── responsive.html    # Multi-breakpoint layout
    └── spa.html           # Client-side JavaScript navigation
```

**Test architecture:** Each test file starts a `test-server.ts` instance on a random port, pointing it at the `fixtures/` directory. Tests then create a `BrowserManager` directly (not via the CLI/HTTP layer) to exercise commands at the function level. This keeps tests fast (~3 s total) and avoids HTTP serialization noise.

**Coverage strategy:** `commands.test.ts` covers the command surface area (navigation, extraction, interaction, visual, tabs). `snapshot.test.ts` covers the ref-assignment algorithm and locator construction in isolation. There are no unit tests for `cli.ts` or `server.ts` — the integration tests cover their behavior end-to-end.

### Dev workflow

```bash
bun install                   # install deps + download Playwright Chromium
bun run dev <command>         # run CLI from source (no compile step)
bun test                      # run all tests
bun test browse/test/commands # run command tests only
bun run build                 # compile binary to browse/dist/browse
```

`bun run dev` uses `bun run browse/src/cli.ts` directly — no compile step, instant feedback. The compiled binary is only needed for distribution and the `setup` installer.

---

## Key design decisions

### 1. CLI over MCP

Claude Code has a Bash tool. A CLI that prints plain text to stdout is the simplest possible interface — zero protocol overhead, zero schema tokens, zero connection management. MCP adds ~1,500-2,000 tokens of protocol framing per call. Over a 20-command session, that is 30,000-40,000 wasted tokens.

### 2. Persistent daemon, not per-command launch

Launching Chromium takes ~3 seconds. By keeping it running between commands, subsequent calls take ~100-200 ms. The daemon auto-shuts down after 30 minutes idle, so it is never a resource leak.

### 3. Fail loud on crash

When Chromium crashes, the server calls `process.exit(1)` immediately. No retry, no self-healing, no silent restart. The CLI detects the dead server on the next call and starts a fresh instance. This design surfaces real failures rather than hiding them under recovery logic.

### 4. Accessibility tree, not DOM injection

Using `page.ariaSnapshot()` to power the ref system avoids all DOM mutation. No injected scripts, no `<span data-ref>` attributes, no CSP issues. The accessibility tree is also dramatically more token-efficient than the full DOM.

### 5. Ref map on BrowserManager

Refs (`@e1`, `@e2`, ...) are stored as a `Map<string, Locator>` on the `BrowserManager` instance. This means the map is always in sync with the last snapshot, is garbage-collected automatically when the server restarts, and requires no DOM bookkeeping. The tradeoff is that refs are invalidated on navigation — a deliberate choice to keep the model honest about page state.

### 6. Skills as Markdown, not code

Workflow orchestration is pure LLM instruction. There is no code in the skill directories because the cognitive load of each skill (what to check, when to stop, what to ask the user) is better expressed in natural language than in a state machine. This makes skills easy to read, modify, and reason about.

### 7. Monorepo with a single build artifact

One `package.json`, one `bun install`, one `bun run build`. The skill files are Markdown and need no build step. The setup script only needs to compile the browser binary and symlink the skill directories. This keeps the install story simple: clone, run `./setup`, done.

---

## Data flow: full command lifecycle

```
User types: $ browse click @e3

cli.ts
  │  Read /tmp/browse-server.json → { port: 9400, token: "uuid", pid: 12345 }
  │  POST http://localhost:9400/command
  │  Body: { "command": "click", "args": ["@e3"] }
  │  Header: Authorization: Bearer uuid
  ▼
server.ts
  │  validateAuth() → check Bearer token
  │  Reset idle timer
  │  Dispatch: "click" → handleWriteCommand(bm, "click", ["@e3"])
  ▼
write-commands.ts → handleWriteCommand()
  │  bm.resolveRef("@e3")
  ▼
browser-manager.ts → resolveRef("@e3")
  │  refMap.get("@e3") → Playwright Locator
  ▼
write-commands.ts
  │  locator.click()
  ▼
Playwright → Chromium (CDP)
  │  Click event on element
  ▼
write-commands.ts
  │  Return "Clicked @e3"
  ▼
server.ts
  │  HTTP 200, body: "Clicked @e3"
  ▼
cli.ts
  │  Print "Clicked @e3" to stdout
  ▼
Claude Code reads stdout
```

---

## Security model

| Threat | Mitigation |
|--------|------------|
| Other processes controlling the browser | Bearer token in state file, `chmod 600`, random UUID per session |
| Stale server from a previous session | PID check on startup; dead server → delete state file → restart |
| Multiple workspaces sharing a browser | `CONDUCTOR_PORT` → deterministic port offset → isolated state files |
| Token leakage via world-readable state file | `chmod 600` immediately after write |
| Idle browser holding system resources | Auto-shutdown after 30 min of inactivity |
