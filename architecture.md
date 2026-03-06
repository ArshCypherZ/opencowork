# Open Cowork: Architecture & Planning Specification

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Competitive Context](#competitive-context)
3. [Design Philosophy](#design-philosophy)
4. [Architecture Overview (6 Layers)](#architecture-overview)
5. [Layer 1: Sandbox (Execution Environment)](#layer-1-sandbox)
6. [Layer 2: Agent Core (LLM + Orchestration)](#layer-2-agent-core)
7. [Layer 3: Tool Runtime (Bash + Browser + MCP)](#layer-3-tool-runtime)
8. [Layer 4: Security Manager](#layer-4-security-manager)
9. [Layer 5: Observability & TUI](#layer-5-observability--tui)
10. [Layer 6: Session & Recovery](#layer-6-session--recovery)
11. [Context Engineering Strategy](#context-engineering-strategy)
12. [Configuration System](#configuration-system)
13. [Feature Roadmap](#feature-roadmap)
14. [Technical Decisions & Rationale](#technical-decisions--rationale)
15. [Edge Cases & Solutions](#edge-cases--solutions)
16. [Security Model](#security-model)
17. [Known Limitations](#known-limitations)

---

## Executive Summary

**Open Cowork** is a Linux-native, open-source alternative to [Claude Cowork](https://claude.com/blog/cowork-research-preview) — Anthropic's agentic desktop assistant (launched January 2026, macOS-only, $100-200/month).

Open Cowork provides an autonomous agent capable of:
- Managing files and directories on the local filesystem
- Automating browser interactions via stealth browser backends (Patchright, Pydoll, Camoufox)
- Connecting to external services via MCP (Model Context Protocol)
- Executing multi-step workflows in a sandboxed environment

### Why This Exists

Claude Cowork is:
- **macOS only** (Windows planned, no Linux support announced)
- **Closed-source and proprietary** (locked to Claude models, $100-200/month)
- **Cloud-dependent** (requires Anthropic subscription and connectivity)

Open Cowork fills the gap:
- **Linux native** — built for Linux desktop users and servers
- **Provider agnostic** — works with any LLM via LiteLLM (DeepSeek, Claude, GPT-4, Ollama, local models)
- **Self-hosted** — runs locally, no subscription required (bring your own API key)
- **Open source** — MIT licensed, community-driven

### Key Differentiators vs Claude Cowork

| Feature | Claude Cowork | Open Cowork |
|---|---|---|
| Platform | macOS (Windows planned) | Linux |
| LLM | Claude only | Any (75+ providers) |
| Cost | $100-200/month | Free (BYO API key) |
| Browser | Chrome extension (screenshot-based) | Stealth browser backends (Patchright default, Pydoll/Camoufox for protected sites) |
| Extensibility | Connectors (Anthropic-reviewed) | MCP servers (open ecosystem) |
| Sandboxing | VM isolation | gVisor / bubblewrap (configurable) |
| Source | Closed | MIT open source |

---

## Competitive Context

### Landscape (March 2026)

| Project | Focus | Sandbox | LLM Support | Browser Stealth | Stars |
|---|---|---|---|---|---|
| **Claude Cowork** | Desktop automation (macOS) | VM isolation | Claude only | Screenshot-based | N/A (closed) |
| **OpenHands** | Coding agent | Docker/E2B/Modal | Multi-provider | None | ~50K |
| **OpenCode** | Terminal coding agent | Local (bubblewrap optional) | 75+ providers | None | ~108K |
| **Browser-Use** | Web automation | None | Multi-provider | Custom Chromium fork (C++) | ~78K |
| **Scrapling** | Adaptive web scraping | None | N/A | Built-in Cloudflare bypass | ~22.5K |
| **Pydoll** | Browser automation | None | N/A | CDP-direct, no WebDriver | ~6.6K |
| **Open Interpreter** | General computer use | Optional Docker | Multi-provider | None | ~55K |
| **Open Cowork** | Desktop automation (Linux) | gVisor/bubblewrap | Multi-provider | Pluggable (Patchright/Pydoll/Camoufox) | — |

**Positioning:** Claude Cowork's feature set (file management, browser automation, document generation, office workflows) for Linux, with any LLM, self-hosted, and open source. This is a knowledge-worker automation agent targeting Linux power users and developers.

---

## Design Philosophy

### Core Principles

1. **Ship, Then Architect**
   - Working software over comprehensive documentation
   - The MVP is < 3000 lines of Python
   - Architecture evolves from implementation experience

2. **Safety Without Annoyance**
   - Auto-grant safe operations (read, copy, organize)
   - Confirm only high-risk operations (delete, encrypt, upload)
   - Use trash bins instead of permanent deletion
   - Sandbox at the kernel level, not just application level

3. **Observability is Non-Negotiable**
   - Users must see what the agent is doing in real-time via TUI
   - Every action must be auditable
   - Provide pause/resume/cancel controls
   - Display running cost per session

4. **Stealth Browser-First, Vision as Fallback**

   The key question for browser automation is not "DOM vs screenshots" — it is how detectable the browser engine is. Modern anti-bot systems (Cloudflare, DataDome, Akamai, Kasada, PerimeterX) detect vanilla Playwright/Selenium in under 50ms via WebDriver flags, CDP leaks, TLS fingerprints, and behavioral analysis.

   Our strategy uses a tiered approach with pluggable browser backends:
   - **Foundation:** Patchright (patched Playwright) — fixes major detection leaks, drop-in replacement
   - **Growth:** Pydoll (CDP-direct, no WebDriver) and Camoufox (C++ Firefox patches) for protected sites
   - **Fallback:** Vision models (screenshot + VLM) only when all browser backends fail

   Realistic expectations:
   - ~70% of sites work with Patchright (patched Playwright)
   - ~20% more work with CDP-direct (Pydoll) or C++ patches (Camoufox)
   - ~5% need vision fallback (canvas elements, image CAPTCHAs)
   - ~5% will not work reliably (banking, heavy anti-bot) — documented as limitation

5. **Context is a Budget, Not a Dump**
   - Every token sent to the LLM costs money and attention
   - Tool results are truncated, not passed raw
   - Conversation history is compacted with semantic preservation
   - Structured state lives outside chat history

---

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  User (TUI)                      │
│            Layer 5: Observability & TUI          │
├─────────────────────────────────────────────────┤
│            Layer 6: Session & Recovery           │
├─────────────────────────────────────────────────┤
│            Layer 2: Agent Core                   │
│         (LLM Provider + ReAct Loop)              │
├─────────────────────────────────────────────────┤
│            Layer 3: Tool Runtime                 │
│   (Bash + Stealth Browser + MCP Servers)         │
├─────────────────────────────────────────────────┤
│            Layer 4: Security Manager             │
│    (Permissions, Path Validation, Network)        │
├─────────────────────────────────────────────────┤
│            Layer 1: Sandbox                      │
│         (bubblewrap / gVisor)                    │
└─────────────────────────────────────────────────┘
```

**6 layers. No gaps.**

- **Layer 1 (Sandbox):** Kernel-level isolation for untrusted code execution
- **Layer 2 (Agent Core):** LLM provider abstraction + ReAct execution loop + cost tracking
- **Layer 3 (Tool Runtime):** Bash REPL, stealth browser (pluggable backends), MCP tool servers
- **Layer 4 (Security):** Permission model, path validation, network policy, secrets exclusion, MCP result sandboxing
- **Layer 5 (Observability):** TUI interface, progress tracking, audit logging, cost display
- **Layer 6 (Session):** Lifecycle management, checkpointing, graceful shutdown

---

## Layer 1: Sandbox

**Purpose:** Kernel-level isolation for executing untrusted LLM-generated code.

### Why Not Just Docker?

Docker containers share the host kernel. LLM-generated code is unpredictable and cannot be audited ahead of execution. CVE-2019-5736 demonstrated container escape is real. The 2026 industry consensus: containers alone are insufficient for agent sandboxing.

### Sandbox Tiers

**Tier 1 — bubblewrap (Foundation, zero dependencies)**

Linux namespace isolation without Docker daemon. Startup in <10ms. Installed via `apt install bubblewrap`.

- Provides namespace isolation: mount, PID, network, user
- Uses explicit **mount allowlist** (never bind-mounts host root):
  - `/usr`, `/bin`, `/sbin`, `/lib`, `/lib64` — read-only (executables and libraries)
  - `/etc/resolv.conf`, `/etc/ssl`, `/etc/ca-certificates`, `/etc/ld.so.cache` — read-only (DNS and TLS)
  - `/workspace` — read-write (user data, bound from host workspace)
  - `/trash` — read-write (deleted files, bound from `~/.cowork/trash`)
  - `/tmp`, `/home`, `/run` — tmpfs (isolated, ephemeral)
  - `/proc`, `/dev` — minimal sandbox-provided mounts
- Network isolation via `--unshare-net` by default; network allowed only when explicitly configured
- PID namespace isolation via `--unshare-pid`
- Process dies with parent via `--die-with-parent`
- No cgroup resource limits (use `ulimit` as a basic alternative)
- Trade-off: lighter isolation than gVisor, but zero friction to set up

**Tier 2 — gVisor (Growth, recommended for production)**

User-space kernel that intercepts syscalls via `runsc` OCI runtime. Startup ~50ms.

- Dramatically reduced kernel attack surface — syscalls are handled by gVisor's Sentry, not the host kernel
- Runs as a Docker runtime (`--runtime=runsc`), providing cgroups + namespace + syscall interception
- Container configuration: read-only root, 4GB memory limit, 2 CPU limit, 256 PID limit, `no-new-privileges`, 512MB tmpfs for `/tmp`
- Requires Docker daemon + gVisor `runsc` runtime installed
- Trade-off: some rare syscall incompatibilities (uncommon for Python/Node workloads)

**Future options (Scale tier):** Docker with gVisor runtime for enterprises with existing Docker infrastructure. Firecracker microVMs for multi-tenant SaaS deployments requiring hardware-level KVM isolation.

### Container Image (Docker/gVisor Tiers Only)

A single base image (~1.5GB) pre-installs common tools:

- **System:** git, curl, wget, jq, ripgrep, tree, file, unzip
- **Office:** libreoffice-calc, libreoffice-writer, ghostscript, imagemagick
- **Browser:** Chromium via Patchright/Playwright install
- **Python packages:** pandas, numpy, openpyxl, python-docx, beautifulsoup4, requests, httpx, Pillow, pymupdf, litellm, mcp, patchright

For the bubblewrap tier (no Docker), the agent runs host-installed tools inside namespace isolation. Required host packages: `python3`, `bwrap`, `git`. Optional: `patchright`, MCP server binaries.

---

## Layer 2: Agent Core

**Purpose:** LLM provider abstraction, execution loop, and cost tracking.

### LLM Provider

Provider-agnostic LLM interface using LiteLLM as a Python library (not proxy server). Zero operational overhead compared to running a proxy service.

**Configuration defaults:**
- Primary model: `deepseek/deepseek-chat`
- Fallback model: `anthropic/claude-sonnet-4`
- Vision model: `anthropic/claude-sonnet-4`
- Max retries: 2
- Timeout: 60s per call
- Max cost per session: $5.00 (configurable)

**Call flow:**
1. Attempt primary model
2. On rate limit: exponential backoff with jitter (`2^attempt + random(0,1)` seconds), then retry
3. On transient error: backoff, then retry with fallback model
4. On authentication error: fail immediately (no retry — bad key won't fix itself)
5. After max retries exhausted: raise error, checkpoint session state

### Cost Tracker

Tracks cumulative session cost using LiteLLM's built-in cost calculator (`litellm.completion_cost()`).

- Records cost, model, and token count for every LLM call
- Checks against configurable `max_cost_per_session_usd` limit after every call
- When budget is exhausted: emit cost_limit event, halt agent loop, notify user via TUI
- Falls back to zero-cost estimate when LiteLLM's cost calculator fails for unknown models

### Agent Loop (ReAct Pattern)

The core execution pattern is ReAct (Reason + Act). The LLM receives context, decides what tool to call, observes the result, and repeats until the task is done. Simpler and more reliable than plan-and-execute for the majority of tasks — the LLM adapts to each result as it arrives rather than predicting the entire path upfront.

**Loop steps:**
1. Add user prompt to message history
2. Build messages: system prompt (with workspace state injected) + compacted history + recent messages
3. Send messages + tool definitions to LLM
4. Record cost from response
5. If LLM returns text (no tool calls) → task complete, return result
6. If LLM returns tool calls → execute **all** of them (LLMs emit parallel calls):
   a. Parse tool call arguments (JSON). On malformed JSON, return error to LLM so it self-corrects
   b. Check permission via SecurityManager. If denied, return "Permission denied" as tool result
   c. Execute tool via ToolRuntime
   d. Update ContextState with result (files created/modified, errors, current URL)
   e. Emit action event to TUI (tool name, truncated args, truncated result)
   f. Add tool result to message history (truncated to 4,000 chars)
7. Check cost limit — halt if exceeded
8. Check turn limit (default 25) — halt if exceeded
9. Compact message history if it exceeds 40 messages (keep last 12 intact, summarize older)
10. Repeat from step 2

**System prompt** instructs the agent on available tools, workspace scoping (`/workspace`), safety rules (trash instead of rm, confirm destructive ops, read before modify, verify results), and the observe-act-verify workflow.

### Plan-and-Execute (Optimization Layer)

For batch-style tasks where the full execution path is predictable (e.g., "rename all .jpeg files to .jpg"), a plan-and-execute layer can reduce LLM calls by generating the plan upfront and executing steps directly. This is an optimization on top of the ReAct loop, not a replacement. It generates a structured plan via LLM, executes deterministic steps without further LLM calls, and falls back to ReAct for any step that fails or requires judgment.

---

## Layer 3: Tool Runtime

**Purpose:** Tool definitions, execution, and MCP integration.

### Tool Registry

Central router that dispatches tool calls to the appropriate handler based on function name prefix:
- `run_bash` → BashTool
- `browser_*` → BrowserTool (via pluggable backend)
- `mcp_*` → MCPManager (routes `mcp_servername_toolname` to correct server)

Unknown tool names return an error to the LLM so it can self-correct.

### Bash Tool

Shell command execution with defense-in-depth safety intercepts. Commands are always parsed via `shlex.split()` and executed as subprocess argument lists — `shell=True` is never used anywhere.

**Safety intercepts (defense-in-depth, not security boundary):**
- **Blocked commands:** `dd`, `mkfs`, `fdisk`, `mount`, `umount`, `chown`, `su`, `sudo` — rejected immediately
- **Trash intercept:** `rm` is intercepted and targets are moved to `/trash` with a unique suffix instead of permanent deletion
- **Path validation:** Any absolute path in command arguments is checked against allowed roots. Paths outside workspace/tmp/trash are rejected or require user permission
- **Output truncation:** stdout over 10,000 chars is truncated to first 5,000 + last 2,000 chars to protect context window

These intercepts are bypassable (the LLM could invoke Python, Perl, or `find -delete`). The real security boundary is the kernel-level sandbox (Layer 1), which restricts filesystem access regardless of how commands are invoked.

### Browser Tool

#### The Anti-Bot Problem (2026)

Modern anti-bot systems detect vanilla Playwright/Selenium through multiple vectors:

| Detection Method | What It Checks |
|---|---|
| **WebDriver flag** | `navigator.webdriver === true` (set by Playwright/Selenium by default) |
| **CDP leaks** | `Runtime.enable`, `Console.enable` calls leave detectable traces |
| **Command flags** | `--enable-automation`, `--disable-extensions` reveal automation |
| **TLS fingerprinting** | Headless browsers have distinct TLS handshake signatures |
| **Behavioral analysis** | Instant clicks, no mouse movement, no scroll patterns |
| **Canvas/WebGL fingerprinting** | Headless browsers produce identifiable rendering artifacts |
| **Prototype chain checks** | JS patching (stealth plugins) leaves detectable inconsistencies |

`playwright-stealth` (JS-level patching) explicitly states: *"Don't expect this to bypass anything but the simplest of bot detection methods."* Anti-bot vendors (Cloudflare, DataDome, Akamai) can currently detect more than they block — they set conservative thresholds to avoid false positives. As AI agent traffic grows, those thresholds will tighten.

#### Browser Backend Abstraction

The browser layer defines a **BrowserBackend interface** that all backends implement. This allows swapping the underlying engine without changing the agent's tool definitions or the LLM's interaction pattern.

All backends expose the same tool surface. The agent and LLM are unaware of which backend is active — the choice is a configuration decision.

#### Stealth Tiers

**Tier 1 — Patchright (Foundation, default)**

Patchright is a patched version of Playwright that fixes the major detection leaks. It is a drop-in replacement — same API, just a different import. Apache-2.0 license. ~1.15K GitHub stars.

Key patches:
- Removes `Runtime.enable` leak (executes JS in isolated ExecutionContexts instead)
- Removes `Console.enable` leak (disables Console API)
- Removes automation command flags (`--enable-automation`, `--disable-extensions`, etc.)
- Adds `--disable-blink-features=AutomationControlled` to prevent `navigator.webdriver` detection

This handles the majority of lightly-to-moderately protected websites. It is the default because it requires zero architectural change from vanilla Playwright.

**Tier 2 — Pydoll (Growth, for protected sites)**

Pydoll is a Python library that connects directly to Chrome DevTools Protocol over WebSocket. No WebDriver binary is involved at all — `navigator.webdriver` is never set in the first place, not patched after the fact. MIT license. ~6.6K GitHub stars.

Key capabilities:
- Direct CDP connection — no WebDriver binary, no WebDriver protocol, no detectable driver artifacts
- Human-like interaction engine: variable keystroke timing (30-120ms), realistic typos (~2% error rate), physics-based scrolling
- Browser fingerprint control: granular modification of Chrome preferences
- Fully async-native and type-checked with mypy — aligns with the project's async architecture
- Cookie and session persistence between runs

Use when Patchright gets blocked: sites checking for CDP artifacts that Patchright doesn't patch, or sites using behavioral analysis that detects instant/robotic interactions.

**Tier 3 — Camoufox (Growth, for heavily protected sites)**

Camoufox is a custom Firefox fork with fingerprint spoofing compiled at the C++ source level. Not JS injection, not runtime patching — the browser binary itself produces genuine-looking fingerprints. MPL-2.0 license. ~1.5K GitHub stars.

Key capabilities:
- C++ level patches: Canvas, WebGL, AudioContext, fonts, GPU, screen geometry all spoofed at source
- Firefox-based — less fingerprinted than Chrome (the vast majority of bots use Chrome/Chromium)
- Automatic fingerprint rotation
- Has a Python API and existing MCP server integration

Use when Chromium-based backends (Patchright, Pydoll) are detected. Firefox's smaller automation footprint in the wild makes it inherently less suspicious.

**Tier 4 — Vision Fallback (Growth)**

Screenshot + vision-capable LLM for sites where DOM access fails entirely. Expensive and slow — a single interaction requires a vision model API call (~$0.01-0.05 per screenshot analysis). Reserved for canvas-rendered content, image-based CAPTCHAs, and sites that defeat all browser backends.

#### Browser Tool Surface

Eight browser actions exposed to the LLM:

| Tool | Description | Key Behavior |
|---|---|---|
| `browser_goto` | Navigate to a URL | Check domain against allowlist. Wait strategy: `domcontentloaded` with 30s timeout (SPAs never reach `networkidle`). |
| `browser_click` | Click an element (CSS selector) | Configurable wait timeout (default 10s). With Pydoll backend, uses human-like mouse movement. |
| `browser_type` | Type into an input field | Uses `fill()` for Patchright. With Pydoll backend, uses variable keystroke timing. |
| `browser_read` | Extract visible text from page/element | Uses `inner_text()` not `content()` (~90% smaller than raw HTML). Truncated to 8,000 chars. |
| `browser_wait` | Wait for a selector or condition | Wait for element to appear, disappear, or become visible. Configurable timeout. |
| `browser_screenshot` | Capture page screenshot | Returns image for debugging or vision fallback. Can target full page or specific element. |
| `browser_scroll` | Scroll the page | Scroll by amount or to element. Essential for lazy-loaded content and infinite scroll pages. |
| `browser_js` | Execute arbitrary JavaScript | Escape hatch for complex interactions: dropdowns, file uploads, iframes, popups, authentication flows. Returns serialized result. |

#### Browser Design Decisions

- **Route interception for domain enforcement:** Playwright/Patchright route handler intercepts all `document`-type requests (not just `goto` calls) and checks against the domain allowlist. This catches redirects and click-triggered navigations.
- **Browser profile persistence:** Save and restore cookies, localStorage, and browser state between sessions. Fresh profiles are a detection signal.
- **Memory management:** `restart()` method kills and re-launches the browser to clear Chromium memory leaks. Trigger between unrelated browsing tasks or when memory usage is concerning.
- **Domain policy for browsing is separate from sandbox network policy:** The sandbox network (`--unshare-net`) controls bash tool network access. The browser has its own domain allowlist that can be more permissive (user may want to browse the web while keeping bash network-restricted). See [Configuration System](#configuration-system).

### MCP Integration

MCP (Model Context Protocol) is the standard for connecting AI agents to external tools. Integration provides access to the entire ecosystem of pre-built MCP servers.

**How it works:**
1. MCP servers are configured in `~/.cowork/config.yaml` with their command and optional environment variables
2. On session start, the agent connects to configured servers via stdio transport
3. Each server's tools are discovered automatically via `list_tools()`
4. Tool definitions are injected into the LLM's tool list, namespaced as `mcp_{servername}_{toolname}` with descriptions prefixed by `[MCP:servername]`
5. When the LLM calls an MCP tool, the request is routed to the correct server

**Result handling (critical for context management):**
MCP servers can return enormous results (full database tables, large file contents). Research shows up to 236x token inflation from naive MCP integration. All MCP tool results are truncated to 5,000 chars. Large results should be written to a file in `/workspace` and the file path returned instead.

**Recommended MCP tools:**
- `@modelcontextprotocol/server-filesystem` — filesystem operations
- `@modelcontextprotocol/server-github` — GitHub integration
- **Scrapling** (22.5K stars, BSD-3) — adaptive web scraping with built-in Cloudflare bypass and an intelligent parser that auto-relocates elements when page structure changes. Recommended as an MCP tool for scraping-specific tasks rather than building scraping into core.

---

## Layer 4: Security Manager

**Purpose:** Permission management and safety enforcement at the application level. This is defense-in-depth ON TOP of kernel-level sandbox isolation (Layer 1).

### Permission Model

Intent-based permission system with three tiers:

| Operation Type | Examples | Permission |
|---|---|---|
| **Read / navigate** | `browser_read`, `browser_goto`, `run_bash` with `ls`/`cat` | Auto-granted |
| **Write within workspace** | `run_bash` with `mkdir`/`cp`/`mv` in `/workspace` | Auto-granted |
| **Risky / external** | Paths outside workspace, network access, unknown MCP tools | Prompt user via TUI |
| **Blocked** | `dd`, `mkfs`, `fdisk`, `mount`, `su`, `sudo` | Always denied |

### Path Validation

All file paths are resolved (following symlinks) and checked against allowed roots:
- `workspace` — the user's working directory
- `/tmp` — temporary files
- `~/.cowork/trash` — deleted files
- Session grants — additional paths the user explicitly allows during a session

Paths outside allowed roots trigger a permission prompt. The sandbox mount namespace provides the hard enforcement — path validation is an additional safety layer.

### Network Policy

Two separate network policies:

1. **Sandbox network (bash tool):** Default deny via `--unshare-net`. No network access from bash commands unless explicitly configured. When enabled, restricted to a domain allowlist (default: pypi.org, github.com, npmjs.org, and their CDN subdomains).

2. **Browser network:** Separate domain allowlist, can be more permissive. Enforced via route interception on all document-level navigations. Users configure allowed browser domains in config (can be set to allow all for open browsing tasks). Sub-resource requests (CDN, fonts, analytics) are not blocked to avoid breaking page rendering.

### Trash-for-Delete

All `rm` commands are intercepted at the bash tool level and converted to moves into `~/.cowork/trash/` with a unique suffix. Trash is cleaned up on session start: files older than 30 days are permanently deleted. Trash size is reported via the `--status` command.

### Secrets Exclusion

A `.coworkignore` file (gitignore syntax) in the workspace root specifies files the agent should not read or include in context. Default patterns:
- `.env`, `.env.*`
- `*.pem`, `*.key`, `*_rsa`, `*_ed25519`
- `credentials.json`, `service-account.json`
- `*.secret`, `*.secrets`

The SecurityManager checks this file before allowing read operations. This is a safety measure, not a security boundary — the sandbox is the hard boundary.

---

## Layer 5: Observability & TUI

**Purpose:** Real-time user interface, audit logging, cost display, and structured logging.

### TUI Layout

```
┌──────────────────────────────────────────────────┐
│  Open Cowork                     [Pause] [Cancel] │
├──────────────────────────────────────────────────┤
│                                                   │
│  Agent Output (streaming)                         │
│  ─────────────────────────────                    │
│  > Planning task...                               │
│  > Step 1/5: mkdir -p output/                     │
│  > Step 2/5: Downloading page...                  │
│  > Step 3/5: Extracting data with pandas          │
│    [████████░░░░░░░░░░] 45%                       │
│                                                   │
│  Cost: $0.03 | Turns: 5 | Model: deepseek-chat   │
├──────────────────────────────────────────────────┤
│  [Permission Request]                             │
│  Agent wants to delete 15 files in /workspace/tmp │
│  Files will be moved to trash (recoverable).      │
│           [Allow]  [Deny]  [Allow All Session]    │
├──────────────────────────────────────────────────┤
│  > Enter task:                                    │
│  ________________________________________________ │
└──────────────────────────────────────────────────┘
```

### EventBus

Async event bus connecting the agent loop to the TUI. Provides:

- **Pub/sub model:** Multiple subscribers (TUI, audit logger) receive events via async queues
- **Event types:** `action` (tool executed), `progress` (step N/M), `permission_request`, `cost_limit`, `error`
- **Permission flow:** Agent emits a permission request event with a request ID. TUI displays prompt. When user clicks Allow/Deny, TUI calls `resolve_permission(request_id, granted)` which resolves the waiting async Future. Default: deny on 60-second timeout.

### Audit Logger

Append-only SQLite audit log via `aiosqlite` (non-blocking in async context):

- Records every tool call: timestamp, session_id, tool, command, result (truncated to 5,000 chars), exit_code, duration_ms, cost_usd
- No UPDATE/DELETE operations — insert only
- WAL journal mode for concurrent read performance
- Stored at `~/.cowork/audit.db`
- Queryable via `--history` command (filter by session, tool, date range)

Application logs use structured JSON format via Python's standard `logging` module for machine-parseable output. Includes tool name, duration, and cost fields on relevant log entries.

---

## Layer 6: Session & Recovery

**Purpose:** Session lifecycle, checkpointing, and graceful shutdown.

### Session Lifecycle

Each session has: a unique ID, workspace path, creation time, last activity time, and status (active/paused/completed/interrupted).

### Git-Based Checkpointing

Workspace state snapshots use a separate git directory (`~/.cowork/checkpoints/<session_id>`) to avoid polluting any existing git repo in the user's workspace.

**Checkpoint behavior:**
1. Initialize: `git init` in checkpoint dir + initial commit of workspace state on session start
2. Save: `git add -A` + `git commit` with descriptive message. O(changed files), not O(all files). Free deduplication via git's content-addressable storage.
3. Restore: `git checkout <commit_hash> -- .` to roll back workspace to a previous state
4. Diff: `git diff --stat <commit> HEAD` to show what changed since a checkpoint

**When checkpoints are saved:**
- Session start (baseline)
- Before risky operations (user-confirmed destructive actions)
- On session pause or interrupt
- On graceful shutdown

### Graceful Shutdown

Registers cleanup handlers for SIGINT, SIGTERM, and `atexit`. Cleanup order:
1. Save checkpoint with "interrupted" / "shutdown" message
2. Close browser (kill Chromium process)
3. Close all MCP server connections
4. Close audit database connection

If the event loop is running, cleanup runs as an async task. If not (atexit), a synchronous checkpoint save is performed as a best-effort fallback.

### Session Resume

The `--resume` flag restores the most recent session:
1. Load session metadata
2. Restore workspace from last checkpoint
3. Reload message history (last 12 messages + compacted summary)
4. Reconnect MCP servers
5. Display summary of where the session left off

---

## Context Engineering Strategy

**Purpose:** Manage the LLM's context window as a finite budget.

Reference: [Anthropic's "Effective Context Engineering for AI Agents" (Sep 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).

The core principle: context is working memory. Fill it with the right information at each step, not everything accumulated so far.

### Token Budget

The context window is partitioned into reserved zones:

| Zone | Budget | Purpose |
|---|---|---|
| System prompt | ~1,500 tokens | Agent instructions + workspace state |
| Tool definitions | ~2,000 tokens | All tool schemas (bash + browser + MCP) |
| Reserved for response | ~4,000 tokens | LLM's output space |
| Available for history | Remaining (~92,500 of 100K) | Messages, tool results, compacted summaries |

Token counting uses LiteLLM's model-aware tokenizer (`litellm.token_counter()`). Falls back to a conservative heuristic (~3.5 chars per token) for models LiteLLM doesn't support.

### Context State (Lives Outside Chat History)

A structured state object tracks key facts across the entire session, surviving message compaction cycles:

- **Files created / modified / deleted** (last 10 of each)
- **Current browser URL**
- **Current step count**
- **Recent errors** (last 3, with truncated details)
- **Key facts** (last 5 — extracted from tool results, e.g., "API returned 404", "file has 1,523 rows")
- **Original goal** (preserved verbatim from first user message)

This state is injected into the system prompt as a `[WORKSPACE STATE]` block on every LLM call, ensuring the agent always knows what has been done even after older messages are compacted away.

### Message Compaction

When message count exceeds 40, compaction is triggered:

1. **Split:** Keep the last 12 messages intact. Older messages are candidates for compaction.
2. **Summarize:** Build a semantic summary from older messages preserving:
   - Which tools were called and with what key arguments (first 120 chars)
   - Abbreviated results (first 150 chars)
   - All ContextState data (files, errors, URLs, key facts)
3. **Inject:** The compacted summary is prepended to the conversation as a `[CONTEXT SUMMARY]` user message.
4. **Discard:** Original older messages are dropped.

Tool results are truncated at insertion time (max 4,000 chars each) to limit accumulation before compaction is needed. Custom JSON encoder handles non-serializable types (Path, datetime, bytes, set).

---

## Configuration System

**Purpose:** User-configurable settings for models, sandbox, browser, security, and cost limits.

### Environment Variables (API Keys)

Sensitive values are set via environment variables only — never stored in config files:

| Variable | Description |
|---|---|
| `OPENCOWORK_API_KEY` | Default LLM API key (or use provider-specific: `DEEPSEEK_API_KEY`, `ANTHROPIC_API_KEY`, etc.) |
| `GITHUB_TOKEN` | For GitHub MCP server (optional) |

### Config File (`~/.cowork/config.yaml`)

```yaml
# LLM configuration
model:
  primary: deepseek/deepseek-chat
  fallback: anthropic/claude-sonnet-4
  vision: anthropic/claude-sonnet-4
  timeout: 60
  max_retries: 2

# Cost limits
cost:
  max_per_session_usd: 5.00

# Sandbox tier: bubblewrap | gvisor
sandbox:
  tier: bubblewrap
  allow_network: false   # for bash tool

# Browser configuration
browser:
  backend: patchright     # patchright | pydoll | camoufox
  headless: true
  allowed_domains:        # for browser navigation (separate from sandbox network)
    - "*"                 # default: allow all (override to restrict)
  persist_profile: true   # save cookies/localStorage between sessions

# Workspace
workspace:
  path: ~/workspace       # default working directory
  ignore_patterns:        # additional .coworkignore patterns
    - "node_modules/"
    - ".git/"

# MCP servers
mcp:
  servers:
    filesystem:
      command: ["npx", "@modelcontextprotocol/server-filesystem", "/workspace"]
    github:
      command: ["npx", "@modelcontextprotocol/server-github"]
      env:
        GITHUB_TOKEN: "${GITHUB_TOKEN}"
```

Configuration is loaded in priority order: CLI flags > environment variables > config file > defaults.

---

## Feature Roadmap

Features are organized by priority tier. Each tier builds on the one above it.

### Foundation

The minimum viable product. A working CLI agent that can execute tasks.

| Layer | Scope |
|---|---|
| Sandbox | bubblewrap namespace isolation |
| Agent Core | LiteLLM library + ReAct loop + cost tracking |
| Tools | Bash (with safety intercepts) + Patchright browser (8 actions) + MCP client |
| Security | Path validation, trash delete, secrets exclusion, default network deny |
| Observability | Rich CLI streaming, structured logging, cost display |
| Session | Single session, graceful shutdown, config system (env vars + YAML) |

**Foundation delivers:**
- `pip install opencowork` → set API key → run
- Execute tasks in bubblewrap sandbox
- Provider switching via LiteLLM (any model)
- File operations with safety intercepts
- Stealth browser automation via Patchright (handles ~70% of websites)
- MCP tool ecosystem access
- Real-time streaming output
- Cost tracking per session
- Configuration via `~/.cowork/config.yaml`
- Graceful shutdown with state preservation

### Growth

Expanded capabilities and richer user experience.

| Addition | Description |
|---|---|
| Pydoll browser backend | CDP-direct automation with human-like interactions for sites Patchright can't handle |
| Camoufox browser backend | C++ Firefox patches for heavily protected sites |
| Vision fallback | Screenshot + VLM for canvas/anti-bot/CAPTCHA sites |
| Textual TUI | Permission prompts, progress tracking, audit viewer |
| Git checkpoints | Workspace state snapshots with separate git directory |
| Session resume | `--resume` flag for interrupted tasks |
| gVisor sandbox | Production-grade syscall interception |
| Plan-and-execute | Optimization layer for batch-style tasks |
| SQLite audit log | Persistent, queryable action history via aiosqlite |
| Browser profile persistence | Save/restore cookies and localStorage between sessions |

### Scale

Enterprise and advanced features.

| Addition | Description |
|---|---|
| Web dashboard UI | Real-time progress, session management, log viewer |
| Multi-session | Concurrent sessions with advisory file locking |
| Firecracker | Hardware-level microVM isolation for multi-tenant deployments |
| Enterprise | RBAC, SSO, compliance reporting |
| Skills system | Document generation, data processing templates |
| Connectors | Google Drive, Slack, Gmail integration via MCP |

---

## Technical Decisions & Rationale

### Why bubblewrap for Foundation?

| Factor | bubblewrap | Docker |
|---|---|---|
| Install | `apt install bubblewrap` | Docker daemon + service |
| Startup | <10ms | 200ms-2s |
| Dependencies | None (Linux kernel) | dockerd, containerd, runc |
| User experience | Transparent (no container) | Requires image pull, volume mounts |
| Security | Namespace isolation | Namespace isolation (shared kernel) |
| Resource limits | ulimit (basic) | cgroups (better) |

bubblewrap for Foundation (zero friction), Docker+gVisor for Growth/Scale (resource control + stronger isolation).

### Why LiteLLM as Library, Not Proxy?

Running LiteLLM as a proxy server adds an extra service to manage, network latency on every LLM call, and a single point of failure. Using `litellm` as a Python library gives the same provider abstraction with zero operational overhead.

**Known trade-off:** LiteLLM has documented reliability issues at scale (token counting inaccuracies, cold start latency, retry loop edge cases). For Foundation-tier usage (single user, moderate volume), these are acceptable. For Scale tier, consider direct provider SDK calls as a fallback path.

### Why Async Throughout?

LLM API calls take 2-30 seconds. Browser page loads take 1-30 seconds. Synchronous execution means no streaming output, no heartbeats, no cancel/interrupt capability, no parallel tool execution. Full async from day 1: `asyncio` event loop, async browser APIs, `aiosqlite`.

### Why MCP in Foundation?

MCP is the standard for agent-tool integration. Without MCP, every tool must be hand-coded and users cannot add their own tools. With MCP in Foundation:
- Filesystem, git, GitHub tools available immediately via `npx`
- Users can connect any MCP server (including Scrapling for adaptive web scraping)
- Tool definitions generated automatically from MCP schemas
- ~200 lines of integration code for massive extensibility

**Known trade-off:** MCP tool results can be large and noisy, causing context window saturation and up to 236x token inflation (per 2025 research). Mitigation: truncate all MCP results to 5,000 chars, prefix tool descriptions with `[MCP:server]` to help the LLM attribute sources.

### Why Git for Checkpoints?

Computing file hashes on every checkpoint is O(n) over all files, painfully slow for large workspaces, and MD5 is cryptographically broken. Git provides O(changed files) incremental tracking, built-in diff/restore/history, free deduplication, and is already installed on every dev machine.

The checkpoint git directory is kept separate from the workspace (`~/.cowork/checkpoints/<session_id>`) so it does not conflict with the user's own git repo.

### Why Patchright over Vanilla Playwright?

Vanilla Playwright is detectable in under 50ms by modern anti-bot systems. The key leaks:
- `Runtime.enable` CDP call leaves traces detectable by page JavaScript
- `Console.enable` CDP call is a known automation indicator
- Default command flags (`--enable-automation`) explicitly signal automation
- `navigator.webdriver` is set to `true`

Patchright fixes all of these at the library level. It is a drop-in replacement (same API, change one import line), is Apache-2.0 licensed, and requires zero architectural changes. There is no reason to use vanilla Playwright when Patchright exists.

Trade-off: Patchright only patches Chromium. Firefox and WebKit are not supported. For Firefox-based stealth, Camoufox is the Growth-tier option.

### Why ReAct over Plan-and-Execute as the Core Loop?

Plan-and-execute assumes the full execution path can be determined upfront. In practice, real tasks involve dynamic content (web pages change), unpredictable errors (disk full, permissions), and branching logic (different actions based on file contents). The LLM cannot reliably predict these at planning time.

ReAct handles this naturally: observe result, decide next action, repeat. The plan-and-execute optimization layer is available for tasks where upfront planning is clearly beneficial (batch file operations, known-path workflows), but it is not the default execution pattern.

---

## Edge Cases & Solutions

### EC-1: LLM Returns Malformed JSON

**Problem:** Tool call arguments contain invalid JSON.
**Solution:** Wrap argument parsing in try/except. On failure, return an error message to the LLM as a tool result so it can self-correct. The ReAct loop naturally handles this — the LLM sees the error and retries with valid JSON.

### EC-2: Browser Page Never Reaches networkidle

**Problem:** SPAs with websockets, analytics, or live updates never reach `networkidle`.
**Solution:** Use `domcontentloaded` with a 30s timeout. This fires when HTML is parsed, which is sufficient for most interaction tasks.

### EC-3: shlex.split Fails on Malformed Commands

**Problem:** LLM generates command with unbalanced quotes. `shlex.split` raises `ValueError`.
**Solution:** Catch the exception and return a clean error to the LLM so it can self-correct.

### EC-4: Command Injection via shell=True

**Problem:** `subprocess.run(command, shell=True)` allows arbitrary shell injection.
**Solution:** `shell=True` is never used. All commands are parsed to lists via `shlex.split` and executed as `subprocess.run(cmd_list, shell=False)`. The sandbox also receives command lists.

### EC-5: browser_read Returns Megabytes

**Problem:** Raw HTML can be 500KB-5MB. Appending this to LLM messages destroys the context budget.
**Solution:** Use `inner_text('body')` instead of `content()` (visible text only, ~90% smaller). Truncate to 8,000 chars. If user needs full content, save to file and return the file path.

### EC-6: Context Window Exhaustion on Long Tasks

**Problem:** After 10-15 tool calls, message history exceeds the context window.
**Solution:** MessageManager with: tool result truncation (4,000 chars per result), sliding window (keep last 12 messages, summarize older), semantic-preserving compaction (action log + ContextState data), and token budget tracking (halt when exhausted).

### EC-7: Docker Daemon Not Running

**Problem:** gVisor/Docker sandbox fails because Docker daemon is not installed.
**Solution:** bubblewrap is the default sandbox (no daemon required). Docker/gVisor is opt-in. Startup detects available sandbox tiers and selects the best one.

### EC-8: Concurrent File Access (Multi-Session)

**Problem:** Two sessions modify the same files simultaneously.
**Solution:** Not in Foundation (single session only). For Growth: advisory file locks via `fcntl.flock()` or SQLite-based lock table.

### EC-9: LLM API Key Expires Mid-Session

**Problem:** API calls start failing partway through a task.
**Solution:** Catch `AuthenticationError` specifically (not retried). Save checkpoint. Prompt user to update API key. Resume from checkpoint.

### EC-10: User Workspace is on NFS/Network Mount

**Problem:** bubblewrap `--bind` may not work with network mounts. File locking semantics differ.
**Solution:** Document as known limitation. Detect mount type at startup and warn. Recommend copying to local disk.

### EC-11: Browser Chromium Missing Dependencies

**Problem:** Minimal Linux installs lack Chromium dependencies (libgbm, libnss, etc.).
**Solution:** `patchright install chromium` (or `playwright install --with-deps chromium`) in setup. For Docker tiers, dependencies are pre-installed in the container image. For bubblewrap tier, bind-mount host's Chromium.

### EC-12: Trash Directory Fills Disk

**Problem:** Files moved to trash accumulate over time.
**Solution:** Run trash cleanup on session start: delete files older than 30 days. Track trash size in `--status` command.

### EC-13: Clicks Navigate to Disallowed Domains

**Problem:** The LLM clicks a link that redirects to a domain outside the allowlist. The `browser_goto` domain check only covers explicit navigation, not redirects.
**Solution:** Route interception on all `document`-type requests. Every page navigation — whether from `goto()`, a redirect, or a click — is checked against the domain allowlist. Disallowed navigations are aborted at the network level.

### EC-14: MCP Tool Results Saturate Context

**Problem:** MCP servers can return enormous results (full database tables, large file contents). Research shows up to 236x token inflation from naive MCP integration.
**Solution:** All MCP tool results are truncated to 5,000 chars. Tool descriptions are prefixed with `[MCP:server]` for attribution. Large results should be written to a file in /workspace and the file path returned instead.

### EC-15: Chromium Memory Leaks in Long Sessions

**Problem:** Chromium's memory usage grows over time. In sessions with many page navigations, the browser process can consume gigabytes.
**Solution:** Browser `restart()` method kills and re-launches the browser. Trigger when memory usage is concerning or between unrelated browsing tasks.

### EC-16: LLM Emits Multiple Parallel Tool Calls

**Problem:** The agent loop only processes the first tool call and ignores the rest.
**Solution:** The agent loop iterates over ALL tool calls in the response. Each call is executed, checked against security, and its result appended to the message history.

### EC-17: Cost Runaway from Agent Loops

**Problem:** A confused agent can loop indefinitely, burning through API credits.
**Solution:** CostTracker with configurable `max_cost_per_session_usd` (default $5). The agent loop checks cost after every LLM call and stops if the budget is exhausted. Cost is displayed in the TUI.

### EC-18: Anti-Bot Detection Blocks Browser Automation

**Problem:** Target website detects the browser as automated and blocks access (Cloudflare challenge page, CAPTCHA, 403 response).
**Solution:** Tiered escalation:
1. Retry with Patchright (may be a transient challenge)
2. Switch to Pydoll backend (CDP-direct, no WebDriver artifacts)
3. Switch to Camoufox backend (C++ Firefox patches)
4. Fall back to vision mode (screenshot + VLM)
5. If all fail, report to user with specific detection details and suggest manual intervention

The agent should detect block signals (challenge pages, repeated 403s, CAPTCHA elements) and escalate automatically rather than retrying the same failing approach.

### EC-19: Secrets Leaked to LLM Context

**Problem:** The agent reads `.env` files or private keys and sends them to the LLM API.
**Solution:** SecurityManager checks `.coworkignore` patterns before allowing file reads. Default patterns exclude `.env`, `*.pem`, `*.key`, credential files. The LLM is instructed via system prompt not to read or transmit credential-like files.

---

## Security Model

### Defense in Depth (5 Layers)

```
Layer  │ What it protects               │ How
───────┼────────────────────────────────┼──────────────────────────────
  1    │ Kernel attack surface          │ gVisor/bubblewrap namespace isolation
  2    │ Filesystem                     │ Explicit mount allowlist (not full root)
  3    │ Network                        │ --unshare-net + route interception
  4    │ Dangerous operations           │ Command blocklist + trash-for-delete
  5    │ User intent                    │ Permission prompts for high-risk ops
```

Security is enforced at the kernel level (sandbox), not just in Python code. The Python SecurityManager is defense-in-depth, not the primary barrier.

### Threat Model

| Threat | Mitigation |
|---|---|
| LLM generates `rm -rf /` | `rm` intercepted to trash. Sandbox restricts writes to /workspace. Even bypassing the intercept (via Python, find -delete, etc.), the sandbox mount namespace prevents access outside /workspace. |
| LLM exfiltrates data via curl | Sandbox uses `--unshare-net` (no network by default). Browser is the only network path when enabled, controlled by route-level domain enforcement. |
| LLM reads ~/.ssh/id_rsa | Sandbox mounts only explicit paths. /home is a tmpfs. Host home directory is never bind-mounted. |
| LLM reads .env or credentials | `.coworkignore` patterns exclude credential files from agent reads. System prompt instructs against transmitting secrets. |
| Container escape (CVE-2019-5736) | gVisor intercepts syscalls in user-space kernel. No shared host kernel surface. |
| Prompt injection via web page | Browser content truncated. MCP results truncated and tagged as external. No automatic code execution from page content. |
| Malicious MCP server | MCP servers listed in config file, user-approved. Results treated as untrusted (truncated, prefixed). MCP servers run inside the same sandbox namespace. |
| Navigation to malicious domain | Route interception checks domain allowlist on every document request, including redirects and click-triggered navigations. |
| Cost runaway | CostTracker enforces per-session spending limit. Agent loop halts when budget is exhausted. |

---

## Known Limitations

1. **Command-level safety intercepts are bypassable.** The `rm` to trash intercept and command blocklist can be circumvented via Python one-liners, `find -delete`, or other interpreters. The kernel-level sandbox is the real security boundary; the intercepts are a safety net for accidental damage, not adversarial resistance.

2. **LiteLLM has documented reliability issues at scale.** Token counting inaccuracies, cold start latency (~3s for serverless), and retry loop edge cases are documented by production users. For single-user workloads this is acceptable. A direct-SDK fallback path should be available for users who hit LiteLLM bugs.

3. **Context compaction is lossy.** Summarizing older messages discards details. Tasks exceeding ~30 turns may experience degraded agent performance as compacted history loses nuance. The ContextState object mitigates this by preserving structured facts across compaction cycles, but it cannot capture everything.

4. **Token counting for non-OpenAI models uses heuristics.** `litellm.token_counter()` supports most major providers but falls back to character-based estimation for obscure models. This can cause the token budget to underestimate or overestimate actual usage.

5. **Single-session only at Foundation tier.** No concurrent session support, no file locking, no shared state. Running multiple instances against the same workspace will cause conflicts.

6. **Anti-bot detection is an arms race.** Patchright handles ~70% of websites today. Pydoll and Camoufox extend coverage to ~90%. The remaining ~10% (banking, heavy anti-bot, image CAPTCHAs) will not work reliably. As anti-bot systems evolve, current stealth measures may become less effective. The pluggable backend architecture allows adopting new stealth technologies as they emerge.

7. **NFS/network mounts may not work with bubblewrap.** `--bind` relies on Linux mount namespaces, which have limited support for some network filesystem types. Local disk is recommended.

8. **TOCTOU in path validation.** `Path.resolve()` followed by a separate file operation has a theoretical race condition if symlinks are modified between the check and the access. The sandbox mount namespace mitigates this in practice by limiting what paths exist inside the namespace.

9. **MCP security depends on server trust.** MCP servers execute with whatever permissions the host process has (inside the sandbox). A malicious MCP server could theoretically perform actions within the sandbox namespace. Mitigation: only connect to reviewed/trusted MCP servers, and run them inside the same sandbox isolation.

10. **Patchright disables Console API.** One of Patchright's anti-detection patches disables `Console.enable`, meaning `console.log()` output is not captured. If console access is needed for debugging, use the `browser_js` tool with custom logging or temporarily switch to vanilla Playwright for development.

11. **Camoufox is MPL-2.0 licensed.** While compatible with MIT projects (file-level copyleft, not project-level), modifications to Camoufox's own source files must remain MPL-2.0. This only matters if forking Camoufox itself, not when using it as a dependency.
