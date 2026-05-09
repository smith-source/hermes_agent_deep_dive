# 12 — Cron Scheduling: tick() Driver + File Lock + Thread Pool + Subprocess Isolation

## Source Files

| File | Lines | Core Responsibility |
|------|------|---------------------|
| `cron/scheduler.py` | 1594 | tick() main loop + file lock + ThreadPoolExecutor dispatch + run_job() execution |
| `cron/jobs.py` | 1050 | Job CRUD + jobs.json storage + schedule computation + status tracking |
| `tools/cronjob_tools.py` | 662 | Agent-accessible cron management (create/list/pause/run/remove/update) |
| `gateway/delivery.py` | 258 | DeliveryRouter routes cron output to platforms |
| `hermes_cli/cron.py` | 307 | CLI cron subcommands (list/create/edit/pause/resume/run/remove/status/tick) |

> **One-line summary**: Hermes's cron system uses a tick() 60-second loop + file lock mutual exclusion + ThreadPoolExecutor parallel dispatch. Jobs are stored in jobs.json, executed via AIAgent subprocess (isolated session + HERMES_CRON_SESSION environment variable), and output is routed through DeliveryRouter to platforms like Telegram/Discord/Slack.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Trigger Layer (Gateway / CLI)                 │
│  ┌─────────────────┐  ┌────────────────────────────────────┐   │
│  │ Gateway backend  │  │ hermes cron tick (manual trigger)  │   │
│  │ thread           │  │                                    │   │
│  │ 60s loop calls   │  │                                    │   │
│  │ tick()           │  │                                    │   │
│  └──────┬──────────┘  └──────────┬─────────────────────────┘   │
│         │                        │                              │
│         ▼                        ▼                              │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │         tick() — File Lock (.tick.lock) + Parallel Dispatch  ││
│  │   1. fcntl.flock(LOCK_EX|LOCK_NB) — Mutual Exclusion        ││
│  │   2. get_due_jobs() → advance_next_run() — at-most-once     ││
│  │   3. ThreadPoolExecutor(max_workers) — Parallel Execution    ││
│  └──────┬──────────────────────────────────────────────────────┘│
└───────┼─────────────────────────────────────────────────────────┘
        │ per-job
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Job Execution Layer (run_job)                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ no_agent path │  │ LLM path     │  │ wake-gate script     │  │
│  │ (script-only  │  │ AIAgent.run()│  │ check (pre-check     │  │
│  │  execution)   │  │ session      │  │  script)             │  │
│  │ stdout →      │  │ isolation    │  │ wakeAgent=false →    │  │
│  │ direct deliver│  │              │  │ skip entire agent run│  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────────┘  │
│         │                 │                  │                  │
│         ▼                 ▼                  ▼                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │   _build_job_prompt() → skill/model/provider/toolsets config ││
│  │   HERMES_CRON_SESSION=1 → approval cron_mode                 ││
│  │   ContextVars → per-job session/delivery state               ││
│  └──────┬──────────────────────────────────────────────────────┘│
└───────┼─────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Delivery Layer (DeliveryRouter)               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Delivery     │  │ Local file   │  │ Platform adapter     │  │
│  │ Target parse │  │ save         │  │ send                 │  │
│  │ (origin/local│  │ output/      │  │ Telegram/Discord/    │  │
│  │  /platform:) │  │ {job_id}/    │  │ Slack/Signal/Matrix/ │  │
│  │              │  │ {ts}.md      │  │ WhatsApp/Mattermost  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────────┘  │
│         │                 │                  │                  │
│         ▼                 ▼                  ▼                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │   truncate (4000 chars) + full output saved to disk          ││
│  └──────┬──────────────────────────────────────────────────────┘│
└───────┼─────────────────────────────────────────────────────────┘
```

## TL;DR

Hermes's cron system is driven by Gateway calling tick() every 60 seconds. tick() uses fcntl file locks to ensure mutual exclusion, advances next_run_at upfront to guarantee at-most-once semantics, then dispatches job execution in parallel via ThreadPoolExecutor. Jobs are stored in ~/.hermes/cron/jobs.json and support three schedule types: cron expressions, fixed intervals, and one-shot. run_job() has two paths: no_agent for pure script execution (stdout delivered directly) and the LLM path (spawn AIAgent with isolated session). The LLM path marks cron jobs via the HERMES_CRON_SESSION environment variable to trigger approval cron_mode, manages per-job session/delivery state via ContextVars, and supports pre-check scripts (wakeAgent=false gate). Output is routed through DeliveryRouter to 16+ platforms (Telegram/Discord/Slack/WhatsApp etc.), with oversized content truncated to 4000 characters and full output saved to disk. CronjobTools provides agent-accessible cron management (create/update/list/pause/resume/run/remove), and CLI operates via hermes cron subcommands.

---

## tick() Main Loop (cron/scheduler.py)

### File Lock Mutual Exclusion

tick() uses a cross-platform file lock to ensure multi-process mutual exclusion:

```python
Source location: cron/scheduler.py:1432-1462

def tick(verbose=True, adapters=None, loop=None) -> int:
    """Check and run all due jobs. Uses a file lock so only one tick
    runs at a time, even if the gateway's in-process ticker and a
    standalone daemon or manual tick overlap."""
    lock_dir, lock_file = _get_lock_paths()
    lock_dir.mkdir(parents=True, exist_ok=True)
    lock_fd = None
    try:
        lock_fd = open(lock_file, "w")
        if fcntl:
            fcntl.flock(lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)  # Non-blocking exclusive lock
        elif msvcrt:
            msvcrt.locking(lock_fd.fileno(), msvcrt.LK_NBLCK, 1)  # Windows
    except (OSError, IOError):
        logger.debug("Tick skipped — another instance holds the lock")
        if lock_fd is not None:
            lock_fd.close()
        return 0  # Another process holds the lock, return immediately
```

### At-Most-Once Semantics: Advance First, Then Execute

```python
Source location: cron/scheduler.py:1464-1478

    try:
        due_jobs = get_due_jobs()
        # Advance next_run_at for all recurring jobs FIRST, under the file lock,
        # before any execution begins. This preserves at-most-once semantics.
        for job in due_jobs:
            advance_next_run(job["id"])
```

This means: even if job execution fails, next_run_at has already been advanced to the next scheduled time, so the same time slot will not be triggered again.

### Parallel Dispatch Strategy

```python
Source location: cron/scheduler.py:1479-1568

        # Resolve max parallel workers: env var > config.yaml > unbounded.
        _max_workers = None
        _env_par = os.getenv("HERMES_CRON_MAX_PARALLEL", "").strip()
        if _env_par:
            _max_workers = int(_env_par) or None
        if _max_workers is None:
            _cfg_par = (_ucfg.get("cron", {})).get("max_parallel_jobs")
            if _cfg_par is not None:
                _max_workers = int(_cfg_par) or None

        # Partition due jobs: workdir-jobs mutate os.environ["TERMINAL_CWD"]
        # process-global — they MUST run sequentially to avoid corrupting
        # each other. Jobs without workdir stay parallel-safe.
        workdir_jobs = [j for j in due_jobs if (j.get("workdir") or "").strip()]
        parallel_jobs = [j for j in due_jobs if not (j.get("workdir") or "").strip()]

        # Sequential pass for workdir jobs.
        for job in workdir_jobs:
            _ctx = contextvars.copy_context()
            _results.append(_ctx.run(_process_job, job))

        # Parallel pass for the rest.
        if parallel_jobs:
            with ThreadPoolExecutor(max_workers=_max_workers) as _tick_pool:
                _futures = []
                for job in parallel_jobs:
                    _ctx = contextvars.copy_context()
                    _futures.append(_tick_pool.submit(_ctx.run, _process_job, job))
                _results.extend(f.result() for f in _futures)
```

### _process_job: Execute + Save + Deliver + Status Update

```python
Source location: cron/scheduler.py:1506-1545

        def _process_job(job: dict) -> bool:
            """Run one due job end-to-end: execute, save, deliver, mark."""
            try:
                success, output, final_response, error = run_job(job)
                output_file = save_job_output(job["id"], output)

                # Deliver the final response to the origin/target chat.
                deliver_content = final_response if success else \
                    f"⚠️ Cron job '{job.get('name', job['id'])}' failed:\n{error}"
                should_deliver = bool(deliver_content)
                if should_deliver and success and SILENT_MARKER in deliver_content.strip().upper():
                    should_deliver = False  # [SILENT] → skip delivery

                delivery_error = None
                if should_deliver:
                    delivery_error = _deliver_result(job, deliver_content, adapters, loop)

                # Empty final_response = soft failure
                if success and not final_response:
                    success = False
                    error = "Agent completed but produced empty response"

                mark_job_run(job["id"], success, error, delivery_error=delivery_error)
                return True
            except Exception as e:
                mark_job_run(job["id"], False, str(e))
                return False
```

### MCP Orphan Cleanup

```python
Source location: cron/scheduler.py:1570-1579

        # Best-effort sweep of MCP stdio subprocesses that survived their
        # session teardown during this tick.
        try:
            from tools.mcp_tool import _kill_orphaned_mcp_children
            _kill_orphaned_mcp_children()
        except Exception as _e:
            logger.debug("Post-tick MCP orphan cleanup failed: %s", _e)
```

---

## run_job() Job Execution (cron/scheduler.py)

### no_agent Path (Pure Script)

```python
Source location: cron/scheduler.py:866-965

    # no_agent short-circuit — the script IS the job, no LLM involvement.
    if job.get("no_agent"):
        script_path = job.get("script")
        if not script_path:
            return False, "", "", "no_agent=True but no script is set"
        # Apply workdir if configured
        if _job_workdir and Path(_job_workdir).is_dir():
            _prior_cwd = os.getcwd()
            os.chdir(_job_workdir)
        try:
            ok, output = _run_job_script(script_path)
        finally:
            if _prior_cwd:
                os.chdir(_prior_cwd)

        # wakeAgent=false gate → silent run
        if not _parse_wake_gate(output):
            return True, silent_doc, SILENT_MARKER, None
        # Empty stdout → silent
        if not output.strip():
            return True, silent_doc, SILENT_MARKER, None
        # Non-empty stdout → deliver verbatim
        return True, doc, output, None
```

### LLM Path (AIAgent)

```python
Source location: cron/scheduler.py:967-1021

    # Default (LLM) path — import AIAgent
    from run_agent import AIAgent

    # Wake-gate: pre-check script
    prerun_script = None
    if script_path:
        prerun_script = _run_job_script(script_path)
        _ran_ok, _script_output = prerun_script
        if _ran_ok and not _parse_wake_gate(_script_output):
            logger.info("Job '%s': wakeAgent=false, skipping agent run", job_name)
            return True, silent_doc, SILENT_MARKER, None

    prompt = _build_job_prompt(job, prerun_script=prerun_script)
    origin = _resolve_origin(job)
    _cron_session_id = f"cron_{job_id}_{_hermes_now().strftime('%Y%m%d_%H%M%S')}"

    # Mark this as a cron session
    os.environ["HERMES_CRON_SESSION"] = "1"

    # Use ContextVars for per-job session/delivery state
    from gateway.session_context import set_session_vars, clear_session_vars, _VAR_MAP
    _ctx_tokens = set_session_vars(
        platform=origin["platform"] if origin else "",
        chat_id=str(origin["chat_id"]) if origin else "",
        chat_name=origin.get("chat_name", "") if origin else "",
    )
```

### Session Isolation: ContextVars Replacing os.environ

```python
Source location: cron/scheduler.py:1032-1081

    _cron_delivery_vars = (
        "HERMES_CRON_AUTO_DELIVER_PLATFORM",
        "HERMES_CRON_AUTO_DELIVER_CHAT_ID",
        "HERMES_CRON_AUTO_DELIVER_THREAD_ID",
    )
    for _var_name in _cron_delivery_vars:
        _VAR_MAP[_var_name].set("")  # ContextVars: per-thread, not process-global

    delivery_target = _resolve_delivery_target(job)
    if delivery_target:
        _VAR_MAP["HERMES_CRON_AUTO_DELIVER_PLATFORM"].set(delivery_target["platform"])
        _VAR_MAP["HERMES_CRON_AUTO_DELIVER_CHAT_ID"].set(str(delivery_target["chat_id"]))
        _VAR_MAP["HERMES_CRON_AUTO_DELIVER_THREAD_ID"].set(
            "" if delivery_target.get("thread_id") is None
            else str(delivery_target["thread_id"])
        )
```

### Toolset Configuration

```python
Source location: cron/scheduler.py:44-72

def _resolve_cron_enabled_toolsets(job: dict, cfg: dict) -> list[str] | None:
    """Resolve the toolset list for a cron job.
    Precedence:
    1. Per-job enabled_toolsets (set via cronjob tool on create/update)
    2. Per-platform hermes tools config for the cron platform
    3. None — AIAgent loads the full default set (legacy behavior)"""
    per_job = job.get("enabled_toolsets")
    if per_job:
        return per_job
    try:
        from hermes_cli.tools_config import _get_platform_tools
        return sorted(_get_platform_tools(cfg or {}, "cron"))
    except Exception:
        return None  # fallback to full default
```

### Model/Provider/Reasoning Configuration

```python
Source location: cron/scheduler.py:1083-1194

    model = job.get("model") or os.getenv("HERMES_MODEL") or ""
    # Load config.yaml for model, reasoning, prefill, toolsets, provider routing
    _cfg = yaml.safe_load(...) or {}
    _model_cfg = _cfg.get("model", {})
    if not job.get("model"):
        if isinstance(_model_cfg, str):
            model = _model_cfg
        elif isinstance(_model_cfg, dict):
            model = _model_cfg.get("default", model)

    # Provider routing with fallback chain
    runtime = resolve_runtime_provider(requested=job.get("provider"))
    # Fallback chain on AuthError
    fb_list = _cfg.get("fallback_providers") or _cfg.get("fallback_model")
    for entry in fb_list:
        runtime = resolve_runtime_provider(requested=entry.get("provider"), ...)

    # Credential pool for provider
    credential_pool = load_pool(runtime_provider)
    if pool.has_credentials():
        credential_pool = pool
```

### Comparison: run_job() Execution Paths

| Feature | no_agent Path | LLM Path (AIAgent) |
|---------|---------------|---------------------|
| Invocation | subprocess (bash/python script) | AIAgent.run() |
| Model usage | None | Yes (configurable per-job model/provider) |
| Session isolation | None | HERMES_CRON_SESSION=1 + ContextVars |
| Approval mode | None | cron_mode (deny/approve) |
| Toolsets | None | Configurable per-job enabled_toolsets |
| wake-gate | wakeAgent=false → silent | wakeAgent=false → skip entire agent run |
| Output format | stdout direct delivery | final_response (agent reply) |
| Cost | None (script execution) | LLM token consumption |
| Use cases | Watchdog / periodic scripts / system health checks | LLM analysis / data processing / automated tasks |

---

## Job Storage and Management (cron/jobs.py)

### Storage Location and Format

```python
Source location: cron/jobs.py:1-44

"""
Cron job storage and management.
Jobs are stored in ~/.hermes/cron/jobs.json
Output is saved to ~/.hermes/cron/output/{job_id}/{timestamp}.md
"""

HERMES_DIR = get_hermes_home().resolve()
CRON_DIR = HERMES_DIR / "cron"
JOBS_FILE = CRON_DIR / "jobs.json"
OUTPUT_DIR = CRON_DIR / "output"

# In-process lock protecting load_jobs→modify→save_jobs cycles.
_jobs_file_lock = threading.Lock()
```

### create_job()

```python
Source location: cron/jobs.py:422-501

def create_job(
    prompt: Optional[str],
    schedule: str,
    name: Optional[str] = None,
    repeat: Optional[int] = None,
    deliver: Optional[str] = None,
    origin: Optional[Dict[str, Any]] = None,
    skill: Optional[str] = None,
    skills: Optional[List[str]] = None,
    model: Optional[str] = None,
    provider: Optional[str] = None,
    base_url: Optional[str] = None,
    script: Optional[str] = None,
    context_from: Optional[Union[str, List[str]]] = None,
    enabled_toolsets: Optional[List[str]] = None,
    workdir: Optional[str] = None,
    no_agent: bool = False,
) -> Dict[str, Any]:
    """Create a new cron job."""
    parsed_schedule = parse_schedule(schedule)
    if repeat is not None and repeat <= 0:
        repeat = None
    if parsed_schedule["kind"] == "once" and repeat is None:
        repeat = 1  # Auto-set repeat=1 for one-shot schedules
    if deliver is None:
        deliver = "origin" if origin else "local"
    job_id = uuid.uuid4().hex[:12]
```

### Schedule Parsing: parse_schedule()

```python
Source location: cron/jobs.py:124-212

def parse_schedule(schedule: str) -> Dict[str, Any]:
    """Parse a schedule string into a structured dict.
    Supports: cron expressions, intervals ('every 5m'), one-shot ('in 2h', 'at 2024-01-01')."""
```

### Schedule Computation: compute_next_run()

```python
Source location: cron/jobs.py:291-340

def compute_next_run(schedule: Dict[str, Any], last_run_at=None) -> Optional[str]:
    """Compute the next run time based on schedule type."""
```

### Job Lifecycle

| Function | Purpose | Key Parameters |
|----------|---------|----------------|
| `create_job()` | Create a new job | prompt, schedule, skills, deliver, model, provider, script, no_agent |
| `get_job(job_id)` | Get a single job | job_id |
| `list_jobs()` | List all jobs | include_disabled |
| `update_job(job_id, updates)` | Update a job | job_id + updates dict |
| `pause_job(job_id, reason)` | Pause a job | job_id, reason |
| `resume_job(job_id)` | Resume a job | job_id |
| `trigger_job(job_id)` | Trigger immediately | job_id |
| `remove_job(job_id)` | Delete a job | job_id |
| `mark_job_run(job_id, success, error)` | Record run result | job_id, success, error, delivery_error |
| `advance_next_run(job_id)` | Advance next run time | job_id |
| `get_due_jobs()` | Get due jobs | (uses _get_due_jobs_locked) |
| `save_job_output(job_id, output)` | Save output to file | job_id, output |

---

## CronjobTools (tools/cronjob_tools.py)

### Agent-Accessible Cron Management

```python
Source location: tools/cronjob_tools.py:257-340

def cronjob(
    action: str,
    job_id: Optional[str] = None,
    prompt: Optional[str] = None,
    schedule: Optional[str] = None,
    name: Optional[str] = None,
    repeat: Optional[int] = None,
    deliver: Optional[str] = None,
    include_disabled: bool = False,
    skill: Optional[str] = None,
    skills: Optional[List[str]] = None,
    model: Optional[str] = None,
    provider: Optional[str] = None,
    base_url: Optional[str] = None,
    reason: Optional[str] = None,
    script: Optional[str] = None,
    context_from: Optional[Union[str, List[str]]] = None,
    enabled_toolsets: Optional[List[str]] = None,
    workdir: Optional[str] = None,
    no_agent: Optional[bool] = None,
    task_id: str = None,
) -> str:
    """Unified cron job management tool."""
```

### Action Routing

| action | Agent Behavior | Call Target |
|--------|----------------|-------------|
| "create" | Create a new cron job | create_job() |
| "update" | Update an existing job | update_job() |
| "list" | List all jobs | list_jobs() |
| "pause" | Pause a job | pause_job() |
| "resume" | Resume a paused job | resume_job() |
| "run" | Trigger a job immediately | trigger_job() |
| "remove" | Delete a job | remove_job() |

### Creation Validation

```python
Source location: tools/cronjob_tools.py:285-326

    if normalized == "create":
        if not schedule:
            return tool_error("schedule is required for create")
        _no_agent = bool(no_agent)
        if _no_agent:
            if not script:
                return tool_error("create with no_agent=True requires a script")
        else:
            if not prompt and not canonical_skills:
                return tool_error("create requires either prompt or at least one skill")
        if prompt:
            scan_error = _scan_cron_prompt(prompt)  # Scan prompt for dangerous content
            if scan_error:
                return tool_error(scan_error)
        if script:
            script_error = _validate_cron_script_path(script)
            if script_error:
                return tool_error(script_error)
        if context_from:
            for ref_id in refs:
                if not _get_job(ref_id):
                    return tool_error(f"context_from job '{ref_id}' not found")
```

### Delivery Platform Validation

```python
Source location: cron/scheduler.py:74-82

_KNOWN_DELIVERY_PLATFORMS = frozenset({
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant", "dingtalk", "feishu",
    "wecom", "wecom_callback", "weixin", "sms", "email", "webhook",
    "bluebubbles", "qqbot", "yuanbao", ...
})
```

### Comparison: CronjobTools vs. hermes CLI cron

| Operation | CronjobTools (agent tool) | hermes CLI cron | hermes cron --help |
|-----------|--------------------------|----------------|-------------------|
| create | cronjob(action="create") | hermes cron create | hermes cron create -s "every 5m" -p "check server" |
| list | cronjob(action="list") | hermes cron list | hermes cron list --all |
| pause | cronjob(action="pause") | hermes cron pause <id> | hermes cron pause abc123 |
| resume | cronjob(action="resume") | hermes cron resume <id> | hermes cron resume abc123 |
| run | cronjob(action="run") | hermes cron run <id> | hermes cron run abc123 |
| remove | cronjob(action="remove") | hermes cron remove <id> | hermes cron remove abc123 |
| update | cronjob(action="update") | hermes cron edit <id> | hermes cron edit abc123 --prompt "new prompt" |
| status | N/A (use list) | hermes cron status | hermes cron status |
| tick | N/A | hermes cron tick | hermes cron tick |

---

## DeliveryRouter (gateway/delivery.py)

### Delivery Target Parsing

```python
Source location: gateway/delivery.py:28-107

@dataclass
class DeliveryTarget:
    """A single delivery target.
    - "origin" → back to source
    - "local" → save to local files
    - "telegram" → Telegram home channel
    - "telegram:123456" → specific Telegram chat"""

    platform: Platform
    chat_id: Optional[str] = None
    thread_id: Optional[str] = None
    is_origin: bool = False
    is_explicit: bool = False

    @classmethod
    def parse(cls, target: str, origin=None) -> "DeliveryTarget":
        """Parse a delivery target string.
        Formats:
        - "origin" → back to source
        - "local" → local files only
        - "telegram" → Telegram home channel
        - "telegram:123456" → specific Telegram chat"""
```

### Delivery Routing and Truncation

```python
Source location: gateway/delivery.py:109-170

class DeliveryRouter:
    """Routes messages to appropriate destinations."""
    def __init__(self, config, adapters=None):
        self.config = config
        self.adapters = adapters or {}
        self.output_dir = get_hermes_home() / "cron" / "output"

    async def deliver(self, content, targets, job_id=None, job_name=None, metadata=None):
        """Deliver content to all specified targets."""
        results = {}
        for target in targets:
            if target.platform == Platform.LOCAL:
                result = self._deliver_local(content, job_id, job_name, metadata)
            else:
                result = await self._deliver_to_platform(target, content, metadata)
            results[target.to_string()] = {"success": True, "result": result}
        return results
```

```python
Source location: gateway/delivery.py:226-254

    async def _deliver_to_platform(self, target, content, metadata):
        """Deliver content to a messaging platform."""
        adapter = self.adapters.get(target.platform)
        if not adapter:
            raise ValueError(f"No adapter configured for {target.platform}")
        # Guard: truncate oversized cron output to stay within platform limits
        if len(content) > MAX_PLATFORM_OUTPUT:
            saved_path = self._save_full_output(content, job_id)
            content = (
                content[:TRUNCATED_VISIBLE]
                + f"\n\n... [truncated, full output saved to {saved_path}]"
            )
        return await adapter.send(target.chat_id, content, metadata=send_metadata)
```

### Comparison: DeliveryRouter Delivery Modes

| Delivery Format | Example | Routing Behavior |
|-----------------|---------|------------------|
| "origin" | Return to originating platform | Deliver back to the chat_id that created the job |
| "local" | Local file save | ~/.hermes/cron/output/{job_id}/{ts}.md |
| "telegram" | Telegram home channel | Deliver to Telegram default channel |
| "telegram:123456" | Specific Telegram chat | Deliver to chat_id=123456 |
| "discord:789:456" | Discord channel + thread | Deliver to specific channel and thread |
| "slack" | Slack home channel | Deliver to Slack default channel |
| Multiple targets | ["origin", "local"] | Deliver to all targets sequentially |

---

## Cron CLI (hermes_cli/cron.py)

### Subcommand Routing

```python
Source location: hermes_cli/cron.py:270-307

def cron_command(args):
    """Handle cron subcommands."""
    subcmd = getattr(args, 'cron_command', None)
    if subcmd is None or subcmd == "list":
        cron_list(show_all)
    elif subcmd == "status":
        cron_status()
    elif subcmd == "tick":
        cron_tick()
    elif subcmd in {"create", "add"}:
        return cron_create(args)
    elif subcmd == "edit":
        return cron_edit(args)
    elif subcmd == "pause":
        return _job_action("pause", args.job_id, "Paused")
    elif subcmd == "resume":
        return _job_action("resume", args.job_id, "Resumed")
    elif subcmd == "run":
        return _job_action("run", args.job_id, "Triggered")
    elif subcmd in {"remove", "rm", "delete"}:
        return _job_action("remove", args.job_id, "Removed")
```

### cron_list() Display Format

```python
Source location: hermes_cli/cron.py:41-116

def cron_list(show_all=False):
    """List all scheduled jobs."""
    jobs = list_jobs(include_disabled=show_all)
    for job in jobs:
        job_id = job.get("id", "?")
        name = job.get("name", "(unnamed)")
        schedule = job.get("schedule_display", job.get("schedule", {}).get("value", "?"))
        state = job.get("state", "scheduled" if job.get("enabled", True) else "paused")
        # Status indicators: [active] [paused] [completed] [disabled]
        # Repeat display: 3/5 or ∞
        # Last run: ok or error message
        # Delivery error: ⚠ Delivery failed
```

### cron_status() Gateway Detection

```python
Source location: hermes_cli/cron.py:132-161

def cron_status():
    """Show cron execution status."""
    pids = find_gateway_pids()
    if pids:
        print("✓ Gateway is running — cron jobs will fire automatically")
        print(f"  PID: {', '.join(map(str, pids))}")
    else:
        print("✗ Gateway is not running — cron jobs will NOT fire")
        print("  To enable automatic execution:")
        print("    hermes gateway install")
        print("    sudo hermes gateway install --system  # Linux servers")
```

### Comparison: CLI cron vs. Agent cronjob Tool

| Dimension | hermes CLI cron | CronjobTools (agent) |
|-----------|----------------|---------------------|
| Trigger method | User command line | LLM-generated tool_call |
| Input validation | argparse | _scan_cron_prompt + _validate_cron_script_path |
| Output format | Colored terminal table | JSON string |
| Job creation | cron_create(args) | cronjob(action="create", ...) |
| Job update | cron_edit(args) | cronjob(action="update", ...) |
| Gateway detection | find_gateway_pids() | N/A |
| Delivery parameter | --deliver "telegram:123" | deliver="telegram:123456" |
| Skill parameter | --skill / --skills | skill / skills |

---

## Job Execution Security

### HERMES_CRON_SESSION Environment Variable

```python
Source location: cron/scheduler.py:1018-1021

    # Mark this as a cron session so the approval system can apply cron_mode.
    os.environ["HERMES_CRON_SESSION"] = "1"
```

This triggers the cron approval mode in `tools/approval.py`:

```python
Source location: tools/approval.py:834-848

    if not is_cli and not is_gateway:
        # Cron sessions: respect cron_mode config
        if os.getenv("HERMES_CRON_SESSION"):
            if _get_cron_approval_mode() == "deny":
                return {
                    "approved": False,
                    "message": (
                        f"BLOCKED: Command flagged as dangerous ({description}) "
                        "but cron jobs run without a user present to approve it. "
                        "To allow dangerous commands in cron jobs, set "
                        "approvals.cron_mode: approve in config.yaml."
                    ),
                }
```

### Delivery Platform Name Validation

```python
Source location: cron/scheduler.py:74-82

_KNOWN_DELIVERY_PLATFORMS = frozenset({
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant", "dingtalk", "feishu",
    ...
})
```

Prevents malicious agents from enumerating environment variables by forging platform names.

### Comparison: Cron Security Mechanisms

| Security Mechanism | Implementation | Purpose |
|-------------------|----------------|---------|
| HERMES_CRON_SESSION | Environment variable marker | Trigger approval cron_mode |
| _KNOWN_DELIVERY_PLATFORMS | frozenset allowlist | Prevent platform name enumeration |
| _scan_cron_prompt | Prompt scanning | Prevent dangerous prompts |
| _validate_cron_script_path | Path validation | Prevent path traversal |
| _resolve_cron_enabled_toolsets | Toolset filtering | Restrict tools available to cron |
| workdir serialization | Partition parallelism | Prevent TERMINAL_CWD race conditions |
| _secure_dir (0700) | Directory permissions | Protect job files |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `cron/scheduler.py` | 1594 | tick() main loop + file lock + ThreadPoolExecutor + run_job (no_agent/LLM paths) |
| `cron/jobs.py` | 1050 | Job CRUD + jobs.json storage + schedule computation + status tracking + file permissions |
| `tools/cronjob_tools.py` | 662 | Agent cron management (7 actions + prompt/script/context_from validation) |
| `gateway/delivery.py` | 258 | DeliveryRouter — target parsing + platform routing + truncation (4000 chars) |
| `hermes_cli/cron.py` | 307 | CLI cron subcommands (list/create/edit/pause/resume/run/remove/status/tick) |

---

[← 11 — Skills and Self-Improvement System](/en/chapters/11-skills-self-improvement) | [→ 13 — ...](/en/chapters/13-rl-training)
