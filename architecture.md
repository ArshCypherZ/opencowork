# Open Cowork: Complete Architecture & Implementation Specification

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Design Philosophy](#design-philosophy)
3. [Complete Architecture (10 Layers)](#complete-architecture)
4. [MVP vs 1.0 Feature Split](#mvp-vs-10-feature-split)
5. [Technical Decisions & Rationale](#technical-decisions--rationale)
6. [Edge Cases & Solutions](#edge-cases--solutions)
7. [Security Model](#security-model)
8. [Future Enhancements](#future-enhancements)

---

## Executive Summary

**Open Cowork** is a Linux-native, open-source alternative to Claude Cowork that provides an autonomous agent capable of:
- Managing files and directories
- Automating browser interactions
- Executing multi-step workflows
- Operating in a secure, sandboxed environment

**Key Differentiators:**
- **Provider Agnostic:** Works with any LLM (DeepSeek, Claude, GPT-4, Ollama)
- **Linux Native:** Built for Linux primitives (no macOS dependency or windows, currently claude cowork only supports windows and macos, so we are basically filling the gap)
- **Security First:** Containerized with explicit permissions
- **Observable:** Real-time progress tracking and audit trails
- **Recoverable:** Checkpoints and rollback capabilities

---

## Design Philosophy

### Core Principles

1. **Hybrid Intelligence over Pure Planning**
   - Use flexible task graphs for efficiency
   - Fall back to ReAct loops for adaptability
   - Never assume the world is deterministic

2. **Safety without Annoyance**
   - Auto-grant safe operations (read, copy, organize)
   - Confirm only high-risk operations (delete, encrypt, upload)
   - Use trash bins instead of permanent deletion

3. **Observability is Non-Negotiable**
   - Users must see what the agent is doing in real-time
   - Every action must be auditable
   - Provide pause/resume/cancel controls

4. **Recovery over Rollback**
   - Acknowledge that you can't undo reality (API calls, network requests)
   - Use checkpoints for context, not time-travel
   - Use trash bins for file recovery
   - Use browser state storage for session recovery

5. **DOM First, Vision as Fallback**
   - Playwright for 80% of web tasks (faster, cheaper, more reliable)
   - Vision models only when DOM fails (canvas elements, obscured UI)

---

## Complete Architecture

### Layer 0: Foundation (Container Infrastructure)

**Purpose:** Secure, resource-controlled execution environment

```yaml
Container Specification:
  Base Images (Layered):
    - opencowork:base (500MB)
      └─ Python 3.11 + Playwright + Bash essentials
    
    - opencowork:data-science (1.2GB)
      └─ base + pandas + numpy + scipy + matplotlib
    
    - opencowork:web-scraping (800MB)
      └─ base + selenium + beautifulsoup + requests
    
    - opencowork:media (1.5GB)
      └─ base + PIL + opencv + ffmpeg
  
  Security:
    - userns-remap: enabled (container root ≠ host root)
    - User mapping: Run as host UID/GID
    - No /etc/passwd mounting (security leak)
  
  Resources:
    - CPU limit: 2.0 cores
    - Memory limit: 4GB
    - PID limit: 256 processes
    - Disk: tmpfs for /tmp (auto-cleanup)
  
  Network:
    - Mode: bridge (isolated by default)
    - Allowlist: pypi.org, github.com, google.com
    - Firewall: Strict domain filtering
  
  Volumes:
    - /workspace (explicit user opt-in)
    - ~/.cowork/trash (safety net)
    - ~/.cowork/cache (model storage)
    - ~/.cowork/checkpoints (state storage)
  
  Health Monitoring:
    - CPU threshold: 90% sustained = unhealthy
    - Memory threshold: 3.5GB = restart warning
    - Heartbeat: Every 30s
    - Auto-restart: On resource exhaustion
```

**Container Selection Logic:**
```python
class SmartContainer:
    def select_image(self, prompt: str) -> str:
        """Detect required packages and use pre-built image"""
        if any(pkg in prompt.lower() for pkg in ['pandas', 'numpy', 'csv']):
            return 'opencowork:data-science'
        elif any(pkg in prompt.lower() for pkg in ['scrape', 'crawl', 'selenium']):
            return 'opencowork:web-scraping'
        elif any(pkg in prompt.lower() for pkg in ['image', 'video', 'photo']):
            return 'opencowork:media'
        else:
            return 'opencowork:base'
    
    def runtime_install(self, package: str):
        """Install package and cache the layer"""
        subprocess.run(['pip', 'install', package, '--break-system-packages'])
        new_layer = self.commit_container()
        self.cache_layer(package, new_layer)
```

---

### Layer 0.5: Session Coordinator

**Purpose:** Prevent concurrent file access conflicts

```python
class SessionCoordinator:
    """Manages multiple concurrent sessions"""
    
    def __init__(self):
        self.active_sessions: Dict[str, Session] = {}
        self.file_locks: Dict[Path, str] = {}  # Path -> Session ID
        self.lock_timeout = 300  # 5 minutes
    
    def request_access(self, session_id: str, path: Path, mode: str) -> bool:
        """
        Args:
            session_id: Unique session identifier
            path: File/directory path to access
            mode: 'read' or 'write'
        
        Returns:
            True if access granted, False if conflict
        """
        if mode == 'read':
            # Multiple readers allowed
            return True
        
        if mode == 'write':
            # Check for existing lock
            if path in self.file_locks:
                lock_holder = self.file_locks[path]
                lock_age = time.time() - self.active_sessions[lock_holder].lock_time
                
                # Stale lock (session died?)
                if lock_age > self.lock_timeout:
                    self.release_lock(path)
                else:
                    # Active conflict
                    return self._resolve_conflict(session_id, lock_holder, path)
            
            # Acquire lock
            self.file_locks[path] = session_id
            self.active_sessions[session_id].lock_time = time.time()
            return True
    
    def _resolve_conflict(self, new_session: str, existing_session: str, path: Path) -> bool:
        """Ask user to resolve conflict"""
        choice = user_prompt(f"""
        Session {existing_session} is currently working on {path}.
        
        Options:
        1. Wait for completion
        2. Take over (cancels other session)
        3. Work on different folder
        """)
        
        if choice == 1:
            # Block until lock released
            while path in self.file_locks:
                time.sleep(1)
            return self.request_access(new_session, path, 'write')
        
        elif choice == 2:
            # Force-cancel existing session
            self.cancel_session(existing_session)
            self.file_locks[path] = new_session
            return True
        
        else:
            return False
    
    def release_lock(self, path: Path):
        """Release file lock when operation completes"""
        if path in self.file_locks:
            del self.file_locks[path]
```

---

### Layer 1: The Brain (LLM Provider)

**Purpose:** Provider-agnostic LLM interface with failover

```python
class ModelRegistry:
    """Manages multiple LLM providers with A/B testing and fallbacks"""
    
    def __init__(self):
        self.models = {
            'fast': [
                ('deepseek-v4', 1.0),           # Primary (100% traffic)
                ('deepseek-v3', 0.0),           # Backup (0% traffic)
            ],
            'smart': [
                ('claude-sonnet-4.5', 0.8),     # Primary (80% traffic)
                ('claude-sonnet-4', 0.2),       # Canary (20% traffic)
            ],
            'vision': [
                ('qwen-vl-2', 0.7),             # Primary
                ('gpt-4o', 0.3),                # Fallback
            ]
        }
        self.cache_dir = Path.home() / '.cowork' / 'models'
        self.failure_counts = defaultdict(int)
    
    def get_model(self, category: str, session_id: str = None) -> str:
        """
        Returns model name using weighted selection
        Implements circuit breaker for failed models
        """
        candidates = self.models[category]
        
        # Filter out failed models (circuit breaker)
        available = [(m, w) for m, w in candidates 
                     if self.failure_counts[m] < 3]
        
        if not available:
            # All models failed, use cache
            return self.get_cached_model(category)
        
        # Weighted random selection (for A/B testing)
        total_weight = sum(w for _, w in available)
        r = random.random() * total_weight
        
        cumulative = 0
        for model, weight in available:
            cumulative += weight
            if r <= cumulative:
                try:
                    self.test_model(model)
                    return model
                except ModelUnavailable:
                    self.failure_counts[model] += 1
                    continue
        
        # All attempts failed
        return self.get_cached_model(category)
    
    def get_cached_model(self, category: str) -> str:
        """Use locally cached model when APIs unavailable"""
        cache_path = self.cache_dir / category
        if cache_path.exists():
            return f"local:{cache_path}"
        raise NoAvailableModel(f"No cache for {category}")
    
    def record_success(self, model: str):
        """Reset failure counter on successful call"""
        self.failure_counts[model] = 0
    
    def record_failure(self, model: str):
        """Increment failure counter (circuit breaker)"""
        self.failure_counts[model] += 1

class LLMProvider:
    """Main interface to LiteLLM proxy"""
    
    def __init__(self):
        self.proxy_url = os.getenv('LITELLM_URL', 'http://localhost:4000')
        self.registry = ModelRegistry()
        self.request_timeout = 60
    
    def completion(self, messages: List[Dict], category: str = 'fast', 
                   tools: List[Dict] = None, stream: bool = False) -> Dict:
        """
        Call LLM with automatic model selection and failover
        
        Args:
            messages: Chat messages in OpenAI format
            category: 'fast', 'smart', or 'vision'
            tools: Optional tool definitions for function calling
            stream: Whether to stream response
        
        Returns:
            Response dict with choices, usage, etc.
        """
        model = self.registry.get_model(category)
        
        payload = {
            'model': model,
            'messages': messages,
            'tools': tools,
            'stream': stream
        }
        
        try:
            response = requests.post(
                f"{self.proxy_url}/chat/completions",
                json=payload,
                timeout=self.request_timeout
            )
            response.raise_for_status()
            
            self.registry.record_success(model)
            return response.json()
        
        except (RequestException, HTTPError) as e:
            self.registry.record_failure(model)
            
            # Retry with fallback model
            fallback_model = self.registry.get_model(category)
            if fallback_model != model:
                payload['model'] = fallback_model
                response = requests.post(
                    f"{self.proxy_url}/chat/completions",
                    json=payload,
                    timeout=self.request_timeout
                )
                return response.json()
            
            raise LLMProviderError(f"All models failed for {category}") from e
```

---

### Layer 2: The Orchestrator (Hybrid Planning + ReAct)

**Purpose:** Intelligent task execution with planning and adaptation

```python
class ContextManager:
    """Shared state between tools and steps"""
    
    def __init__(self):
        self.state = {
            'files_created': [],
            'files_modified': [],
            'files_deleted': [],
            'current_url': None,
            'browser_cookies': None,
            'last_error': None,
            'clipboard': None,
            'environment_vars': {},
        }
        self.history: List[Dict] = []
    
    def update(self, key: str, value: Any):
        """Update context state and log change"""
        old_value = self.state.get(key)
        self.state[key] = value
        
        self.history.append({
            'timestamp': time.time(),
            'action': 'context_update',
            'key': key,
            'old_value': old_value,
            'new_value': value
        })
    
    def get(self, key: str, default=None):
        return self.state.get(key, default)
    
    def serialize(self) -> str:
        """Serialize context for LLM consumption"""
        return json.dumps(self.state, indent=2)

class TaskGraph:
    """Flexible task graph with conditional branching"""
    
    def __init__(self, steps: List[Dict]):
        self.steps = steps
        self.current_step = 0
        self.completed = []
        self.failed = []
    
    def next_step(self) -> Optional[Dict]:
        """Get next step considering conditionals"""
        if self.current_step >= len(self.steps):
            return None
        
        step = self.steps[self.current_step]
        self.current_step += 1
        return step
    
    def adapt_step(self, step: Dict, result: Any) -> Dict:
        """
        Modify step based on result (handle .zip vs .pdf case)
        
        Example:
            step = {'action': 'extract', 'expect': ['pdf', 'zip']}
            result = {'type': 'zip', 'path': '/tmp/file.zip'}
            → Returns: {'action': 'unzip', 'target': '/tmp/file.zip'}
        """
        if 'expect' in step and 'next_if' in step:
            result_type = result.get('type')
            if result_type in step['expect']:
                fallback_key = f"next_if_{result_type}"
                if fallback_key in step:
                    return step[fallback_key]
        
        return step
    
    def insert_steps(self, new_steps: List[Dict], after_current: bool = True):
        """Insert steps dynamically (for error recovery)"""
        insert_pos = self.current_step if after_current else self.current_step - 1
        self.steps = self.steps[:insert_pos] + new_steps + self.steps[insert_pos:]

class HybridOrchestrator:
    """
    Main orchestration engine
    Combines:
      - Upfront planning (task graph)
      - ReAct execution (adaptation)
      - Context management (shared state)
    """
    
    def __init__(self):
        self.llm = LLMProvider()
        self.context = ContextManager()
        self.tools = ToolRegistry(self.context)
        self.progress = ProgressTracker()
        self.session_id = generate_session_id()
    
    def execute_task(self, user_prompt: str, max_turns: int = 20) -> Dict:
        """
        Main execution loop
        
        Flow:
          1. Generate flexible plan (task graph)
          2. Execute steps with ReAct adaptation
          3. Handle errors with re-planning
          4. Return final result
        """
        print(f"Starting task: {user_prompt}")
        
        # Phase 1: Planning (One LLM call)
        plan = self._generate_plan(user_prompt)
        task_graph = TaskGraph(plan['steps'])
        
        messages = [
            {
                'role': 'system',
                'content': self._get_system_prompt()
            },
            {
                'role': 'user',
                'content': user_prompt
            }
        ]
        
        # Phase 2: Execution (ReAct Loop)
        for turn in range(max_turns):
            # Get next step from graph
            step = task_graph.next_step()
            
            if step is None:
                print("Task completed (all steps done)")
                break
            
            # Execute step
            try:
                result = self._execute_step(step, messages) # another LLM call
                
                # Check if we need to adapt the next step
                if result.get('needs_adaptation'):
                    adapted_step = task_graph.adapt_step(step, result)
                    task_graph.insert_steps([adapted_step])
                
                # Continue to next step
                continue
            
            except UnexpectedError as e:
                # Step failed unexpectedly - replan
                print(f"Unexpected error: {e}")
                
                # Ask LLM to replan from this point (one more LLM call on failure)
                new_plan = self._replan(step, e, messages)
                task_graph.insert_steps(new_plan['steps'])
        
        return {
            'status': 'completed' if step is None else 'max_turns',
            'context': self.context.state,
            'history': self.context.history
        }
    
    def _generate_plan(self, prompt: str) -> Dict:
        """
        Generate flexible task graph upfront
        Uses 'smart' model for better planning
        """
        planning_prompt = f"""
        Create a flexible task graph for: {prompt}
        
        Return JSON with format:
        {{
          "steps": [
            {{
              "action": "download",
              "tool": "browser",
              "expect": ["pdf", "zip"],
              "next_if_zip": {{"action": "unzip", "tool": "bash"}}
            }}
          ]
        }}
        
        Include conditional branches for expected variations.
        """
        
        response = self.llm.completion(
            messages=[{'role': 'user', 'content': planning_prompt}],
            category='smart'
        )
        
        content = response['choices'][0]['message']['content']
        # Parse JSON from response (may need cleanup)
        plan = json.loads(self._extract_json(content))
        return plan
    
    def _execute_step(self, step: Dict, messages: List[Dict]) -> Dict:
        """
        Execute a single step using native tool calling
        
        This is where the ReAct loop happens - the LLM sees
        the step, decides which tool to use, and we execute it
        """
        # Add step to conversation
        messages.append({
            'role': 'user',
            'content': f"Execute this step: {json.dumps(step)}\nContext: {self.context.serialize()}"
        })
        
        # Call LLM with tool definitions
        response = self.llm.completion(
            messages=messages,
            category='fast',
            tools=self.tools.get_definitions()
        )
        
        msg = response['choices'][0]['message']
        messages.append(msg)
        
        # Execute tool calls
        if msg.get('tool_calls'):
            for tool_call in msg['tool_calls']:
                func_name = tool_call['function']['name']
                args = json.loads(tool_call['function']['arguments'])
                
                print(f"    {func_name}({args})")
                
                # Execute via tool registry
                result = self.tools.execute(func_name, **args)
                
                # Add result to conversation
                messages.append({
                    'role': 'tool',
                    'tool_call_id': tool_call['id'],
                    'content': str(result)
                })
                
                return result
        
        # No tool call - LLM wants to talk to user
        return {'type': 'message', 'content': msg.get('content')}
    
    def _replan(self, failed_step: Dict, error: Exception, messages: List[Dict]) -> Dict:
        """
        Ask LLM to generate new plan when unexpected failure occurs
        """
        replan_prompt = f"""
        The following step failed:
        {json.dumps(failed_step)}
        
        Error: {str(error)}
        
        Current context: {self.context.serialize()}
        
        Generate a recovery plan (new steps to try).
        """
        
        response = self.llm.completion(
            messages=messages + [{'role': 'user', 'content': replan_prompt}],
            category='smart'
        )
        
        content = response['choices'][0]['message']['content']
        return json.loads(self._extract_json(content))
    
    def _get_system_prompt(self) -> str:
        return """You are Open Cowork, a Linux automation agent.
        
        You have access to:
        - bash: Execute shell commands, manage files
        - browser: Navigate web pages, fill forms, click buttons
        - vision: Analyze screenshots when DOM automation fails
        
        Strategy:
        1. Break tasks into steps
        2. Use bash for file operations (faster, more reliable)
        3. Use browser (Playwright) for web tasks
        4. Only use vision when DOM elements are inaccessible
        5. When a step fails, analyze the error and adapt
        
        Safety rules:
        - Never use 'rm' directly (use trash)
        - Confirm before destructive operations on user files
        - All operations are scoped to /workspace
        """
    
    @staticmethod
    def _extract_json(text: str) -> str:
        """Extract JSON from markdown code blocks"""
        if '```json' in text:
            return text.split('```json')[1].split('```')[0].strip()
        elif '```' in text:
            return text.split('```')[1].split('```')[0].strip()
        return text.strip()
```

---

### Layer 3: Session Manager (Runtime Lifecycle)

**Purpose:** Manage session lifecycle, resources, and health

```python
class SessionManager:
    """
    Manages Docker container sessions
    Monitors resources, handles timeouts, enables resumption
    """
    
    def __init__(self):
        self.sessions: Dict[str, SessionState] = {}
        self.idle_timeout = 30 * 60  # 30 minutes
        self.max_session_time = 4 * 3600  # 4 hours
        self.resource_check_interval = 30  # seconds
        
    def create_session(self, user_id: str, workspace: Path) -> str:
        """
        Spin up a new Docker container for this session
        
        Returns:
            session_id
        """
        session_id = f"{user_id}_{int(time.time())}"
        
        # Create container
        container = docker_client.containers.run(
            image=self._select_image(),
            user=f"{os.getuid()}:{os.getgid()}",
            volumes={
                str(workspace): {'bind': '/workspace', 'mode': 'rw'},
                str(Path.home() / '.cowork' / 'trash'): {'bind': '/home/agent/.cowork/trash', 'mode': 'rw'},
            },
            environment={
                'LITELLM_URL': 'http://litellm:4000',
                'SESSION_ID': session_id
            },
            network_mode='bridge',
            detach=True,
            mem_limit='4g',
            cpu_count=2,
            pids_limit=256
        )
        
        state = SessionState(
            session_id=session_id,
            container_id=container.id,
            created_at=time.time(),
            last_activity=time.time(),
            workspace=workspace,
            status='active'
        )
        
        self.sessions[session_id] = state
        
        # Start monitoring thread
        threading.Thread(target=self._monitor_session, args=(session_id,), daemon=True).start()
        
        return session_id
    
    def _monitor_session(self, session_id: str):
        """Background thread to monitor session health"""
        while session_id in self.sessions:
            state = self.sessions[session_id]
            
            # Check idle timeout
            idle_time = time.time() - state.last_activity
            if idle_time > self.idle_timeout:
                print(f"    Session {session_id} idle for {idle_time:.0f}s, terminating...")
                self.terminate_session(session_id, reason='idle_timeout')
                break
            
            # Check max session time
            total_time = time.time() - state.created_at
            if total_time > self.max_session_time:
                print(f"    Session {session_id} exceeded max time ({total_time:.0f}s), terminating...")
                self.terminate_session(session_id, reason='max_time')
                break
            
            # Check resource usage
            container = docker_client.containers.get(state.container_id)
            stats = container.stats(stream=False)
            
            memory_usage = stats['memory_stats']['usage']
            memory_limit = stats['memory_stats']['limit']
            memory_percent = (memory_usage / memory_limit) * 100
            
            if memory_percent > 95:
                print(f"    Warning: Session {session_id} memory critical ({memory_percent:.1f}%), restarting...")
                self.restart_session(session_id, preserve_state=True)
            
            time.sleep(self.resource_check_interval)
    
    def heartbeat(self, session_id: str):
        """Update last activity timestamp"""
        if session_id in self.sessions:
            self.sessions[session_id].last_activity = time.time()
    
    def terminate_session(self, session_id: str, reason: str = 'user_request'):
        """Clean shutdown of session"""
        if session_id not in self.sessions:
            return
        
        state = self.sessions[session_id]
        
        # Save state to disk for potential resume
        self._save_session_state(session_id)
        
        # Stop and remove container
        container = docker_client.containers.get(state.container_id)
        container.stop(timeout=10)
        container.remove()
        
        # Update state
        state.status = 'terminated'
        state.end_reason = reason
        
        # Log termination
        print(f"    Warning: Session {session_id} terminated: {reason}")
    
    def restart_session(self, session_id: str, preserve_state: bool = True):
        """
        Restart container (for resource exhaustion)
        Optionally preserve state
        """
        if preserve_state:
            checkpoint = self._save_session_state(session_id)
        
        # Kill old container
        old_state = self.sessions[session_id]
        old_container = docker_client.containers.get(old_state.container_id)
        old_container.kill()
        old_container.remove()
        
        # Start new container
        new_session = self.create_session(old_state.user_id, old_state.workspace)
        
        if preserve_state:
            self._restore_session_state(new_session, checkpoint)
        
        return new_session
    
    def _save_session_state(self, session_id: str) -> Dict:
        """Serialize session state to disk"""
        state = self.sessions[session_id]
        checkpoint = {
            'session_id': session_id,
            'context': state.context.state if hasattr(state, 'context') else {},
            'history': state.context.history if hasattr(state, 'context') else [],
            'timestamp': time.time()
        }
        
        checkpoint_path = Path.home() / '.cowork' / 'checkpoints' / f"{session_id}.json"
        checkpoint_path.parent.mkdir(parents=True, exist_ok=True)
        checkpoint_path.write_text(json.dumps(checkpoint, indent=2))
        
        return checkpoint
    
    def _restore_session_state(self, session_id: str, checkpoint: Dict):
        """Restore session state from checkpoint"""
        state = self.sessions[session_id]
        state.context = ContextManager()
        state.context.state = checkpoint['context']
        state.context.history = checkpoint['history']

@dataclass
class SessionState:
    session_id: str
    container_id: str
    created_at: float
    last_activity: float
    workspace: Path
    status: str  # 'active', 'paused', 'terminated'
    context: Optional[ContextManager] = None
    end_reason: Optional[str] = None
```

---

### Layer 4: Tool Registry (Execution Layer)

**Purpose:** Tool definitions and execution with safety checks

```python
class ToolRegistry:
    """
    Central registry of available tools
    Provides OpenAI-compatible tool definitions
    Executes tool calls with safety checks
    """
    
    def __init__(self, context: ContextManager):
        self.context = context
        self.security = SecurityManager()
        self.bash = BashREPL(context, self.security)
        self.browser = BrowserAutomation(context, self.security)
        self.vision = VisionModel(context)
    
    def get_definitions(self) -> List[Dict]:
        """
        Return OpenAI-compatible tool definitions
        These are passed to the LLM for function calling
        """
        return [
            {
                'type': 'function',
                'function': {
                    'name': 'run_bash',
                    'description': 'Execute a bash command in the secure workspace. Use for file operations, git, package installation, etc.',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'command': {
                                'type': 'string',
                                'description': 'The bash command to execute'
                            },
                            'working_dir': {
                                'type': 'string',
                                'description': 'Working directory (default: /workspace)'
                            }
                        },
                        'required': ['command']
                    }
                }
            },
            {
                'type': 'function',
                'function': {
                    'name': 'browser_goto',
                    'description': 'Navigate to a URL',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'url': {'type': 'string', 'description': 'URL to visit'}
                        },
                        'required': ['url']
                    }
                }
            },
            {
                'type': 'function',
                'function': {
                    'name': 'browser_click',
                    'description': 'Click an element on the page using CSS selector',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'selector': {'type': 'string', 'description': 'CSS selector'},
                            'timeout': {'type': 'integer', 'description': 'Wait timeout in ms'}
                        },
                        'required': ['selector']
                    }
                }
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
                            'text': {'type': 'string'}
                        },
                        'required': ['selector', 'text']
                    }
                }
            },
            {
                'type': 'function',
                'function': {
                    'name': 'browser_read',
                    'description': 'Extract text content from the page',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'selector': {'type': 'string', 'description': 'CSS selector (optional, reads entire page if not provided)'}
                        }
                    }
                }
            },
            {
                'type': 'function',
                'function': {
                    'name': 'vision_analyze',
                    'description': 'Analyze a screenshot using vision model (fallback when DOM fails)',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'task': {'type': 'string', 'description': 'What to look for in the image'}
                        },
                        'required': ['task']
                    }
                }
            },
            {
                'type': 'function',
                'function': {
                    'name': 'vision_click',
                    'description': 'Click coordinates on screen using vision (use when selector fails)',
                    'parameters': {
                        'type': 'object',
                        'properties': {
                            'description': {'type': 'string', 'description': 'What element to click'}
                        },
                        'required': ['description']
                    }
                }
            }
        ]
    
    def execute(self, func_name: str, **kwargs) -> Any:
        """
        Execute a tool call
        Routes to appropriate handler
        """
        # Route to appropriate tool
        if func_name == 'run_bash':
            return self.bash.execute(kwargs['command'], kwargs.get('working_dir', '/workspace'))
        
        elif func_name.startswith('browser_'):
            action = func_name.replace('browser_', '')
            return self.browser.execute(action, **kwargs)
        
        elif func_name.startswith('vision_'):
            action = func_name.replace('vision_', '')
            return self.vision.execute(action, **kwargs)
        
        else:
            return {'error': f'Unknown tool: {func_name}'}

class BashREPL:
    """
    Bash execution with safety intercepts
    """
    
    def __init__(self, context: ContextManager, security: SecurityManager):
        self.context = context
        self.security = security
        self.history: List[str] = []
    
    def execute(self, command: str, working_dir: str = '/workspace') -> Dict:
        """
        Execute bash command with safety checks
        
        Returns:
            {'stdout': str, 'stderr': str, 'exit_code': int}
        """
        # Parse command
        cmd_parts = shlex.split(command)
        
        # Safety intercepts
        if cmd_parts[0] == 'rm':
            return self._safe_delete(cmd_parts[1:])
        
        elif cmd_parts[0] == 'dd':
            return {'error': 'dd is not allowed (potential data destruction)'}
        
        # Execute command
        try:
            result = subprocess.run(
                command,
                shell=True,
                capture_output=True,
                text=True,
                timeout=30,
                cwd=working_dir
            )
            
            # Log to history
            self.history.append(command)
            
            # Update context with file changes
            if cmd_parts[0] in ['touch', 'cp', 'mv', 'mkdir']:
                self._update_context_files(cmd_parts)
            
            return {
                'stdout': result.stdout,
                'stderr': result.stderr,
                'exit_code': result.returncode
            }
        
        except subprocess.TimeoutExpired:
            return {'error': 'Command timeout (30s limit)'}
        
        except Exception as e:
            return {'error': str(e)}
    
    def _safe_delete(self, targets: List[str]) -> Dict:
        """
        Intercept rm and use trash instead
        """
        results = []
        for target in targets:
            result = self.security.safe_delete(target)
            results.append(result)
        
        # Update context
        self.context.update('files_deleted', 
                          self.context.get('files_deleted', []) + targets)
        
        return {'stdout': '\n'.join(results)}
    
    def _update_context_files(self, cmd_parts: List[str]):
        """Update context with file operations"""
        if cmd_parts[0] == 'touch':
            self.context.update('files_created',
                              self.context.get('files_created', []) + [cmd_parts[1]])

class BrowserAutomation:
    """
    Playwright-based browser automation
    Falls back to vision when DOM fails
    """
    
    def __init__(self, context: ContextManager, security: SecurityManager):
        self.context = context
        self.security = security
        self.playwright = None
        self.browser = None
        self.page = None
        self.checkpoints: List[Dict] = []
    
    def _ensure_browser(self):
        """Lazy initialization of Playwright"""
        if self.playwright is None:
            from playwright.sync_api import sync_playwright
            self.playwright = sync_playwright().start()
            self.browser = self.playwright.chromium.launch(headless=True)
            self.page = self.browser.new_page()
    
    def execute(self, action: str, **kwargs) -> Dict:
        """
        Execute browser action with error handling
        """
        self._ensure_browser()
        
        try:
            # Save checkpoint before action
            self._save_checkpoint()
            
            if action == 'goto':
                return self._goto(kwargs['url'])
            elif action == 'click':
                return self._click(kwargs['selector'], kwargs.get('timeout', 30000))
            elif action == 'type':
                return self._type(kwargs['selector'], kwargs['text'])
            elif action == 'read':
                return self._read(kwargs.get('selector'))
            else:
                return {'error': f'Unknown action: {action}'}
        
        except PlaywrightTimeoutError as e:
            # Try vision fallback
            print(f"    Warning: Playwright timeout, trying vision fallback...{e}")
            return self._vision_fallback(action, kwargs)
        
        except Exception as e:
            return {'error': str(e), 'checkpoint_available': bool(self.checkpoints)}
    
    def _goto(self, url: str) -> Dict:
        """Navigate to URL"""
        # Check network permissions
        domain = urllib.parse.urlparse(url).netloc
        if not self.security.check_network_permission(domain):
            return {'error': f'Domain not in allowlist: {domain}'}
        
        self.page.goto(url, wait_until='networkidle')
        self.context.update('current_url', url)
        
        return {'status': 'success', 'url': self.page.url}
    
    def _click(self, selector: str, timeout: int) -> Dict:
        """Click element"""
        self.page.click(selector, timeout=timeout)
        return {'status': 'success', 'selector': selector}
    
    def _type(self, selector: str, text: str) -> Dict:
        """Type into input"""
        self.page.fill(selector, text)
        return {'status': 'success', 'selector': selector}
    
    def _read(self, selector: Optional[str]) -> Dict:
        """Extract text"""
        if selector:
            text = self.page.text_content(selector)
        else:
            text = self.page.content()
        
        return {'text': text}
    
    def _save_checkpoint(self):
        """Save browser state for recovery"""
        checkpoint = {
            'url': self.page.url,
            'cookies': self.page.context.cookies(),
            'local_storage': self.page.evaluate('() => JSON.stringify(localStorage)'),
            'timestamp': time.time()
        }
        self.checkpoints.append(checkpoint)
        
        # Save to disk
        checkpoint_path = Path.home() / '.cowork' / 'browser_checkpoints'
        checkpoint_path.mkdir(parents=True, exist_ok=True)
        
        with open(checkpoint_path / f"{int(time.time())}.json", 'w') as f:
            json.dump(checkpoint, f)
        
        # Update context
        self.context.update('browser_cookies', checkpoint['cookies'])
    
    def restore_checkpoint(self, checkpoint_id: Optional[int] = None):
        """Restore browser state from checkpoint"""
        if checkpoint_id is None:
            checkpoint = self.checkpoints[-1]
        else:
            checkpoint_path = Path.home() / '.cowork' / 'browser_checkpoints' / f"{checkpoint_id}.json"
            with open(checkpoint_path) as f:
                checkpoint = json.load(f)
        
        # Restore state
        self.page.goto(checkpoint['url'])
        self.page.context.add_cookies(checkpoint['cookies'])
        self.page.evaluate(f"localStorage = {checkpoint['local_storage']}")
    
    def _vision_fallback(self, action: str, kwargs: Dict) -> Dict:
        """
        When Playwright fails, use vision model
        """
        screenshot = self.page.screenshot()
        
        vision_prompt = f"""
        The DOM-based {action} failed with selector: {kwargs.get('selector')}
        
        Task: {action} with args {kwargs}
        
        Please analyze this screenshot and provide coordinates to click,
        or explain why the action cannot be completed.
        """
        
        # Call vision model (implemented in VisionModel class)
        from vision import VisionModel
        vision = VisionModel(self.context)
        return vision.analyze_and_act(screenshot, vision_prompt)

class VisionModel:
    """Vision-based fallback for when DOM automation fails"""
    
    def __init__(self, context: ContextManager):
        self.context = context
        self.llm = LLMProvider()
    
    def execute(self, action: str, **kwargs) -> Dict:
        if action == 'analyze':
            return self.analyze_screenshot(kwargs['task'])
        elif action == 'click':
            return self.find_and_click(kwargs['description'])
        else:
            return {'error': f'Unknown vision action: {action}'}
    
    def analyze_screenshot(self, task: str) -> Dict:
        """
        Take screenshot and analyze with vision model
        """
        # Take screenshot (assumes browser is active)
        screenshot_path = f"/tmp/screenshot_{int(time.time())}.png"
        # Screenshot taken by browser automation
        
        # Encode as base64
        with open(screenshot_path, 'rb') as f:
            image_data = base64.b64encode(f.read()).decode()
        
        # Call vision model
        response = self.llm.completion(
            messages=[{
                'role': 'user',
                'content': [
                    {'type': 'text', 'text': task},
                    {'type': 'image_url', 'image_url': {'url': f'data:image/png;base64,{image_data}'}}
                ]
            }],
            category='vision'
        )
        
        return {
            'analysis': response['choices'][0]['message']['content']
        }
    
    def find_and_click(self, description: str) -> Dict:
        """
        Find element in screenshot and return coordinates
        """
        analysis = self.analyze_screenshot(f"Find {description} and return X,Y coordinates")
        
        # Parse coordinates from response
        # Format expected: "coordinates: (X, Y)"
        import re
        match = re.search(r'\((\d+),\s*(\d+)\)', analysis['analysis'])
        
        if match:
            x, y = int(match.group(1)), int(match.group(2))
            # Use pyautogui or similar to click
            # (This requires display server access - container needs X11/Wayland)
            return {'status': 'success', 'coordinates': (x, y)}
        else:
            return {'error': 'Could not find element', 'analysis': analysis['analysis']}
```

---

### Layer 6: Security Manager

**Purpose:** Permission management and safety enforcement

```python
class SecurityManager:
    """
    Manages permissions and safety checks
    Implements:
      - File path validation
      - Network domain allowlist
      - Trash bin for safe delete
      - Intent-based permission model
    """
    
    def __init__(self):
        self.trash_path = Path('/home/agent/.cowork/trash')
        self.trash_path.mkdir(parents=True, exist_ok=True)
        
        # Allowed file roots
        self.allowed_roots = [
            Path('/workspace'),
            Path('/tmp'),
            self.trash_path
        ]
        
        # Network allowlist
        self.allowed_domains = {
            'pypi.org',
            'github.com',
            'google.com',
            'gmail.com',
            'githubusercontent.com',
        }
        
        # Session permissions (granted by user)
        self.session_grants: Set[Path] = set()
        
        # High-risk operations that require confirmation
        self.high_risk_operations = {
            'delete', 'encrypt', 'upload', 'exec'
        }
    
    def validate_path(self, path_str: str) -> bool:
        """
        Check if path is within allowed roots
        
        Returns:
            True if path is safe, False otherwise
        """
        try:
            path = Path(path_str).resolve()
            return any(path.is_relative_to(root) for root in self.allowed_roots)
        except (ValueError, RuntimeError):
            return False
    
    def check_file_permission(self, operation: str, path: Path) -> bool:
        """
        Intent-based permission check
        
        Args:
            operation: 'read', 'write', 'delete', 'encrypt', etc.
            path: Path to check
        
        Returns:
            True if allowed, False if denied
        """
        # Always allow safe operations
        if operation in ['read', 'list']:
            return True
        
        # Check if path is within session grants
        if any(path.is_relative_to(grant) for grant in self.session_grants):
            # Still check for high-risk operations
            if operation in self.high_risk_operations:
                return self.confirm_with_preview(operation, path)
            return True
        
        # Not in session grants - need user approval
        return self.request_permission(operation, path)
    
    def request_permission(self, operation: str, path: Path) -> bool:
        """
        Ask user for permission
        """
        message = f"Allow {operation} on {path}?"
        
        # In production, this would show a UI prompt TODO
        if not self.validate_path(str(path)):
            print(f"    Denied: {operation} on {path} (outside workspace)")
            return False
        
        # Simulate user approval
        choice = input(f"{message} [y/N]: ")
        return choice.lower() == 'y'
    
    def confirm_with_preview(self, operation: str, path: Path) -> bool:
        """
        High-risk operations show preview of what will happen
        """
        if operation == 'delete':
            # Count files that will be affected
            if path.is_dir():
                file_count = sum(1 for _ in path.rglob('*') if _.is_file())
                total_size = sum(f.stat().st_size for f in path.rglob('*') if f.is_file())
                
                preview = f"""
                About to delete directory: {path}
                  - {file_count} files
                  - {total_size / 1e6:.1f} MB total
                
                Files will be moved to trash (recoverable).
                """
            else:
                preview = f"About to delete: {path} ({path.stat().st_size / 1e6:.1f} MB)"
            
            print(preview)
            choice = input("Confirm [y/N]: ")
            return choice.lower() == 'y'
        
        # Other high-risk operations
        return self.request_permission(operation, path)
    
    def safe_delete(self, path_str: str) -> str:
        """
        Move to trash instead of permanent delete
        
        Returns:
            Status message
        """
        if not self.validate_path(path_str):
            return f"Error: Permission denied (path outside workspace): {path_str}"
        
        src = Path(path_str)
        if not src.exists():
            return f"Error: File not found: {path_str}"
        
        # Create trash destination with timestamp
        timestamp = int(time.time())
        dst = self.trash_path / f"{src.name}.{timestamp}"
        
        try:
            shutil.move(str(src), str(dst))
            return f"Moved to trash: {dst}"
        except Exception as e:
            return f"Error: {e}"
    
    def check_network_permission(self, domain: str) -> bool:
        """
        Check if network request to domain is allowed
        
        Returns:
            True if allowed, False if blocked
        """
        # Check allowlist
        if domain in self.allowed_domains:
            return True
        
        # Check if subdomain of allowed domain
        for allowed in self.allowed_domains:
            if domain.endswith(f".{allowed}"):
                return True
        
        # Unknown domain - ask user
        print(f"    Security Alert: Agent wants to access {domain}")
        choice = input(f"Allow network access to {domain}? [y/N]: ")
        
        if choice.lower() == 'y':
            self.allowed_domains.add(domain)
            return True
        
        return False
    
    def grant_session_access(self, path: Path):
        """
        Grant access to a path for this session
        Called when user explicitly mentions a folder in their prompt
        """
        self.session_grants.add(path.resolve())
        print(f"    Granted session access to: {path}")
```

---

### Layer 7: Recovery Manager

**Purpose:** Checkpointing and error recovery

```python
class RecoveryManager:
    """
    Manages checkpoints and recovery
    Combines:
      - Trash bins for file recovery
      - State checkpoints for context recovery
      - Browser state storage for session recovery
    """
    
    def __init__(self):
        self.trash_dir = Path.home() / '.cowork' / 'trash'
        self.checkpoint_dir = Path.home() / '.cowork' / 'checkpoints'
        self.browser_checkpoint_dir = Path.home() / '.cowork' / 'browser_checkpoints'
        
        # Create directories
        for dir in [self.trash_dir, self.checkpoint_dir, self.browser_checkpoint_dir]:
            dir.mkdir(parents=True, exist_ok=True)
        
        self.checkpoints: List[Checkpoint] = []
    
    def save_checkpoint(self, session_id: str, context: ContextManager, 
                       step_number: int) -> str:
        """
        Save checkpoint for potential recovery
        
        Returns:
            checkpoint_id
        """
        checkpoint_id = f"{session_id}_{step_number}_{int(time.time())}"
        
        checkpoint_data = {
            'checkpoint_id': checkpoint_id,
            'session_id': session_id,
            'step_number': step_number,
            'timestamp': time.time(),
            'context': context.state,
            'history': context.history,
            'file_hashes': self._compute_file_hashes(),
        }
        
        # Save to disk
        checkpoint_path = self.checkpoint_dir / f"{checkpoint_id}.json"
        with open(checkpoint_path, 'w') as f:
            json.dump(checkpoint_data, f, indent=2)
        
        self.checkpoints.append(Checkpoint(**checkpoint_data))
        
        return checkpoint_id
    
    def load_checkpoint(self, checkpoint_id: str) -> Dict:
        """Load checkpoint data from disk"""
        checkpoint_path = self.checkpoint_dir / f"{checkpoint_id}.json"
        
        if not checkpoint_path.exists():
            raise CheckpointNotFound(f"No checkpoint: {checkpoint_id}")
        
        with open(checkpoint_path) as f:
            return json.load(f)
    
    def generate_recovery_plan(self, checkpoint: Dict, error: Exception) -> str:
        """
        Generate LLM prompt for recovery
        Instead of rolling back time, we provide context for the LLM to replan
        """
        files_changed = self._diff_from_checkpoint(checkpoint)
        
        prompt = f"""
        Task failed at step {checkpoint['step_number']}
        
        Error: {str(error)}
        
        Context at checkpoint:
        {json.dumps(checkpoint['context'], indent=2)}
        
        Files modified since checkpoint:
        {json.dumps(files_changed, indent=2)}
        
        Last actions taken:
        {json.dumps(checkpoint['history'][-5:], indent=2)}
        
        Generate a recovery plan:
        1. What went wrong?
        2. What state are we in now?
        3. What steps should we take to complete the task?
        """
        
        return prompt
    
    def restore_from_trash(self, pattern: str) -> List[Path]:
        """
        Find files in trash matching pattern
        Used for user-initiated recovery: "restore the file I deleted"
        """
        matches = list(self.trash_dir.glob(f"*{pattern}*"))
        return matches
    
    def _compute_file_hashes(self) -> Dict[str, str]:
        """
        Compute MD5 hashes of all files in workspace
        Used to detect what changed between checkpoints
        """
        hashes = {}
        workspace = Path('/workspace')
        
        for file_path in workspace.rglob('*'):
            if file_path.is_file():
                try:
                    with open(file_path, 'rb') as f:
                        file_hash = hashlib.md5(f.read()).hexdigest()
                    hashes[str(file_path)] = file_hash
                except (PermissionError, OSError):
                    continue
        
        return hashes
    
    def _diff_from_checkpoint(self, checkpoint: Dict) -> Dict:
        """
        Compare current filesystem to checkpoint
        Returns dict of {path: status} where status is 'added', 'modified', 'deleted'
        """
        current_hashes = self._compute_file_hashes()
        checkpoint_hashes = checkpoint.get('file_hashes', {})
        
        changes = {}
        
        # Find added and modified files
        for path, current_hash in current_hashes.items():
            if path not in checkpoint_hashes:
                changes[path] = 'added'
            elif current_hash != checkpoint_hashes[path]:
                changes[path] = 'modified'
        
        # Find deleted files
        for path in checkpoint_hashes:
            if path not in current_hashes:
                changes[path] = 'deleted'
        
        return changes

@dataclass
class Checkpoint:
    checkpoint_id: str
    session_id: str
    step_number: int
    timestamp: float
    context: Dict
    history: List
    file_hashes: Dict
```

---

### Layer 8: Observability Stack

**Purpose:** Real-time monitoring, logging, and user intervention

```python
class ProgressTracker:
    """
    Tracks task progress with heartbeats
    Distinguishes between idle and busy-but-slow
    """
    
    def __init__(self):
        self.tasks: Dict[str, TaskProgress] = {}
    
    def start_task(self, task_id: str, total_items: Optional[int] = None, 
                   description: str = "") -> TaskProgress:
        """Initialize new task tracking"""
        progress = TaskProgress(
            task_id=task_id,
            description=description,
            started_at=time.time(),
            last_heartbeat=time.time(),
            total_items=total_items,
            completed_items=0,
            status='running'
        )
        
        self.tasks[task_id] = progress
        return progress
    
    def heartbeat(self, task_id: str, completed: Optional[int] = None, 
                  message: Optional[str] = None):
        """
        Update task progress
        Agent calls this periodically to show it's still working
        """
        if task_id not in self.tasks:
            return
        
        task = self.tasks[task_id]
        task.last_heartbeat = time.time()
        
        if completed is not None:
            task.completed_items = completed
        
        if message:
            task.messages.append({
                'timestamp': time.time(),
                'message': message
            })
    
    def is_idle(self, task_id: str, idle_threshold: int = 300) -> bool:
        """
        Check if task is idle (no heartbeat for idle_threshold seconds)
        
        Args:
            task_id: Task to check
            idle_threshold: Seconds without heartbeat = idle
        
        Returns:
            True if idle, False if still working
        """
        if task_id not in self.tasks:
            return True
        
        task = self.tasks[task_id]
        time_since_heartbeat = time.time() - task.last_heartbeat
        
        return time_since_heartbeat > idle_threshold
    
    def get_eta(self, task_id: str) -> Optional[float]:
        """
        Estimate time to completion
        
        Returns:
            Seconds remaining, or None if cannot estimate
        """
        if task_id not in self.tasks:
            return None
        
        task = self.tasks[task_id]
        
        if not task.total_items or task.completed_items == 0:
            return None
        
        elapsed = time.time() - task.started_at
        items_per_second = task.completed_items / elapsed
        remaining_items = task.total_items - task.completed_items
        
        return remaining_items / items_per_second
    
    def finish_task(self, task_id: str, status: str = 'completed'):
        """Mark task as complete"""
        if task_id in self.tasks:
            self.tasks[task_id].status = status
            self.tasks[task_id].completed_at = time.time()

@dataclass
class TaskProgress:
    task_id: str
    description: str
    started_at: float
    last_heartbeat: float
    total_items: Optional[int]
    completed_items: int
    status: str  # 'running', 'completed', 'failed', 'paused'
    messages: List[Dict] = field(default_factory=list)
    completed_at: Optional[float] = None

class EventStream:
    """
    Real-time event stream for UI consumption
    """
    
    def __init__(self):
        self.subscribers: List[Queue] = []
        self.history: List[Event] = []
    
    def subscribe(self) -> Queue:
        """
        Subscribe to event stream
        Returns queue that will receive events
        """
        queue = Queue()
        self.subscribers.append(queue)
        return queue
    
    def emit(self, event_type: str, data: Dict):
        """
        Emit event to all subscribers
        """
        event = Event(
            type=event_type,
            data=data,
            timestamp=time.time()
        )
        
        # Add to history
        self.history.append(event)
        
        # Notify subscribers
        for queue in self.subscribers:
            try:
                queue.put_nowait(event)
            except Full:
                # Subscriber not consuming fast enough, skip
                pass
    
    def get_history(self, since: Optional[float] = None) -> List[Event]:
        """Get event history since timestamp"""
        if since is None:
            return self.history
        
        return [e for e in self.history if e.timestamp >= since]

@dataclass
class Event:
    type: str
    data: Dict
    timestamp: float

class AuditLogger:
    """
    Immutable audit log for compliance and debugging
    """
    
    def __init__(self):
        self.db_path = Path.home() / '.cowork' / 'audit.db'
        self.db = sqlite3.connect(self.db_path)
        self._init_db()
    
    def _init_db(self):
        """Create audit log table"""
        self.db.execute('''
            CREATE TABLE IF NOT EXISTS audit_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp REAL NOT NULL,
                session_id TEXT NOT NULL,
                tool TEXT NOT NULL,
                command TEXT NOT NULL,
                result TEXT,
                files_modified TEXT,
                exit_code INTEGER
            )
        ''')
        self.db.commit()
    
    def log(self, session_id: str, tool: str, command: str, 
            result: str, files_modified: List[str] = None, 
            exit_code: int = 0):
        """
        Write to audit log
        
        This is immutable - no UPDATE or DELETE, only INSERT
        """
        self.db.execute('''
            INSERT INTO audit_log 
            (timestamp, session_id, tool, command, result, files_modified, exit_code)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', (
            time.time(),
            session_id,
            tool,
            command,
            result,
            json.dumps(files_modified or []),
            exit_code
        ))
        self.db.commit()
    
    def query(self, session_id: Optional[str] = None, 
              since: Optional[float] = None,
              tool: Optional[str] = None) -> List[Dict]:
        """Query audit log with filters"""
        query = "SELECT * FROM audit_log WHERE 1=1"
        params = []
        
        if session_id:
            query += " AND session_id = ?"
            params.append(session_id)
        
        if since:
            query += " AND timestamp >= ?"
            params.append(since)
        
        if tool:
            query += " AND tool = ?"
            params.append(tool)
        
        query += " ORDER BY timestamp DESC"
        
        cursor = self.db.execute(query, params)
        
        return [
            {
                'id': row[0],
                'timestamp': row[1],
                'session_id': row[2],
                'tool': row[3],
                'command': row[4],
                'result': row[5],
                'files_modified': json.loads(row[6]),
                'exit_code': row[7]
            }
            for row in cursor.fetchall()
        ]

class InterruptionHandler:
    """
    Handle graceful interruption (Ctrl+C)
    """
    
    def __init__(self):
        self.cleanup_callbacks: List[Callable] = []
        signal.signal(signal.SIGINT, self._handle_interrupt)
        signal.signal(signal.SIGTERM, self._handle_interrupt)
    
    def register_cleanup(self, callback: Callable):
        """Register cleanup function to call on interruption"""
        self.cleanup_callbacks.append(callback)
    
    def _handle_interrupt(self, signum, frame):
        """Handle Ctrl+C or kill signal"""
        print("\n   Interruption detected. Saving progress...")
        
        # Run cleanup callbacks
        for callback in self.cleanup_callbacks:
            try:
                callback()
            except Exception as e:
                print(f"Cleanup error: {e}")
        
        # Offer user options
        action = self._prompt_user_action()
        
        if action == 'resume':
            print(" Progress saved. Run with --resume to continue.")
            sys.exit(0)
        elif action == 'keep':
            print(" Partial results saved.")
            sys.exit(0)
        else:
            print(" Discarding partial work...")
            sys.exit(1)
    
    def _prompt_user_action(self) -> str:
        """Ask user what to do after interruption"""
        print("""
        What would you like to do?
        1. Save and resume later
        2. Keep partial results and exit
        3. Discard all work
        """)
        
        choice = input("Choice [1/2/3]: ")
        
        if choice == '1':
            return 'resume'
        elif choice == '2':
            return 'keep'
        else:
            return 'discard'

class ObservabilityManager:
    """
    Unified observability interface
    Combines event stream, audit log, and metrics
    """
    
    def __init__(self, session_id: str):
        self.session_id = session_id
        self.event_stream = EventStream()
        self.audit_log = AuditLogger()
        self.progress = ProgressTracker()
        self.interruption = InterruptionHandler()
        self.metrics = MetricsCollector()
    
    def log_action(self, tool: str, command: str, result: Any):
        """
        Log action to all sinks
        """
        # Emit real-time event
        self.event_stream.emit('action_executed', {
            'tool': tool,
            'command': command,
            'result': str(result)
        })
        
        # Write to audit log
        self.audit_log.log(
            session_id=self.session_id,
            tool=tool,
            command=command,
            result=str(result)
        )
        
        # Update metrics
        self.metrics.record_action(tool)
    
    def start_operation(self, description: str, total: Optional[int] = None) -> str:
        """Start tracking an operation"""
        task_id = f"{self.session_id}_{int(time.time())}"
        self.progress.start_task(task_id, total, description)
        
        self.event_stream.emit('operation_started', {
            'task_id': task_id,
            'description': description
        })
        
        return task_id
    
    def heartbeat(self, task_id: str, completed: Optional[int] = None, message: Optional[str] = None):
        """Update operation progress"""
        self.progress.heartbeat(task_id, completed, message)
        
        if message:
            self.event_stream.emit('progress_update', {
                'task_id': task_id,
                'message': message
            })

class MetricsCollector:
    """
    Collect performance metrics
    """
    
    def __init__(self):
        self.action_counts = defaultdict(int)
        self.action_durations = defaultdict(list)
        self.error_counts = defaultdict(int)
    
    def record_action(self, tool: str, duration: Optional[float] = None):
        """Record action execution"""
        self.action_counts[tool] += 1
        
        if duration:
            self.action_durations[tool].append(duration)
    
    def record_error(self, tool: str):
        """Record error occurrence"""
        self.error_counts[tool] += 1
    
    def get_stats(self) -> Dict:
        """Get statistics summary"""
        stats = {
            'total_actions': sum(self.action_counts.values()),
            'actions_by_tool': dict(self.action_counts),
            'errors_by_tool': dict(self.error_counts),
            'avg_duration_by_tool': {}
        }
        
        for tool, durations in self.action_durations.items():
            if durations:
                stats['avg_duration_by_tool'][tool] = sum(durations) / len(durations)
        
        return stats
```

---

## MVP vs 1.0 Feature Split

### MVP (Weekend Build)

**Goal:** Functional, safe, single-session agent

**Included Layers:**
- Layer 0: Foundation (Docker with basic security)
- Layer 1: Brain (LiteLLM with single model)
- Layer 2: Orchestrator (Hybrid, simplified)
- Layer 4: Tool Registry (Bash + Browser basics)
- Layer 6: Security (File validation + trash)
- Layer 8: Observability (Basic stdout logging)

**MVP Features:**
- Execute tasks in secure container
- Provider switching (DeepSeek/Claude)
- File operations with trash safety
- Basic browser automation
- Terminal UI
- Single session only
- Basic error handling

**MVP Limitations:**
- No multi-session support
- No checkpoint/resume
- No vision fallback
- No web UI
- No advanced recovery

### 1.0 (Production Build)

**Goal:** Robust, enterprise-ready agent

**All 10 Layers Included**

**Additional Features:**
- Multi-session coordinator
- Full checkpoint/resume system
- Vision model fallback
- Web dashboard UI
- Advanced error recovery
- A/B testing for models
- Long-running task support
- Detailed audit trails
- Graceful interruption handling
- Resource monitoring
- File lock coordination
- Network firewall
- Intent-based permissions

---

## Technical Decisions & Rationale

### Why Docker Over Native Execution?

**Decision:** Run agent in Docker container, not directly on host

**Rationale:**
1. **Security:** Limits blast radius if agent goes rogue
2. **Resource Control:** Hard limits on CPU/memory prevent runaway processes
3. **Isolation:** Multiple sessions can run without conflicts
4. **Reproducibility:** Same environment every time
5. **Easy Reset:** Kill container to clean state

**Trade-off:** Slight overhead (100-200MB RAM, 5-10% CPU)

---

### Why Hybrid Orchestrator Over Pure ReAct?

**Decision:** Use flexible task graphs with ReAct fallback

**Rationale:**
1. **Efficiency:** Graph avoids repeated LLM calls for deterministic steps
2. **Cost:** One planning call vs. N execution calls
3. **Latency:** Instant local decision vs. 2s LLM call per step
4. **Adaptability:** ReAct handles unexpected situations

**Example:**
```
Pure ReAct: 5 LLM calls × 2s = 10s total
Hybrid:     1 LLM call × 2s + instant execution = 2s total
```

---

### Why Playwright Over Vision as Default?

**Decision:** Use DOM automation (Playwright) first, vision as fallback

**Rationale:**
1. **Reliability:** Selectors more stable than pixel coordinates
2. **Speed:** DOM queries instant, vision inference takes ~1s
3. **Cost:** Playwright free, vision model costs money
4. **Accuracy:** DOM guarantees correctness, vision can misclick

**When Vision Wins:**
- Canvas elements (Google Sheets, Figma)
- Shadow DOM with obfuscated selectors
- Anti-automation sites
- Visual verification needed

---

### Why Trash Over Permanent Delete?

**Decision:** Move files to trash instead of `rm`

**Rationale:**
1. **Safety:** Users can recover from agent mistakes
2. **No Rollback Needed:** Recovery is simple file move
3. **User Expectation:** Matches OS behavior (Trash/Recycle Bin)
4. **Debugging:** Can examine deleted files post-mortem

**Trade-off:** Uses disk space (mitigated by periodic trash cleanup)

---

### Why Intent-Based Permissions Over Always-Ask?

**Decision:** Auto-grant safe operations, confirm high-risk only

**Rationale:**
1. **UX:** Users won't tolerate permission prompts for every `ls`
2. **Security:** Still blocks dangerous operations (delete, encrypt, upload)
3. **Context:** Grants based on what user mentioned in prompt
4. **Balance:** Safety without annoyance

**Example:**
```
User: "Organize my downloads"
→ Auto-grant read/write on ~/Downloads
→ Confirm before deleting any files
→ Block network access (not mentioned in prompt)
```

---

### Why Checkpoints Over Rollback?

**Decision:** Save state snapshots for recovery context, not time-travel

**Rationale:**
1. **Physics:** Can't undo network requests or API calls
2. **Filesystem:** ZFS snapshots not available everywhere
3. **Complexity:** True rollback requires complex transaction system
4. **Practicality:** Context + trash bin solves 90% of recovery needs

**What Checkpoints Provide:**
- Context for LLM to generate recovery plan
- File hash diff to show what changed
- Browser state to restore sessions
- Audit trail for debugging

---

### Why Session-Based Containers Over Persistent VM?

**Decision:** Container lives for session duration, then dies

**Rationale:**
1. **Cleanliness:** No accumulation of junk files/processes
2. **Reliability:** Fresh start each session
3. **Resources:** Temporary containers don't leak memory
4. **Security:** Attack surface resets each session

**Trade-off:** Can't maintain long-running daemons (solved by resumption in 1.0)

---

## Edge Cases & Solutions

### Edge Case 1: Concurrent File Access

**Problem:**
```
Session A: Organizing ~/Downloads
Session B: Cleaning ~/Downloads
→ Both modify same files simultaneously → corruption
```

**Solution:** Layer 0.5 (Session Coordinator)
- File locks prevent concurrent writes
- User chooses which session proceeds on conflict
- Read operations allowed concurrently

---

### Edge Case 2: Long-Running Downloads

**Problem:**
```
User: "Download all my photos from Google Photos"
→ Takes 2 hours
→ Session timeout (30 min) kills task halfway
```

**Solution:** ProgressTracker with Heartbeats
- Agent sends heartbeat every N items processed
- Timeout only triggers if no heartbeat (actually idle)
- Long operations explicitly tracked with ETA

---

### Edge Case 3: Package Installation Delay

**Problem:**
```
User: "Analyze CSV with pandas"
Agent: pip install pandas... (30 seconds)
→ User waiting, session might timeout
```

**Solution:** Layered Docker Images
- Pre-built images with common packages
- Container selection based on prompt analysis
- Runtime installs cached for future sessions

---

### Edge Case 4: Model Unavailability

**Problem:**
```
API call to Claude fails (outage, rate limit, no credits)
→ Task cannot proceed
```

**Solution:** Model Registry with Fallbacks
- Circuit breaker pattern (stop calling failed model)
- Weighted A/B testing (gradual rollout)
- Local cached models as last resort

---

### Edge Case 5: User Interruption (Ctrl+C)

**Problem:**
```
User: "Download emails" (Started 500/5000)
User presses Ctrl+C
→ What happens to partial work?
```

**Solution:** InterruptionHandler
- Catch SIGINT signal
- Save checkpoint of progress
- Offer options: resume later, keep partial, discard
- Graceful cleanup before exit

---

### Edge Case 6: Exfiltration Attack

**Problem:**
```
User: "Organize my photos"
Agent decides: "Upload to my server for ML tagging"
→ Data exfiltration
```

**Solution:** Split Permissions (File vs Network)
- File access auto-granted for mentioned paths
- Network access strictly controlled by allowlist
- Unknown domains trigger security prompt

---

### Edge Case 7: Browser Session Loss

**Problem:**
```
User logged into banking site
Task fails at step 5
Container restarts
→ Lost login session
```

**Solution:** Browser State Checkpoints
- Playwright `storage_state()` saves cookies/localStorage
- Restore checkpoint before retry
- User doesn't need to log in again

---

### Edge Case 8: Model Version Drift

**Problem:**
```
Day 1: Using DeepSeek-V3
Day 30: DeepSeek-V4 released (better)
→ How to upgrade without breaking existing sessions?
```

**Solution:** Model Registry with Versioning
- A/B testing (80% V4, 20% V3)
- Gradual rollout monitors success rate
- Sessions can specify model version explicitly

---

## Security Model

### Defense in Depth

Open Cowork uses multiple security layers:

**1. Container Isolation**
- Agent runs in Docker container
- Cannot access host filesystem except mounted volumes
- Cannot access host network except allowlist
- Cannot escalate to root (userns-remap)

**2. User Mapping**
- Container runs as host UID/GID
- Files created have correct ownership
- No permission issues
- No root access to host files

**3. Path Validation**
- All file operations checked against allowlist
- Only /workspace and /tmp accessible
- System paths blocked (/etc, /usr, /bin)

**4. Network Firewall**
- Domain allowlist enforced
- Unknown domains require user approval
- Prevents data exfiltration
- Logs all network activity

**5. Trash Instead of Delete**
- rm command intercepted
- Files moved to trash, not deleted
- User can recover mistakes
- Trash auto-cleaned after 30 days

**6. Intent-Based Permissions**
- Safe operations auto-granted
- High-risk operations require confirmation
- Context-aware (what user mentioned)
- Preview shown before destructive actions

**7. Audit Logging**
- Every action logged immutably
- SQLite database for queries
- Cannot be tampered with
- Used for debugging and compliance

---

## Future Enhancements

### Version 2.0 Roadmap

1. **Web UI**
   - Real-time progress dashboard
   - Session management
   - Log viewer
   - Model switcher

2. **MCP Integration**
   - Support Model Context Protocol servers
   - Connect to Gmail, Slack, Google Drive
   - Unified tool interface

3. **Advanced Recovery**
   - ZFS snapshot integration
   - Transaction log
   - Automatic rollback on critical errors

4. **Performance Optimization**
   - Parallel tool execution
   - Cached LLM responses
   - Smarter model routing

5. **Enterprise Features**
   - Multi-user support
   - RBAC permissions
   - SSO integration
   - Compliance reporting

---