# Open Cowork: Architecture & Implementation Specification

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
12. [Feature Roadmap](#feature-roadmap)
13. [Technical Decisions & Rationale](#technical-decisions--rationale)
14. [Edge Cases & Solutions](#edge-cases--solutions)
15. [Security Model](#security-model)
16. [Known Limitations](#known-limitations)

---

## Executive Summary

**Open Cowork** is a Linux-native, open-source alternative to [Claude Cowork](https://claude.com/blog/cowork-research-preview) — Anthropic's agentic desktop assistant (launched January 2026, macOS-only, $100-200/month).

Open Cowork provides an autonomous agent capable of:
- Managing files and directories on the local filesystem
- Automating browser interactions via Playwright
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
| Browser | Chrome extension (screenshot-based) | Playwright (DOM-first, faster) |
| Extensibility | Connectors (Anthropic-reviewed) | MCP servers (open ecosystem) |
| Sandboxing | VM isolation | gVisor / bubblewrap (configurable) |
| Source | Closed | MIT open source |

---

## Competitive Context

### Landscape (March 2026)

| Project | Focus | Sandbox | LLM Support | Stars |
|---|---|---|---|---|
| **Claude Cowork** | Desktop automation (macOS) | VM isolation | Claude only | N/A (closed) |
| **OpenHands** | Coding agent | Docker/E2B/Modal | Multi-provider | ~50K |
| **OpenCode** | Terminal coding agent | Local (bubblewrap optional) | 75+ providers | ~108K |
| **Browser-Use** | Web automation | None | Multi-provider | ~60K |
| **Open Interpreter** | General computer use | Optional Docker | Multi-provider | ~55K |
| **Open Cowork** | Desktop automation (Linux) | gVisor/bubblewrap | Multi-provider | — |

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

4. **DOM First, Vision as Fallback**
   - Playwright for 90% of web tasks (faster, cheaper, more reliable)
   - Vision models only when DOM fails (canvas elements, anti-automation sites)
   - Direct DOM access avoids screenshot round-trip latency and vision model costs

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
│      (Bash + Browser + MCP Servers)              │
├─────────────────────────────────────────────────┤
│            Layer 4: Security Manager             │
│    (Permissions, Path Validation, Network)        │
├─────────────────────────────────────────────────┤
│            Layer 1: Sandbox                      │
│      (gVisor / bubblewrap / Docker)              │
└─────────────────────────────────────────────────┘
```

**6 layers. No gaps.**

- **Layer 1 (Sandbox):** Kernel-level isolation for untrusted code execution
- **Layer 2 (Agent Core):** LLM provider abstraction + ReAct execution loop + cost tracking
- **Layer 3 (Tool Runtime):** Bash REPL, Playwright browser, MCP tool servers
- **Layer 4 (Security):** Permission model, path validation, network policy, MCP result sandboxing
- **Layer 5 (Observability):** TUI interface, progress tracking, audit logging, cost display
- **Layer 6 (Session):** Lifecycle management, checkpointing, graceful shutdown

---

## Layer 1: Sandbox

**Purpose:** Kernel-level isolation for executing untrusted LLM-generated code.

### Why Not Just Docker?

Docker containers share the host kernel. LLM-generated code is unpredictable and cannot be audited ahead of execution. CVE-2019-5736 demonstrated container escape is real. The 2026 industry consensus: containers alone are insufficient for agent sandboxing.

### Sandbox Tiers

```yaml
Tier 1 — bubblewrap (zero dependencies):
  Description: Linux namespace isolation without Docker daemon
  Startup: <10ms
  Security: Namespace isolation (mount, PID, network, user)
  Use case: Default for local development / single-user
  Trade-off: No cgroup resource limits (use ulimit instead)

Tier 2 — gVisor (recommended production):
  Description: User-space kernel that intercepts syscalls
  Startup: ~50ms (via runsc OCI runtime)
  Security: Dramatically reduced kernel attack surface
  Use case: Production, multi-user, untrusted workloads
  Trade-off: Some syscall incompatibilities (rare for Python/Node)

Tier 3 — Docker with gVisor runtime (enterprise):
  Description: Full container orchestration with gVisor isolation
  Startup: ~200ms
  Security: gVisor + cgroups + namespace isolation
  Use case: Enterprise with existing Docker infrastructure
  Trade-off: Requires Docker daemon, heavier setup

Tier 4 — Firecracker microVM (maximum security):
  Description: Hardware-level isolation via KVM
  Startup: ~125ms
  Security: Separate kernel per session, VM-level isolation
  Use case: Multi-tenant SaaS, compliance-critical
  Trade-off: Requires KVM support, more operational complexity
```

### bubblewrap Implementation

```python
import subprocess
import os
from pathlib import Path


class BubblewrapSandbox:
    """
    Lightweight Linux namespace sandbox using bubblewrap (bwrap).
    No Docker daemon required. Installed via: apt install bubblewrap

    Mount strategy: explicit allowlist of system paths.
    Only /usr, /bin, /sbin, /lib, /lib64, and minimal /etc are exposed read-only.
    /proc and /dev are mounted as minimal isolated filesystems.
    The host root is never bind-mounted wholesale.
    """

    # System paths to expose read-only inside the sandbox
    SYSTEM_RO_PATHS = ['/usr', '/bin', '/sbin', '/lib', '/lib64']

    # Minimal /etc entries for DNS resolution and TLS
    ETC_RO_PATHS = [
        '/etc/resolv.conf',
        '/etc/ssl',
        '/etc/ca-certificates',
        '/etc/ld.so.cache',
        '/etc/ld.so.conf',
        '/etc/ld.so.conf.d',
        '/etc/alternatives',
    ]

    def __init__(self, workspace: Path, config: dict = None):
        self.workspace = workspace.resolve()
        self.trash_dir = Path.home() / '.cowork' / 'trash'
        self.trash_dir.mkdir(parents=True, exist_ok=True)
        self.config = config or {}

    def execute(self, command: list[str], timeout: int = 30) -> dict:
        bwrap_cmd = self._build_bwrap_command(command)

        try:
            result = subprocess.run(
                bwrap_cmd,
                capture_output=True,
                text=True,
                timeout=timeout,
                shell=False,
            )
            return {
                'stdout': result.stdout,
                'stderr': result.stderr,
                'exit_code': result.returncode,
            }
        except subprocess.TimeoutExpired:
            return {
                'stdout': '',
                'stderr': f'Command timed out after {timeout}s',
                'exit_code': -1,
            }
        except FileNotFoundError:
            return {
                'stdout': '',
                'stderr': 'bubblewrap (bwrap) not found. Install: apt install bubblewrap',
                'exit_code': -1,
            }

    def _build_bwrap_command(self, command: list[str]) -> list[str]:
        bwrap = ['bwrap']

        # Read-only system paths (explicit allowlist, never full root)
        for sys_path in self.SYSTEM_RO_PATHS:
            if os.path.exists(sys_path):
                bwrap.extend(['--ro-bind', sys_path, sys_path])

        # Minimal /etc entries (DNS, TLS, dynamic linker)
        for etc_path in self.ETC_RO_PATHS:
            if os.path.exists(etc_path):
                bwrap.extend(['--ro-bind', etc_path, etc_path])

        # Writable workspace and trash
        bwrap.extend([
            '--bind', str(self.workspace), '/workspace',
            '--bind', str(self.trash_dir), '/trash',
        ])

        # Isolated tmpfs for sensitive paths
        bwrap.extend([
            '--tmpfs', '/tmp',
            '--tmpfs', '/home',
            '--tmpfs', '/run',
        ])

        # Minimal /proc and /dev (not the host's real /proc or /dev)
        bwrap.extend([
            '--proc', '/proc',
            '--dev', '/dev',
        ])

        # Namespace isolation
        bwrap.extend([
            '--unshare-pid',
            '--unshare-uts',
            '--die-with-parent',
            '--chdir', '/workspace',
        ])

        # Network isolation is the default; allow only when explicitly configured
        if not self.config.get('allow_network', False):
            bwrap.append('--unshare-net')

        bwrap.extend(['--'] + command)
        return bwrap


class GVisorSandbox:
    """
    gVisor-based sandbox using Docker with runsc runtime.
    Requires: docker + gVisor runsc runtime installed.
    """

    def __init__(self, workspace: Path, image: str = 'opencowork:base'):
        self.workspace = workspace.resolve()
        self.image = image

    async def create_container(self) -> str:
        import docker
        client = docker.from_env()

        container = client.containers.run(
            image=self.image,
            runtime='runsc',
            user=f'{os.getuid()}:{os.getgid()}',
            volumes={
                str(self.workspace): {'bind': '/workspace', 'mode': 'rw'},
            },
            environment={
                'WORKSPACE': '/workspace',
            },
            network_mode='none',
            detach=True,
            mem_limit='4g',
            cpu_count=2,
            pids_limit=256,
            security_opt=['no-new-privileges'],
            read_only=True,
            tmpfs={'/tmp': 'size=512m'},
        )
        return container.id

    async def execute(self, container_id: str, command: list[str], timeout: int = 30) -> dict:
        import docker
        client = docker.from_env()
        container = client.containers.get(container_id)

        try:
            exit_code, output = container.exec_run(
                cmd=command,
                workdir='/workspace',
                demux=True,
            )
            stdout = output[0].decode() if output[0] else ''
            stderr = output[1].decode() if output[1] else ''
            return {'stdout': stdout, 'stderr': stderr, 'exit_code': exit_code}
        except Exception as e:
            return {'stdout': '', 'stderr': str(e), 'exit_code': -1}
```

### Container Image (Docker/gVisor Tiers Only)

A single well-equipped base image for tiers that use Docker:

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl wget jq ripgrep tree file unzip \
    libreoffice-calc libreoffice-writer \
    ghostscript imagemagick \
    && rm -rf /var/lib/apt/lists/*

RUN pip install playwright && playwright install chromium --with-deps

RUN pip install --no-cache-dir \
    pandas numpy openpyxl python-docx \
    beautifulsoup4 requests httpx \
    Pillow pdf2image pymupdf \
    litellm mcp

COPY agent/ /opt/agent/
WORKDIR /workspace

RUN useradd -m -u 1000 agent
USER agent
```

**Size:** ~1.5GB (acceptable — OpenHands is ~10GB).
Pre-install common packages. If a rare package is needed, `pip install` at runtime with user confirmation.

For the bubblewrap tier (no Docker), the agent runs host-installed tools inside namespace isolation. Required host packages: `python3`, `bwrap`, `git`. Optional: `playwright`, MCP server binaries.

---

## Layer 2: Agent Core

**Purpose:** LLM provider abstraction, execution loop, and cost tracking.

### LLM Provider

```python
import litellm
import asyncio
import random
from dataclasses import dataclass, field


@dataclass
class ModelConfig:
    primary: str = 'deepseek/deepseek-chat'
    fallback: str = 'anthropic/claude-sonnet-4'
    vision: str = 'anthropic/claude-sonnet-4'
    max_retries: int = 2
    timeout: int = 60
    max_cost_per_session_usd: float = 5.0


class LLMProviderError(Exception):
    pass


class LLMProvider:
    """
    Provider-agnostic LLM interface via litellm (used as library, not proxy server).
    """

    def __init__(self, config: ModelConfig = None):
        self.config = config or ModelConfig()

    async def completion(
        self,
        messages: list[dict],
        model: str = None,
        tools: list[dict] = None,
        stream: bool = False,
    ) -> dict:
        model = model or self.config.primary

        for attempt in range(self.config.max_retries + 1):
            current_model = model if attempt == 0 else self.config.fallback
            try:
                response = await litellm.acompletion(
                    model=current_model,
                    messages=messages,
                    tools=tools,
                    stream=stream,
                    timeout=self.config.timeout,
                )
                return response
            except litellm.RateLimitError:
                # Respect rate limits with exponential backoff + jitter
                wait = (2 ** attempt) + random.uniform(0, 1)
                await asyncio.sleep(wait)
                continue
            except litellm.AuthenticationError as e:
                # Do not retry auth failures
                raise LLMProviderError(f'Authentication failed for {current_model}: {e}') from e
            except Exception as e:
                if attempt == self.config.max_retries:
                    raise LLMProviderError(f'All models failed: {e}') from e
                # Exponential backoff for transient errors
                wait = (2 ** attempt) + random.uniform(0, 1)
                await asyncio.sleep(wait)
                continue
```

### Cost Tracker

```python
@dataclass
class CostTracker:
    max_cost_usd: float = 5.0
    session_cost_usd: float = 0.0
    calls: list[dict] = field(default_factory=list)

    def record(self, response):
        """Record cost from an LLM response using litellm's cost calculator."""
        try:
            cost = litellm.completion_cost(completion_response=response)
        except Exception:
            cost = 0.0
        self.session_cost_usd += cost
        self.calls.append({
            'model': getattr(response, 'model', 'unknown'),
            'cost_usd': cost,
            'tokens': response.usage.total_tokens if hasattr(response, 'usage') and response.usage else 0,
        })

    def exceeds_limit(self) -> bool:
        return self.session_cost_usd >= self.max_cost_usd
```

### Agent Loop (ReAct Pattern)

The core execution pattern is ReAct (Reason + Act). The LLM receives context, decides what tool to call, observes the result, and repeats until the task is done.

This is simpler and more reliable than plan-and-execute for the majority of tasks. The LLM does not need to predict the entire execution path upfront — it adapts to each result as it arrives.

```python
import json


class AgentLoop:
    """
    ReAct execution loop.

    Flow per turn:
      1. Send messages + tool definitions to LLM
      2. If LLM returns tool calls, execute ALL of them
      3. Append results to message history
      4. If context is growing large, compact older messages
      5. Repeat until LLM produces a text response (task complete) or limits hit
    """

    def __init__(
        self,
        llm: LLMProvider,
        tools: 'ToolRuntime',
        security: 'SecurityManager',
        event_bus: 'EventBus',
        context: 'ContextState',
        message_manager: 'MessageManager',
        cost_tracker: CostTracker,
    ):
        self.llm = llm
        self.tools = tools
        self.security = security
        self.event_bus = event_bus
        self.context = context
        self.message_manager = message_manager
        self.cost_tracker = cost_tracker

    async def execute_task(self, user_prompt: str, max_turns: int = 25) -> dict:
        self.message_manager.add_user_message(user_prompt)

        turn = 0
        for turn in range(max_turns):
            messages = self.message_manager.get_messages()

            response = await self.llm.completion(
                messages=messages,
                tools=self.tools.get_definitions(),
            )

            self.cost_tracker.record(response)
            msg = response.choices[0].message
            self.message_manager.add_assistant_message(msg)

            if not msg.tool_calls:
                break

            # Process ALL tool calls in the response (LLMs emit parallel calls)
            for tool_call in msg.tool_calls:
                func_name = tool_call.function.name
                try:
                    args = json.loads(tool_call.function.arguments)
                except json.JSONDecodeError:
                    self.message_manager.add_tool_result(
                        tool_call_id=tool_call.id,
                        result={'error': 'Malformed tool call arguments'},
                    )
                    continue

                if not await self.security.check_permission(func_name, args):
                    result = {'error': 'Permission denied by user'}
                else:
                    result = await self.tools.execute(func_name, **args)

                self.context.update_from_result(func_name, args, result)
                await self.event_bus.emit_action(func_name, str(args)[:200], str(result)[:200])

                self.message_manager.add_tool_result(
                    tool_call_id=tool_call.id,
                    result=result,
                )

            # Check cost limit
            if self.cost_tracker.exceeds_limit():
                await self.event_bus.emit('cost_limit', {
                    'spent': self.cost_tracker.session_cost_usd,
                    'limit': self.cost_tracker.max_cost_usd,
                })
                break

        return {
            'status': 'completed',
            'cost_usd': self.cost_tracker.session_cost_usd,
            'turns': turn + 1,
            'context': self.context.summary(),
        }


SYSTEM_PROMPT = """You are Open Cowork, an autonomous Linux agent.
You help users accomplish tasks by executing commands, browsing the web, and connecting to external services.

Available tools:
- run_bash: Execute shell commands in a sandboxed workspace (/workspace)
- browser_goto: Navigate to a URL
- browser_click: Click an element (CSS selector)
- browser_type: Type into an input field
- browser_read: Extract text from a page or element
- mcp_call: Call a connected MCP tool server

Rules:
- All file operations are scoped to /workspace
- Use trash (mv to /trash) instead of rm for deletions
- Confirm destructive operations with the user
- Read files prior to modifying them
- Verify results after multi-step operations

When working:
1. State what you're about to do
2. Execute the action
3. Check the result
4. Continue or adjust based on output"""
```

### Plan-and-Execute (Optimization Layer)

For batch-style tasks where the full execution path is predictable (e.g., "rename all .jpeg files to .jpg"), a plan-and-execute layer can reduce LLM calls by generating the plan upfront and executing steps directly.

This is an optimization on top of the ReAct loop, not a replacement. It is appropriate when:
- The task is decomposable into concrete tool calls at planning time
- Steps do not depend on dynamic content (web pages, API responses)
- The user requests a predictable, reviewable plan

```python
class PlanAndExecute:
    """
    Optional optimization layer. Generates a plan via structured output,
    then executes deterministic steps without calling the LLM.
    Falls back to ReAct for any step that fails or requires judgment.
    """

    def __init__(self, llm: LLMProvider, tools: 'ToolRuntime', security: 'SecurityManager'):
        self.llm = llm
        self.tools = tools
        self.security = security

    async def generate_plan(self, prompt: str, workspace_state: str) -> list[dict]:
        response = await self.llm.completion(
            messages=[
                {'role': 'system', 'content': PLANNER_SYSTEM_PROMPT},
                {'role': 'user', 'content': f'{prompt}\n\nWorkspace contents:\n{workspace_state}'},
            ],
            tools=[PLAN_TOOL_DEFINITION],
        )
        tool_call = response.choices[0].message.tool_calls[0]
        plan_data = json.loads(tool_call.function.arguments)
        return plan_data.get('steps', [])

    async def execute_plan(self, steps: list[dict]) -> list[dict]:
        results = []
        for step in steps:
            if not await self.security.check_permission(step['tool'], step.get('args', {})):
                results.append({'step': step, 'error': 'Permission denied'})
                continue
            try:
                result = await self.tools.execute(step['tool'], **step.get('args', {}))
                results.append({'step': step, 'result': result})
            except Exception as e:
                results.append({'step': step, 'error': str(e)})
                # Stop executing remaining steps on failure; caller should fall back to ReAct
                break
        return results


PLANNER_SYSTEM_PROMPT = """You are generating an execution plan for a Linux automation task.

Create a plan with concrete, executable steps using the create_plan tool.
Each step specifies a tool and its arguments.

Available tools: run_bash, browser_goto, browser_click, browser_type, browser_read, mcp_call

Safety rules:
- Use trash instead of rm for deletions
- All file operations scoped to /workspace
- Include verification steps where appropriate"""


PLAN_TOOL_DEFINITION = {
    'type': 'function',
    'function': {
        'name': 'create_plan',
        'description': 'Create an execution plan for the user task',
        'parameters': {
            'type': 'object',
            'properties': {
                'goal': {
                    'type': 'string',
                    'description': 'One-sentence summary of the goal',
                },
                'steps': {
                    'type': 'array',
                    'items': {
                        'type': 'object',
                        'properties': {
                            'tool': {
                                'type': 'string',
                                'enum': ['run_bash', 'browser_goto', 'browser_click',
                                         'browser_type', 'browser_read', 'mcp_call'],
                            },
                            'args': {'type': 'object'},
                            'description': {'type': 'string'},
                        },
                        'required': ['tool', 'args', 'description'],
                    },
                },
            },
            'required': ['goal', 'steps'],
        },
    },
}
```

---

## Layer 3: Tool Runtime

**Purpose:** Tool definitions, execution, and MCP integration.

### Tool Registry

```python
import shlex
import urllib.parse
from typing import Any


class ToolRuntime:
    """Central tool registry. Routes tool calls to the appropriate handler."""

    def __init__(self, sandbox: 'BubblewrapSandbox | GVisorSandbox', security: 'SecurityManager'):
        self.sandbox = sandbox
        self.security = security
        self.bash = BashTool(sandbox, security)
        self.browser = BrowserTool(security)
        self.mcp_manager = MCPManager()

    def get_definitions(self) -> list[dict]:
        return [
            *self.bash.definitions(),
            *self.browser.definitions(),
            *self.mcp_manager.get_tool_definitions(),
        ]

    async def execute(self, func_name: str, **kwargs) -> Any:
        if func_name == 'run_bash':
            return await self.bash.execute(kwargs['command'], kwargs.get('working_dir', '/workspace'))
        elif func_name.startswith('browser_'):
            action = func_name.replace('browser_', '')
            return await self.browser.execute(action, **kwargs)
        elif func_name.startswith('mcp_'):
            # Route mcp_servername_toolname to the correct server and tool
            parts = func_name.split('_', 2)
            if len(parts) == 3:
                return await self.mcp_manager.call_tool(parts[1], parts[2], kwargs)
            return await self.mcp_manager.call_tool(
                kwargs.get('server', ''), kwargs.get('tool', ''), kwargs.get('args', {}),
            )
        else:
            return {'error': f'Unknown tool: {func_name}'}

    async def close(self):
        await self.browser.close()
        await self.mcp_manager.close()
```

### Bash Tool

```python
class BashTool:
    """
    Bash execution with safety intercepts.

    The command blocklist and rm-to-trash intercept are defense-in-depth measures,
    not a security boundary. The LLM can invoke Python, Perl, or other interpreters
    to bypass command-level intercepts. The real security boundary is the kernel-level
    sandbox (Layer 1), which restricts filesystem access regardless of how commands
    are invoked.

    shell=True is never used. Commands are parsed to lists via shlex and
    executed as subprocess arguments.
    """

    BLOCKED_COMMANDS = {'dd', 'mkfs', 'fdisk', 'mount', 'umount', 'chown', 'su', 'sudo'}
    TRASH_INTERCEPTED = {'rm'}

    def __init__(self, sandbox, security: 'SecurityManager'):
        self.sandbox = sandbox
        self.security = security

    def definitions(self) -> list[dict]:
        return [{
            'type': 'function',
            'function': {
                'name': 'run_bash',
                'description': 'Execute a bash command in the secure workspace. Use for file operations, data processing, package installation.',
                'parameters': {
                    'type': 'object',
                    'properties': {
                        'command': {'type': 'string', 'description': 'The bash command to execute'},
                        'working_dir': {'type': 'string', 'description': 'Working directory (default: /workspace)'},
                    },
                    'required': ['command'],
                },
            },
        }]

    async def execute(self, command: str, working_dir: str = '/workspace') -> dict:
        try:
            cmd_parts = shlex.split(command)
        except ValueError as e:
            return {'error': f'Invalid command syntax: {e}', 'exit_code': -1}

        if not cmd_parts:
            return {'error': 'Empty command', 'exit_code': -1}

        base_cmd = cmd_parts[0]

        if base_cmd in self.BLOCKED_COMMANDS:
            return {'error': f'Command blocked for safety: {base_cmd}', 'exit_code': -1}

        if base_cmd in self.TRASH_INTERCEPTED:
            return await self._safe_delete(cmd_parts[1:])

        for part in cmd_parts[1:]:
            if part.startswith('/') and not self.security.validate_path(part):
                return {'error': f'Path outside allowed roots: {part}', 'exit_code': -1}

        result = self.sandbox.execute(cmd_parts, timeout=30)

        # Truncate large outputs to protect context window
        stdout = result.get('stdout', '')
        if len(stdout) > 10_000:
            result['stdout'] = stdout[:5_000] + '\n\n... [truncated, showing first 5000 and last 2000 chars] ...\n\n' + stdout[-2_000:]
            result['truncated'] = True

        return result

    async def _safe_delete(self, targets: list[str]) -> dict:
        results = []
        for target in targets:
            if target.startswith('-'):
                continue
            result = self.security.safe_delete(target)
            results.append(result)
        return {'stdout': '\n'.join(results), 'exit_code': 0}
```

### Browser Tool (Playwright, Async)

```python
class BrowserTool:
    """
    Playwright-based browser automation.

    Key design choices:
    - Async Playwright API
    - 'domcontentloaded' wait strategy (SPAs never reach 'networkidle')
    - inner_text() instead of content() to reduce token usage ~90%
    - Route interception to enforce domain allowlist on ALL navigations,
      including redirects and clicks, not only explicit goto calls
    """

    MAX_CONTENT_CHARS = 8_000

    def __init__(self, security: 'SecurityManager'):
        self.security = security
        self._pw = None
        self._browser = None
        self._page = None

    async def _ensure_browser(self):
        if self._pw is None:
            from playwright.async_api import async_playwright
            self._pw = await async_playwright().start()
            self._browser = await self._pw.chromium.launch(headless=True)
            self._page = await self._browser.new_page()

            # Intercept document navigations to enforce domain allowlist globally
            await self._page.route('**/*', self._route_handler)

    async def _route_handler(self, route):
        """Block document navigations to domains outside the allowlist."""
        if route.request.resource_type == 'document':
            domain = urllib.parse.urlparse(route.request.url).netloc
            if domain and not self.security.check_network_permission(domain):
                await route.abort('blockedbyclient')
                return
        await route.continue_()

    def definitions(self) -> list[dict]:
        return [
            {
                'type': 'function',
                'function': {
                    'name': 'browser_goto',
                    'description': 'Navigate to a URL',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'url': {'type': 'string', 'description': 'URL to navigate to'},
                        },
                        'required': ['url'],
                    },
                },
            },
            {
                'type': 'function',
                'function': {
                    'name': 'browser_click',
                    'description': 'Click an element using CSS selector',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'selector': {'type': 'string'},
                            'timeout': {'type': 'integer', 'description': 'Wait timeout in ms (default 10000)'},
                        },
                        'required': ['selector'],
                    },
                },
            },
            {
                'type': 'function',
                'function': {
                    'name': 'browser_type',
                    'description': 'Type text into an input field',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'selector': {'type': 'string'},
                            'text': {'type': 'string'},
                        },
                        'required': ['selector', 'text'],
                    },
                },
            },
            {
                'type': 'function',
                'function': {
                    'name': 'browser_read',
                    'description': 'Extract text content from the page or a specific element',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'selector': {
                                'type': 'string',
                                'description': 'CSS selector (reads visible text of full page if omitted)',
                            },
                        },
                    },
                },
            },
        ]

    async def execute(self, action: str, **kwargs) -> dict:
        await self._ensure_browser()
        try:
            if action == 'goto':
                return await self._goto(kwargs['url'])
            elif action == 'click':
                return await self._click(kwargs['selector'], kwargs.get('timeout', 10_000))
            elif action == 'type':
                return await self._type(kwargs['selector'], kwargs['text'])
            elif action == 'read':
                return await self._read(kwargs.get('selector'))
            else:
                return {'error': f'Unknown browser action: {action}'}
        except Exception as e:
            return {'error': str(e), 'action': action}

    async def _goto(self, url: str) -> dict:
        domain = urllib.parse.urlparse(url).netloc
        if not self.security.check_network_permission(domain):
            return {'error': f'Domain not in allowlist: {domain}'}

        await self._page.goto(url, wait_until='domcontentloaded', timeout=30_000)
        return {'status': 'success', 'url': self._page.url, 'title': await self._page.title()}

    async def _click(self, selector: str, timeout: int) -> dict:
        await self._page.click(selector, timeout=timeout)
        return {'status': 'success', 'selector': selector}

    async def _type(self, selector: str, text: str) -> dict:
        await self._page.fill(selector, text)
        return {'status': 'success'}

    async def _read(self, selector: str = None) -> dict:
        if selector:
            text = await self._page.inner_text(selector)
        else:
            text = await self._page.inner_text('body')

        if len(text) > self.MAX_CONTENT_CHARS:
            text = text[:self.MAX_CONTENT_CHARS] + f'\n\n... [truncated at {self.MAX_CONTENT_CHARS} chars]'

        return {'text': text, 'length': len(text)}

    async def close(self):
        if self._browser:
            await self._browser.close()
        if self._pw:
            await self._pw.stop()
        self._pw = None
        self._browser = None
        self._page = None

    async def restart(self):
        """Kill and restart the browser to clear memory leaks and stale state."""
        await self.close()
        await self._ensure_browser()
```

### MCP Integration

```python
from dataclasses import dataclass


class MCPManager:
    """
    Model Context Protocol integration.

    MCP is the standard for connecting AI agents to external tools.
    Supports the entire ecosystem of pre-built MCP servers:
    filesystem, git, GitHub, Slack, Google Drive, databases, etc.

    Tool results from MCP servers are treated as untrusted external data
    and are truncated/sanitized prior to inclusion in LLM context.

    Reference: https://modelcontextprotocol.io/
    """

    def __init__(self):
        self.servers: dict[str, 'MCPServerConnection'] = {}

    async def connect_server(self, name: str, command: list[str], env: dict = None):
        """
        Connect to an MCP server (stdio transport).

        Example:
            await mcp.connect_server('filesystem',
                ['npx', '@modelcontextprotocol/server-filesystem', '/workspace'])
            await mcp.connect_server('github',
                ['npx', '@modelcontextprotocol/server-github'],
                {'GITHUB_TOKEN': '...'})
        """
        from mcp import ClientSession, StdioServerParameters
        from mcp.client.stdio import stdio_client

        server_params = StdioServerParameters(
            command=command[0],
            args=command[1:],
            env=env,
        )
        transport = await stdio_client(server_params).__aenter__()
        session = ClientSession(*transport)
        await session.initialize()

        self.servers[name] = MCPServerConnection(
            name=name,
            session=session,
            tools=await session.list_tools(),
        )

    def get_tool_definitions(self) -> list[dict]:
        definitions = []
        for server in self.servers.values():
            for tool in server.tools:
                definitions.append({
                    'type': 'function',
                    'function': {
                        'name': f'mcp_{server.name}_{tool.name}',
                        'description': f'[MCP:{server.name}] {tool.description}',
                        'parameters': tool.inputSchema,
                    },
                })
        return definitions

    async def call_tool(self, server_name: str, tool_name: str, args: dict) -> dict:
        if server_name not in self.servers:
            return {'error': f'MCP server not connected: {server_name}'}

        server = self.servers[server_name]
        try:
            result = await server.session.call_tool(tool_name, args)
            content = str(result.content)
            # Truncate large MCP results to prevent context saturation
            if len(content) > 5_000:
                content = content[:4_000] + '\n...[truncated]...\n' + content[-500:]
            return {'result': content, 'source': 'mcp', 'server': server_name}
        except Exception as e:
            return {'error': str(e)}

    async def close(self):
        for server in self.servers.values():
            try:
                await server.session.__aexit__(None, None, None)
            except Exception:
                pass
        self.servers.clear()


@dataclass
class MCPServerConnection:
    name: str
    session: Any
    tools: list
```

---

## Layer 4: Security Manager

**Purpose:** Permission management and safety enforcement at the application level.
This is defense-in-depth ON TOP of kernel-level sandbox isolation (Layer 1).

```python
import shutil
import uuid
from pathlib import Path


class SecurityManager:
    """
    Application-level security (Layer 4).
    Works in conjunction with kernel-level sandbox (Layer 1).

    Defense layers:
    1. Sandbox (kernel) — namespace/gVisor/Firecracker isolation
    2. This class — path validation, permission model, trash delete
    3. Network — sandbox --unshare-net + route interception in browser
    """

    def __init__(self, workspace: Path, event_bus: 'EventBus' = None):
        self.workspace = workspace.resolve()
        self.trash_path = Path.home() / '.cowork' / 'trash'
        self.trash_path.mkdir(parents=True, exist_ok=True)

        self.allowed_roots = [
            self.workspace,
            Path('/tmp'),
            self.trash_path,
        ]

        # Network allowlist (enforced at both application and kernel level)
        self.allowed_domains: set[str] = {
            'pypi.org', 'files.pythonhosted.org',
            'github.com', 'raw.githubusercontent.com',
            'registry.npmjs.org',
        }

        self.session_grants: set[Path] = set()
        self.event_bus = event_bus

    def validate_path(self, path_str: str) -> bool:
        """Check if path is within allowed roots. Resolves symlinks."""
        try:
            path = Path(path_str).resolve()
            if any(path.is_relative_to(root) for root in self.allowed_roots):
                return True
            return any(path.is_relative_to(grant) for grant in self.session_grants)
        except (ValueError, RuntimeError, OSError):
            return False

    async def check_permission(self, tool: str, args: dict) -> bool:
        """
        Intent-based permission check.
        Read operations: auto-granted.
        Write operations within workspace: auto-granted.
        Risky operations: prompt user via EventBus.
        """
        if tool in ('browser_read', 'browser_goto'):
            return True

        if tool == 'run_bash':
            command = args.get('command', '')
            try:
                cmd_parts = shlex.split(command) if command else []
            except ValueError:
                return False
            if cmd_parts and cmd_parts[0] in BashTool.BLOCKED_COMMANDS:
                return False
            for part in cmd_parts:
                if part.startswith('/') and not self.validate_path(part):
                    return await self._ask_permission(
                        f'Command accesses path outside workspace: {part}',
                        risk='high',
                    )
            return True

        if tool in ('browser_click', 'browser_type'):
            return True

        return await self._ask_permission(
            f'Allow {tool} with args {json.dumps(args, default=str)[:200]}?',
            risk='medium',
        )

    def check_network_permission(self, domain: str) -> bool:
        if domain in self.allowed_domains:
            return True
        for allowed in self.allowed_domains:
            if domain.endswith(f'.{allowed}'):
                return True
        return False

    def safe_delete(self, path_str: str) -> str:
        """Move to trash instead of permanent delete."""
        if not self.validate_path(path_str):
            return f'Error: Path outside workspace: {path_str}'

        src = Path(path_str)
        if not src.exists():
            return f'Error: Not found: {path_str}'

        dst = self.trash_path / f'{src.name}.{uuid.uuid4().hex[:8]}'
        try:
            shutil.move(str(src), str(dst))
            return f'Moved to trash: {dst.name}'
        except Exception as e:
            return f'Error: {e}'

    async def _ask_permission(self, message: str, risk: str = 'medium') -> bool:
        """Ask user via EventBus. Returns False on timeout or if no EventBus configured."""
        if self.event_bus:
            return await self.event_bus.request_permission(message, risk)
        return False

    def grant_session_access(self, path: Path):
        self.session_grants.add(path.resolve())

    def add_allowed_domain(self, domain: str):
        self.allowed_domains.add(domain)
```

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

```python
import asyncio
import time
import uuid as uuid_mod


class EventBus:
    """Async event bus for TUI updates and permission requests."""

    def __init__(self):
        self._subscribers: list[asyncio.Queue] = []
        self._pending_permissions: dict[str, asyncio.Future] = {}

    def subscribe(self) -> asyncio.Queue:
        queue = asyncio.Queue(maxsize=1000)
        self._subscribers.append(queue)
        return queue

    async def emit(self, event_type: str, data: dict):
        event = {'type': event_type, 'data': data, 'timestamp': time.time()}
        for queue in self._subscribers:
            try:
                queue.put_nowait(event)
            except asyncio.QueueFull:
                pass

    async def emit_action(self, tool: str, command: str, result: str = ''):
        await self.emit('action', {'tool': tool, 'command': command, 'result': result})

    async def emit_progress(self, step: int, total: int, message: str):
        await self.emit('progress', {'step': step, 'total': total, 'message': message})

    async def request_permission(self, message: str, risk: str) -> bool:
        """
        Emit permission request and wait for TUI to resolve it.
        The TUI calls resolve_permission() when the user clicks Allow/Deny.
        Defaults to deny on 60s timeout.
        """
        request_id = uuid_mod.uuid4().hex
        loop = asyncio.get_running_loop()
        future = loop.create_future()
        self._pending_permissions[request_id] = future

        await self.emit('permission_request', {
            'message': message,
            'risk': risk,
            'request_id': request_id,
        })

        try:
            return await asyncio.wait_for(future, timeout=60.0)
        except asyncio.TimeoutError:
            return False
        finally:
            self._pending_permissions.pop(request_id, None)

    def resolve_permission(self, request_id: str, granted: bool):
        """Called by TUI when user clicks Allow/Deny."""
        future = self._pending_permissions.get(request_id)
        if future and not future.done():
            future.set_result(granted)
```

### Audit Logger

```python
from typing import Optional


class AuditLogger:
    """
    Append-only audit log using aiosqlite (non-blocking in async context).
    No UPDATE/DELETE operations — insert only.
    """

    def __init__(self, db_path: Path = None):
        self.db_path = db_path or Path.home() / '.cowork' / 'audit.db'
        self.db_path.parent.mkdir(parents=True, exist_ok=True)
        self._db = None

    async def _ensure_db(self):
        if self._db is None:
            import aiosqlite
            self._db = await aiosqlite.connect(str(self.db_path))
            await self._db.execute('PRAGMA journal_mode=WAL')
            await self._db.execute('''
                CREATE TABLE IF NOT EXISTS audit_log (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    timestamp REAL NOT NULL,
                    session_id TEXT NOT NULL,
                    tool TEXT NOT NULL,
                    command TEXT NOT NULL,
                    result TEXT,
                    exit_code INTEGER,
                    duration_ms REAL,
                    cost_usd REAL
                )
            ''')
            await self._db.commit()

    async def log(self, session_id: str, tool: str, command: str,
                  result: str = '', exit_code: int = 0, duration_ms: float = 0,
                  cost_usd: float = 0):
        await self._ensure_db()
        await self._db.execute(
            'INSERT INTO audit_log (timestamp, session_id, tool, command, result, exit_code, duration_ms, cost_usd) '
            'VALUES (?, ?, ?, ?, ?, ?, ?, ?)',
            (time.time(), session_id, tool, command, result[:5000], exit_code, duration_ms, cost_usd),
        )
        await self._db.commit()

    async def query(self, session_id: str = None, limit: int = 100) -> list[dict]:
        await self._ensure_db()
        sql = 'SELECT * FROM audit_log'
        params = []
        if session_id:
            sql += ' WHERE session_id = ?'
            params.append(session_id)
        sql += ' ORDER BY timestamp DESC LIMIT ?'
        params.append(limit)

        cursor = await self._db.execute(sql, params)
        columns = [d[0] for d in cursor.description]
        rows = await cursor.fetchall()
        return [dict(zip(columns, row)) for row in rows]

    async def close(self):
        if self._db:
            await self._db.close()
            self._db = None
```

### Structured Logging

```python
import logging
import json as json_mod


class JSONFormatter(logging.Formatter):
    """JSON log formatter for machine-parseable output."""

    def format(self, record):
        log_data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
        }
        if hasattr(record, 'tool'):
            log_data['tool'] = record.tool
        if hasattr(record, 'duration_ms'):
            log_data['duration_ms'] = record.duration_ms
        if hasattr(record, 'cost_usd'):
            log_data['cost_usd'] = record.cost_usd
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        return json_mod.dumps(log_data)


def setup_logging(json_output: bool = False, level: str = 'INFO'):
    handler = logging.StreamHandler()
    if json_output:
        handler.setFormatter(JSONFormatter())
    else:
        handler.setFormatter(logging.Formatter('%(asctime)s %(levelname)s %(name)s: %(message)s'))
    logging.root.addHandler(handler)
    logging.root.setLevel(getattr(logging, level))
```

---

## Layer 6: Session & Recovery

**Purpose:** Session lifecycle, checkpointing, and graceful shutdown.

```python
import signal
import subprocess
import atexit
from dataclasses import dataclass


@dataclass
class Session:
    id: str
    workspace: Path
    created_at: float
    last_activity: float
    status: str = 'active'


class SessionManager:
    """
    Manages session lifecycle with git-based checkpointing.

    Checkpoints are stored in a separate git directory (~/.cowork/checkpoints/<session_id>)
    to avoid polluting any existing git repo in the user's workspace.
    """

    CHECKPOINT_BASE = Path.home() / '.cowork' / 'checkpoints'

    def __init__(self, workspace: Path):
        self.workspace = workspace.resolve()
        self.session: Optional[Session] = None
        self.CHECKPOINT_BASE.mkdir(parents=True, exist_ok=True)
        self._shutdown_handlers_registered = False

    def _git(self, *args) -> subprocess.CompletedProcess:
        """Run git with a separate checkpoint directory."""
        git_dir = self.CHECKPOINT_BASE / (self.session.id if self.session else 'default')
        cmd = ['git', f'--git-dir={git_dir}', f'--work-tree={self.workspace}', *args]
        return subprocess.run(cmd, capture_output=True, text=True)

    def create_session(self) -> Session:
        session = Session(
            id=uuid_mod.uuid4().hex,
            workspace=self.workspace,
            created_at=time.time(),
            last_activity=time.time(),
        )
        self.session = session
        self._init_checkpoint_repo()
        return session

    def _init_checkpoint_repo(self):
        git_dir = self.CHECKPOINT_BASE / self.session.id
        if not git_dir.exists():
            git_dir.mkdir(parents=True, exist_ok=True)
            self._git('init')
            self._git('add', '-A')
            self._git('commit', '-m', 'session start', '--allow-empty')

    def save_checkpoint(self, message: str = '') -> str:
        self._git('add', '-A')
        msg = f'checkpoint: {message}' if message else f'checkpoint: {time.time()}'
        self._git('commit', '-m', msg, '--allow-empty')
        result = self._git('rev-parse', 'HEAD')
        return result.stdout.strip()

    def restore_checkpoint(self, commit_hash: str):
        self._git('checkout', commit_hash, '--', '.')

    def get_changes_since(self, commit_hash: str) -> str:
        result = self._git('diff', '--stat', commit_hash, 'HEAD')
        return result.stdout

    def heartbeat(self):
        if self.session:
            self.session.last_activity = time.time()


class GracefulShutdown:
    """
    Registers cleanup handlers for SIGINT, SIGTERM, and atexit.
    Ensures browser, MCP connections, and session state are cleaned up
    even on unexpected termination.
    """

    def __init__(self, session_manager: SessionManager, tool_runtime: 'ToolRuntime',
                 audit_logger: AuditLogger):
        self.session_manager = session_manager
        self.tool_runtime = tool_runtime
        self.audit_logger = audit_logger
        self._shutting_down = False

    def register(self):
        signal.signal(signal.SIGINT, self._handle_signal)
        signal.signal(signal.SIGTERM, self._handle_signal)
        atexit.register(self._sync_cleanup)

    def _handle_signal(self, signum, frame):
        if self._shutting_down:
            return
        self._shutting_down = True

        loop = asyncio.get_event_loop()
        if loop.is_running():
            loop.create_task(self._async_cleanup())
        else:
            loop.run_until_complete(self._async_cleanup())

    async def _async_cleanup(self):
        try:
            self.session_manager.save_checkpoint('interrupted')
        except Exception:
            pass
        try:
            await self.tool_runtime.close()
        except Exception:
            pass
        try:
            await self.audit_logger.close()
        except Exception:
            pass

    def _sync_cleanup(self):
        """atexit handler (synchronous context)."""
        try:
            self.session_manager.save_checkpoint('shutdown')
        except Exception:
            pass
```

### Trash Cleanup

```python
def cleanup_trash(trash_dir: Path, max_age_days: int = 30):
    """Delete trash files older than max_age_days. Run on session start."""
    cutoff = time.time() - (max_age_days * 86400)
    for item in trash_dir.iterdir():
        try:
            if item.stat().st_mtime < cutoff:
                if item.is_dir():
                    shutil.rmtree(item)
                else:
                    item.unlink()
        except OSError:
            pass
```

---

## Context Engineering Strategy

**Purpose:** Manage the LLM's context window as a finite budget.

Reference: [Anthropic's "Effective Context Engineering for AI Agents" (Sep 2025)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).

The core principle: context is working memory. Fill it with the right information at each step, not everything accumulated so far.

### Token Budget

```python
@dataclass
class TokenBudget:
    max_tokens: int = 100_000
    system_prompt_tokens: int = 1_500
    tool_definitions_tokens: int = 2_000
    reserved_for_response: int = 4_000
    _used: int = 0

    @property
    def available(self) -> int:
        return self.max_tokens - self.system_prompt_tokens - self.tool_definitions_tokens - self.reserved_for_response - self._used

    @property
    def exhausted(self) -> bool:
        return self.available < 2_000

    def consume(self, tokens: int):
        self._used += tokens

    def count_tokens(self, text: str, model: str = '') -> int:
        """
        Count tokens using litellm's model-aware tokenizer.
        Falls back to a heuristic for unknown models.
        """
        try:
            return litellm.token_counter(model=model, text=text)
        except Exception:
            # Fallback: ~3.5 chars per token (conservative estimate)
            return len(text) // 3
```

### Context State

```python
class ContextState:
    """
    Structured state that lives OUTSIDE the chat history.

    This object is the primary mechanism for preserving information
    across context compaction cycles. When older messages are summarized
    and discarded, the data in ContextState survives intact.
    """

    def __init__(self):
        self.files_created: list[str] = []
        self.files_modified: list[str] = []
        self.files_deleted: list[str] = []
        self.current_url: Optional[str] = None
        self.current_step: int = 0
        self.errors: list[str] = []
        self.key_facts: list[str] = []
        self.original_goal: str = ''

    def update_from_result(self, tool: str, args: dict, result: dict):
        self.current_step += 1

        if tool == 'run_bash':
            cmd = args.get('command', '')
            if 'mkdir' in cmd or 'touch' in cmd or 'cp' in cmd:
                self.files_created.append(cmd.split()[-1] if cmd.split() else cmd)
            if result.get('exit_code', 0) != 0:
                self.errors.append(f'{cmd[:80]}: {result.get("stderr", "")[:100]}')
        elif tool == 'browser_goto':
            self.current_url = args.get('url')
        elif tool.startswith('mcp_'):
            if result.get('error'):
                self.errors.append(f'MCP {tool}: {result["error"][:100]}')

    def compact_summary(self) -> str:
        """Injected into the system prompt on every LLM call."""
        parts = [f'Step: {self.current_step}']
        if self.current_url:
            parts.append(f'Browser: {self.current_url}')
        if self.files_created:
            parts.append(f'Files created: {", ".join(self.files_created[-10:])}')
        if self.files_modified:
            parts.append(f'Files modified: {", ".join(self.files_modified[-10:])}')
        if self.errors:
            parts.append(f'Recent errors: {"; ".join(self.errors[-3:])}')
        if self.key_facts:
            parts.append(f'Key facts: {"; ".join(self.key_facts[-5:])}')
        return '\n'.join(parts)

    def summary(self) -> dict:
        return {
            'files_created': self.files_created,
            'files_modified': self.files_modified,
            'files_deleted': self.files_deleted,
            'current_url': self.current_url,
            'current_step': self.current_step,
            'errors': self.errors,
            'key_facts': self.key_facts,
        }
```

### Message Manager

```python
class MessageManager:
    """
    Manages LLM message history with semantic-preserving compaction.

    Compaction strategy (applied when message count exceeds max_messages):
    1. Tool results are truncated at insertion time (max 4000 chars each)
    2. Older messages are summarized, preserving:
       - Which tools were called and with what key arguments
       - Abbreviated results (first 150 chars)
       - ContextState data (files, errors, URLs, key facts)
    3. Recent messages are kept intact (last keep_recent messages)
    """

    def __init__(self, system_prompt: str, context_state: ContextState,
                 token_budget: TokenBudget, model: str = ''):
        self.system_prompt = system_prompt
        self.context_state = context_state
        self.budget = token_budget
        self.model = model
        self.messages: list[dict] = []
        self.max_messages = 40
        self.keep_recent = 12
        self.compacted_summary: Optional[str] = None

    def add_user_message(self, content: str):
        self.messages.append({'role': 'user', 'content': content})
        self._track_tokens(content)

    def add_assistant_message(self, message):
        msg_dict = message.model_dump() if hasattr(message, 'model_dump') else message
        self.messages.append(msg_dict)

    def add_tool_result(self, tool_call_id: str, result: dict):
        content = json.dumps(result, cls=SafeEncoder, ensure_ascii=False)
        if len(content) > 4_000:
            content = content[:3_000] + '\n...[truncated]...\n' + content[-500:]
        self.messages.append({
            'role': 'tool',
            'tool_call_id': tool_call_id,
            'content': content,
        })
        self._track_tokens(content)
        self._maybe_compact()

    def get_messages(self) -> list[dict]:
        result = [{'role': 'system', 'content': self._build_system_prompt()}]
        if self.compacted_summary:
            result.append({
                'role': 'user',
                'content': f'[CONTEXT SUMMARY]\n{self.compacted_summary}',
            })
        result.extend(self.messages)
        return result

    def _build_system_prompt(self) -> str:
        state_block = self.context_state.compact_summary()
        if state_block:
            return f'{self.system_prompt}\n\n[WORKSPACE STATE]\n{state_block}'
        return self.system_prompt

    def _track_tokens(self, text: str):
        tokens = self.budget.count_tokens(text, self.model)
        self.budget.consume(tokens)

    def _maybe_compact(self):
        if len(self.messages) <= self.max_messages:
            return

        old_messages = self.messages[:-self.keep_recent]
        self.messages = self.messages[-self.keep_recent:]

        # Build semantic summary that preserves what was done and what happened
        action_log = []
        for msg in old_messages:
            if msg.get('role') == 'assistant':
                tool_calls = msg.get('tool_calls', [])
                for tc in tool_calls:
                    func = tc.get('function', {})
                    name = func.get('name', '?')
                    args_preview = func.get('arguments', '')[:120]
                    action_log.append(f'- {name}({args_preview})')
            elif msg.get('role') == 'tool':
                content = msg.get('content', '')[:150]
                action_log.append(f'  result: {content}')

        state = self.context_state
        summary_parts = [
            f'Original goal: {state.original_goal}',
            '',
            'Completed actions:',
            *action_log[-30:],
            '',
            f'Files created: {", ".join(state.files_created[-10:])}',
            f'Files modified: {", ".join(state.files_modified[-10:])}',
        ]
        if state.errors:
            summary_parts.append(f'Errors encountered: {"; ".join(state.errors[-5:])}')
        if state.key_facts:
            summary_parts.append(f'Key facts: {"; ".join(state.key_facts[-5:])}')

        self.compacted_summary = '\n'.join(summary_parts)


class SafeEncoder(json.JSONEncoder):
    """Handles Path, datetime, bytes, and set serialization."""

    def default(self, obj):
        if isinstance(obj, Path):
            return str(obj)
        if hasattr(obj, 'isoformat'):
            return obj.isoformat()
        if isinstance(obj, bytes):
            return obj.decode('utf-8', errors='replace')
        if isinstance(obj, set):
            return list(obj)
        return super().default(obj)
```

---

## Feature Roadmap

Features are organized by priority tier. Each tier builds on the one above it.

### Foundation

The minimum viable product. A working CLI agent that can execute tasks.

| Layer | Scope |
|---|---|
| Sandbox | bubblewrap namespace isolation |
| Agent Core | LiteLLM library + ReAct loop + cost tracking |
| Tools | Bash (with safety intercepts) + MCP client |
| Security | Path validation, trash delete, default network deny |
| Observability | Rich CLI streaming, structured logging, cost display |
| Session | Single session, graceful shutdown |

**Foundation delivers:**
- `pip install opencowork` → set API key → run
- Execute tasks in bubblewrap sandbox
- Provider switching via LiteLLM (any model)
- File operations with safety intercepts
- MCP tool ecosystem access
- Real-time streaming output
- Cost tracking per session
- Graceful shutdown with state preservation

### Growth

Expanded capabilities and richer user experience.

| Addition | Description |
|---|---|
| Playwright browser | DOM-first web automation with route-level domain enforcement |
| Textual TUI | Permission prompts, progress tracking, audit viewer |
| Git checkpoints | Workspace state snapshots with separate git directory |
| Session resume | `--resume` flag for interrupted tasks |
| gVisor sandbox | Production-grade syscall interception |
| Plan-and-execute | Optimization layer for batch-style tasks |
| SQLite audit log | Persistent, queryable action history via aiosqlite |
| Config file | YAML/TOML config for models, sandbox tier, permissions, domains |

### Scale

Enterprise and advanced features.

| Addition | Description |
|---|---|
| Web dashboard UI | Real-time progress, session management, log viewer |
| Multi-session | Concurrent sessions with advisory file locking |
| Firecracker | Hardware-level microVM isolation for multi-tenant deployments |
| Vision fallback | Screenshot + VLM for canvas/anti-bot sites |
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

LLM API calls take 2-30 seconds. Browser page loads take 1-30 seconds. Synchronous execution means no streaming output, no heartbeats, no cancel/interrupt capability, no parallel tool execution. Full async from day 1: `asyncio` event loop, `playwright.async_api`, `aiosqlite`.

### Why MCP in Foundation?

MCP is the standard for agent-tool integration. Without MCP, every tool must be hand-coded and users cannot add their own tools. With MCP in Foundation:
- Filesystem, git, GitHub tools available immediately via `npx`
- Users can connect any MCP server
- Tool definitions generated automatically from MCP schemas
- ~200 lines of integration code for massive extensibility

**Known trade-off:** MCP tool results can be large and noisy, causing context window saturation and up to 236x token inflation (per 2025 research). Mitigation: truncate all MCP results, prefix tool descriptions with `[MCP:server]` to help the LLM attribute sources.

### Why Git for Checkpoints?

Computing file hashes on every checkpoint is O(n) over all files, painfully slow for large workspaces, and MD5 is cryptographically broken. Git provides O(changed files) incremental tracking, built-in diff/restore/history, free deduplication, and is already installed on every dev machine.

The checkpoint git directory is kept separate from the workspace (`~/.cowork/checkpoints/<session_id>`) so it does not conflict with the user's own git repo.

### Why DOM-First Browser Automation?

Screenshot-based browser automation (Claude Cowork's approach) requires a vision model API call for every interaction — slow and expensive. Playwright DOM automation gives direct element access via CSS selectors (instant, deterministic, free). Vision fallback is deferred to Growth tier for cases where DOM fails (canvas elements, anti-bot protections).

### Why ReAct over Plan-and-Execute as the Core Loop?

Plan-and-execute assumes the full execution path can be determined upfront. In practice, real tasks involve dynamic content (web pages change), unpredictable errors (disk full, permissions), and branching logic (different actions based on file contents). The LLM cannot reliably predict these at planning time.

ReAct handles this naturally: observe result, decide next action, repeat. The plan-and-execute optimization layer is available for tasks where upfront planning is clearly beneficial (batch file operations, known-path workflows), but it is not the default execution pattern.

---

## Edge Cases & Solutions

### EC-1: LLM Returns Malformed JSON

**Problem:** Tool call arguments contain invalid JSON.

**Solution:** Wrap `json.loads(tool_call.function.arguments)` in try/except. On failure, return an error to the LLM as a tool result so it can self-correct. The ReAct loop naturally handles this — the LLM sees the error and retries.

```python
try:
    args = json.loads(tool_call.function.arguments)
except json.JSONDecodeError:
    self.message_manager.add_tool_result(
        tool_call_id=tool_call.id,
        result={'error': 'Malformed tool call arguments. Please retry with valid JSON.'},
    )
    continue
```

### EC-2: Browser Page Never Reaches networkidle

**Problem:** SPAs with websockets, analytics, or live updates never reach `networkidle`.

**Solution:** Use `domcontentloaded` with a 30s timeout. This fires when HTML is parsed, which is sufficient for most interaction tasks.

### EC-3: shlex.split Fails on Malformed Commands

**Problem:** LLM generates command with unbalanced quotes. `shlex.split` raises `ValueError`.

**Solution:** Catch the exception and return a clean error to the LLM.

### EC-4: Command Injection via shell=True

**Problem:** `subprocess.run(command, shell=True)` allows arbitrary shell injection via `;`, `&&`, backticks, `$()`.

**Solution:** `shell=True` is never used anywhere in the codebase. All commands are parsed to lists via `shlex.split` and executed as `subprocess.run(cmd_list, shell=False)`. The sandbox layer also receives command lists.

### EC-5: browser_read Returns Megabytes

**Problem:** Raw HTML can be 500KB-5MB. Appending this to LLM messages destroys the context budget.

**Solution:**
1. Use `page.inner_text('body')` instead of `page.content()` (visible text only, ~90% smaller)
2. Truncate to 8,000 chars
3. If user needs full content, save to file and return the file path

### EC-6: Context Window Exhaustion on Long Tasks

**Problem:** After 10-15 tool calls, message history exceeds the context window.

**Solution:** `MessageManager` with:
1. Tool result truncation (cap at 4,000 chars per result)
2. Sliding window (keep last 12 messages, summarize older ones)
3. Semantic-preserving compaction (action log + ContextState data)
4. Token budget tracking (stop when exhausted)

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

### EC-11: Playwright Chromium Missing Dependencies

**Problem:** Minimal Linux installs lack Chromium dependencies (libgbm, libnss, etc.).

**Solution:** `playwright install --with-deps chromium` in setup. For Docker tiers, dependencies are pre-installed in the container image. For bubblewrap tier, bind-mount host's Chromium.

### EC-12: Trash Directory Fills Disk

**Problem:** Files moved to trash accumulate over time.

**Solution:** Run `cleanup_trash()` on session start: delete files older than 30 days. Track trash size in `--status` command.

### EC-13: Non-Serializable Context State

**Problem:** `json.dumps()` fails on Path objects, datetime, or bytes.

**Solution:** `SafeEncoder` (see Context Engineering section) handles Path, datetime, bytes, and set types.

### EC-14: Clicks Navigate to Disallowed Domains

**Problem:** The LLM clicks a link that redirects to a domain outside the allowlist. The `browser_goto` domain check only covers explicit navigation, not redirects or click-triggered navigations.

**Solution:** Playwright route interception on all `document` type requests. Every page navigation — whether from `goto()`, a redirect, or a click — is checked against the domain allowlist. Disallowed navigations are aborted at the network level.

### EC-15: MCP Tool Results Saturate Context

**Problem:** MCP servers can return enormous results (full database tables, large file contents). Research shows up to 236x token inflation from naive MCP integration.

**Solution:** All MCP tool results are truncated to 5,000 chars. Tool descriptions are prefixed with `[MCP:server]` for attribution. Large results should be written to a file in /workspace and the file path returned instead.

### EC-16: Chromium Memory Leaks in Long Sessions

**Problem:** Chromium's memory usage grows over time. In sessions with many page navigations, the browser process can consume gigabytes.

**Solution:** `BrowserTool.restart()` method kills and re-launches the browser. Trigger this when memory usage is concerning or between unrelated browsing tasks.

### EC-17: LLM Emits Multiple Parallel Tool Calls

**Problem:** The agent loop only processes the first tool call and ignores the rest.

**Solution:** The agent loop iterates over ALL tool calls in the response. Each call is executed, checked against security, and its result appended to the message history.

### EC-18: Cost Runaway from Agent Loops

**Problem:** A confused agent can loop indefinitely, burning through API credits.

**Solution:** `CostTracker` with a configurable `max_cost_per_session_usd` (default $5). The agent loop checks `exceeds_limit()` after every LLM call and stops if the budget is exhausted. Cost is displayed in the TUI.

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

### Mount Allowlist Strategy

The bubblewrap sandbox explicitly mounts only required system paths:
- `/usr`, `/bin`, `/sbin`, `/lib`, `/lib64` — read-only (executables and libraries)
- `/etc/resolv.conf`, `/etc/ssl`, `/etc/ca-certificates` — read-only (DNS and TLS)
- `/workspace` — read-write (user data)
- `/trash` — read-write (deleted files)
- `/tmp`, `/home`, `/run` — tmpfs (isolated, ephemeral)
- `/proc`, `/dev` — minimal sandbox-provided mounts

The host root filesystem, `/etc/passwd`, `/etc/shadow`, host `/proc`, and other users' home directories are never exposed to the sandbox.

### Threat Model

| Threat | Mitigation |
|---|---|
| LLM generates `rm -rf /` | `rm` intercepted → trash. Sandbox restricts writes to /workspace. Even bypassing the intercept (via Python, find -delete, etc.), the sandbox mount namespace prevents access outside /workspace. |
| LLM exfiltrates data via curl | Sandbox uses `--unshare-net` (no network by default). Browser is the only network path when enabled, controlled by route-level domain enforcement. |
| LLM reads ~/.ssh/id_rsa | Sandbox mounts only explicit paths. /home is a tmpfs. Host home directory is never bind-mounted. |
| Container escape (CVE-2019-5736) | gVisor intercepts syscalls in user-space kernel. No shared host kernel surface. |
| Prompt injection via web page | Browser content truncated. MCP results truncated and tagged as external. No automatic code execution from page content. |
| Malicious MCP server | MCP servers listed in config file, user-approved. Results treated as untrusted (truncated, prefixed). MCP servers run inside the same sandbox namespace. |
| Navigation to malicious domain | Playwright route interception checks domain allowlist on every document request, including redirects and click-triggered navigations. |
| Cost runaway | CostTracker enforces per-session spending limit. Agent loop halts when budget is exhausted. |

---

## Known Limitations

1. **Command-level safety intercepts are bypassable.** The `rm` → trash intercept and command blocklist can be circumvented via Python one-liners, `find -delete`, or other interpreters. The kernel-level sandbox is the real security boundary; the intercepts are a safety net for accidental damage, not adversarial resistance.

2. **LiteLLM has documented reliability issues at scale.** Token counting inaccuracies, cold start latency (~3s for serverless), and retry loop edge cases are documented by production users. For single-user workloads this is acceptable. A direct-SDK fallback path should be available for users who hit LiteLLM bugs.

3. **Context compaction is lossy.** Summarizing older messages discards details. Tasks exceeding ~30 turns may experience degraded agent performance as compacted history loses nuance. The ContextState object mitigates this by preserving structured facts across compaction cycles, but it cannot capture everything.

4. **Token counting for non-OpenAI models uses heuristics.** `litellm.token_counter()` supports most major providers but falls back to character-based estimation for obscure models. This can cause the token budget to underestimate or overestimate actual usage.

5. **Single-session only at Foundation tier.** No concurrent session support, no file locking, no shared state. Running multiple instances against the same workspace will cause conflicts.

6. **No vision/screenshot fallback at Foundation tier.** Browser automation relies entirely on DOM access. Canvas-rendered content, image-based CAPTCHAs, and aggressive anti-bot sites will not be handled.

7. **NFS/network mounts may not work with bubblewrap.** `--bind` relies on Linux mount namespaces, which have limited support for some network filesystem types. Local disk is recommended.

8. **TOCTOU in path validation.** `Path.resolve()` followed by a separate file operation has a theoretical race condition if symlinks are modified between the check and the access. The sandbox mount namespace mitigates this in practice by limiting what paths exist inside the namespace.

9. **MCP security depends on server trust.** MCP servers execute with whatever permissions the host process has (inside the sandbox). A malicious MCP server could theoretically perform actions within the sandbox namespace. Mitigation: only connect to reviewed/trusted MCP servers, and run them inside the same sandbox isolation.

10. **Playwright route interception cannot block sub-resource requests to disallowed domains** without breaking most websites (CDNs, fonts, analytics are cross-domain). Only document-level navigations are enforced. A compromised page could theoretically exfiltrate data via image/script requests to third-party domains, but it cannot access workspace data outside the browser context.
