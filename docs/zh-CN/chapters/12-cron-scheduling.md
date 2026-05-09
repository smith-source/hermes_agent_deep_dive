
# 12 — 定时任务与调度: tick() 驱动 + 文件锁 + 线程池 + 子进程隔离

## 源码文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `cron/scheduler.py` | 1594 | tick() 主循环 + 文件锁 + 线程池分发 + run_job() 执行 |
| `cron/jobs.py` | 1050 | 作业 CRUD + jobs.json 存储 + 计算调度 + 状态追踪 |
| `tools/cronjob_tools.py` | 662 | agent 可访问的 cron 管理 (create/list/pause/run/remove/update) |
| `gateway/delivery.py` | 258 | DeliveryRouter 路由 cron 输出到平台 |
| `hermes_cli/cron.py` | 307 | CLI cron 子命令 (list/create/edit/pause/resume/run/remove/status/tick) |

> **一句话总结**: Hermes 的 cron 系统使用 tick() 60 秒循环 + 文件锁互斥 + ThreadPoolExecutor 并行分发, 作业存储在 jobs.json, 执行时通过 AIAgent 子进程 (隔离 session + HERMES_CRON_SESSION 环境变量) 运行, 输出通过 DeliveryRouter 路由到 Telegram/Discord/Slack 等平台。

## 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    触发层 (Gateway / CLI)                        │
│  ┌─────────────────┐  ┌────────────────────────────────────┐   │
│  │ Gateway 后台线程 │  │ hermes cron tick (手动触发)         │   │
│  │ 60s 循环调用     │  │                                    │   │
│  │ tick()           │  │                                    │   │
│  └──────┬──────────┘  └──────────┬─────────────────────────┘   │
│         │                        │                              │
│         ▼                        ▼                              │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │         tick() — 文件锁 (.tick.lock) + 并行分发               ││
│  │   1. fcntl.flock(LOCK_EX|LOCK_NB) — 互斥                    ││
│  │   2. get_due_jobs() → advance_next_run() — at-most-once     ││
│  │   3. ThreadPoolExecutor(max_workers) — 并行执行               ││
│  └──────┬──────────────────────────────────────────────────────┘│
└───────┼─────────────────────────────────────────────────────────┘
        │ per-job
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    作业执行层 (run_job)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ no_agent 路径 │  │ LLM 路径     │  │ wake-gate 脚本检查   │  │
│  │ (纯脚本执行)  │  │ AIAgent.run()│  │ (预检查脚本)         │  │
│  │ stdout → 直投 │  │ session 隔离 │  │ wakeAgent=false →    │  │
│  │              │  │              │  │ 跳过整个 agent run    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────────┘  │
│         │                 │                  │                  │
│         ▼                 ▼                  ▼                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │   _build_job_prompt() → skill/model/provider/toolsets 配置   ││
│  │   HERMES_CRON_SESSION=1 → 审批 cron_mode                    ││
│  │   ContextVars → per-job session/delivery 状态               ││
│  └──────┬──────────────────────────────────────────────────────┘│
└───────┼─────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    交付层 (DeliveryRouter)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Delivery     │  │ 本地文件保存 │  │ 平台适配器发送       │  │
│  │ Target 解析  │  │ output/      │  │ Telegram/Discord/    │  │
│  │ (origin/local│  │ {job_id}/    │  │ Slack/Signal/Matrix/ │  │
│  │  /platform:) │  │ {ts}.md      │  │ WhatsApp/Mattermost  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────────────┘  │
│         │                 │                  │                  │
│         ▼                 ▼                  ▼                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │   truncate (4000 chars) + full output saved to disk          ││
│  └──────┬──────────────────────────────────────────────────────┘│
└───────┼─────────────────────────────────────────────────────────┘
```

## TL;DR

Hermes 的 cron 系统由 Gateway 每 60 秒调用 tick(), tick() 使用 fcntl 文件锁确保互斥, 提前推进 next_run_at 保证 at-most-once 语义, 然后通过 ThreadPoolExecutor 并行分发作业执行。作业存储在 ~/.hermes/cron/jobs.json, 支持 cron 表达式 / 固定间隔 / one-shot 三种调度类型。run_job() 有两条路径: no_agent 纯脚本执行 (stdout 直接投递) 和 LLM 路径 (spawn AIAgent with isolated session)。LLM 路径通过 HERMES_CRON_SESSION 环境变量标记 cron 作业以触发审批 cron_mode, 通过 ContextVars 管理 per-job 会话/交付状态, 支持预检查脚本 (wakeAgent=false gate)。输出通过 DeliveryRouter 路由到 16+ 平台 (Telegram/Discord/Slack/WhatsApp 等), 超长内容截断为 4000 字符并保存完整输出到磁盘。CronjobTools 提供 agent 可访问的 cron 管理 (create/update/list/pause/resume/run/remove), CLI 通过 hermes cron 子命令操作。

---

## tick() 主循环 (cron/scheduler.py)

### 文件锁互斥

tick() 使用跨平台文件锁确保多进程互斥:

```python
源码位置: cron/scheduler.py:1432-1462

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
            fcntl.flock(lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)  # 非阻塞排他锁
        elif msvcrt:
            msvcrt.locking(lock_fd.fileno(), msvcrt.LK_NBLCK, 1)  # Windows
    except (OSError, IOError):
        logger.debug("Tick skipped — another instance holds the lock")
        if lock_fd is not None:
            lock_fd.close()
        return 0  # 另一个进程持有锁, 直接返回
```

### At-Most-Once 语义: 先推进后执行

```python
源码位置: cron/scheduler.py:1464-1478

    try:
        due_jobs = get_due_jobs()
        # Advance next_run_at for all recurring jobs FIRST, under the file lock,
        # before any execution begins. This preserves at-most-once semantics.
        for job in due_jobs:
            advance_next_run(job["id"])
```

这意味着: 即使作业执行失败, next_run_at 也已推进到下一个调度时间, 不会重复触发同一个时间点。

### 并行分发策略

```python
源码位置: cron/scheduler.py:1479-1568

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

### _process_job: 执行 + 保存 + 交付 + 状态更新

```python
源码位置: cron/scheduler.py:1506-1545

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

### MCP orphan 清理

```python
源码位置: cron/scheduler.py:1570-1579

        # Best-effort sweep of MCP stdio subprocesses that survived their
        # session teardown during this tick.
        try:
            from tools.mcp_tool import _kill_orphaned_mcp_children
            _kill_orphaned_mcp_children()
        except Exception as _e:
            logger.debug("Post-tick MCP orphan cleanup failed: %s", _e)
```

---

## run_job() 作业执行 (cron/scheduler.py)

### no_agent 路径 (纯脚本)

```python
源码位置: cron/scheduler.py:866-965

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

### LLM 路径 (AIAgent)

```python
源码位置: cron/scheduler.py:967-1021

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

### 会话隔离: ContextVars 替代 os.environ

```python
源码位置: cron/scheduler.py:1032-1081

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

### 工具集配置

```python
源码位置: cron/scheduler.py:44-72

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

### 模型/提供商/推理配置

```python
源码位置: cron/scheduler.py:1083-1194

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

### 比较: run_job() 执行路径

| 特性 | no_agent 路径 | LLM 路径 (AIAgent) |
|------|---------------|---------------------|
| 调用方式 | subprocess (bash/python script) | AIAgent.run() |
| 模型使用 | 无 | 有 (可配置 per-job model/provider) |
| 会话隔离 | 无 | HERMES_CRON_SESSION=1 + ContextVars |
| 审批模式 | 无 | cron_mode (deny/approve) |
| 工具集 | 无 | 可配置 per-job enabled_toolsets |
| wake-gate | wakeAgent=false → silent | wakeAgent=false → skip entire agent run |
| 输出格式 | stdout 直投 | final_response (agent 回复) |
| 费用 | 无 (脚本运行) | LLM token 消耗 |
| 适用场景 | watchdog / 定期脚本 / 系统健康检查 | LLM 分析 / 数据处理 / 自动化任务 |

---

## 作业存储与管理 (cron/jobs.py)

### 存储位置与格式

```python
源码位置: cron/jobs.py:1-44

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
源码位置: cron/jobs.py:422-501

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

### 调度解析: parse_schedule()

```python
源码位置: cron/jobs.py:124-212

def parse_schedule(schedule: str) -> Dict[str, Any]:
    """Parse a schedule string into a structured dict.
    Supports: cron expressions, intervals ('every 5m'), one-shot ('in 2h', 'at 2024-01-01')."""
```

### 调度计算: compute_next_run()

```python
源码位置: cron/jobs.py:291-340

def compute_next_run(schedule: Dict[str, Any], last_run_at=None) -> Optional[str]:
    """Compute the next run time based on schedule type."""
```

### 作业生命周期

| 函数 | 作用 | 关键参数 |
|------|------|----------|
| `create_job()` | 创建新作业 | prompt, schedule, skills, deliver, model, provider, script, no_agent |
| `get_job(job_id)` | 获取单个作业 | job_id |
| `list_jobs()` | 列出所有作业 | include_disabled |
| `update_job(job_id, updates)` | 更新作业 | job_id + updates dict |
| `pause_job(job_id, reason)` | 暂停作业 | job_id, reason |
| `resume_job(job_id)` | 恢复作业 | job_id |
| `trigger_job(job_id)` | 立即触发 | job_id |
| `remove_job(job_id)` | 删除作业 | job_id |
| `mark_job_run(job_id, success, error)` | 记录运行结果 | job_id, success, error, delivery_error |
| `advance_next_run(job_id)` | 推进下一次运行时间 | job_id |
| `get_due_jobs()` | 获取到期作业 | (uses _get_due_jobs_locked) |
| `save_job_output(job_id, output)` | 保存输出到文件 | job_id, output |

---

## CronjobTools (tools/cronjob_tools.py)

### agent 可访问的 cron 管理

```python
源码位置: tools/cronjob_tools.py:257-340

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

### 操作路由

| action | 代理行为 | 调用目标 |
|--------|----------|----------|
| "create" | 创建新 cron 作业 | create_job() |
| "update" | 更新现有作业 | update_job() |
| "list" | 列出所有作业 | list_jobs() |
| "pause" | 暂停作业 | pause_job() |
| "resume" | 恢复暂停的作业 | resume_job() |
| "run" | 立即触发作业 | trigger_job() |
| "remove" | 删除作业 | remove_job() |

### 创建验证

```python
源码位置: tools/cronjob_tools.py:285-326

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
            scan_error = _scan_cron_prompt(prompt)  # 扫描 prompt 中的危险内容
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

### 交付平台验证

```python
源码位置: cron/scheduler.py:74-82

_KNOWN_DELIVERY_PLATFORMS = frozenset({
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant", "dingtalk", "feishu",
    "wecom", "wecom_callback", "weixin", "sms", "email", "webhook",
    "bluebubbles", "qqbot", "yuanbao", ...
})
```

### 比较: CronjobTools vs. hermes CLI cron

| 操作 | CronjobTools (agent tool) | hermes CLI cron | hermes cron --help |
|------|--------------------------|----------------|-------------------|
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

### 交付目标解析

```python
源码位置: gateway/delivery.py:28-107

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

### 交付路由与截断

```python
源码位置: gateway/delivery.py:109-170

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
源码位置: gateway/delivery.py:226-254

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

### 比较: DeliveryRouter 交付模式

| 交付格式 | 示例 | 路由行为 |
|----------|------|----------|
| "origin" | 原始来源平台回投 | 回到创建作业的 chat_id |
| "local" | 本地文件保存 | ~/.hermes/cron/output/{job_id}/{ts}.md |
| "telegram" | Telegram home channel | 投递到 Telegram 默认频道 |
| "telegram:123456" | 特定 Telegram chat | 投递到 chat_id=123456 |
| "discord:789:456" | Discord channel + thread | 投递到特定频道和线程 |
| "slack" | Slack home channel | 投递到 Slack 默认频道 |
| 多目标 | ["origin", "local"] | 依次投递到所有目标 |

---

## Cron CLI (hermes_cli/cron.py)

### 子命令路由

```python
源码位置: hermes_cli/cron.py:270-307

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

### cron_list() 显示格式

```python
源码位置: hermes_cli/cron.py:41-116

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

### cron_status() Gateway 检测

```python
源码位置: hermes_cli/cron.py:132-161

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

### 比较: CLI cron vs. agent cronjob tool

| 维度 | hermes CLI cron | CronjobTools (agent) |
|------|----------------|---------------------|
| 触发方式 | 用户命令行 | LLM 生成 tool_call |
| 输入验证 | argparse | _scan_cron_prompt + _validate_cron_script_path |
| 输出格式 | 彩色终端表格 | JSON string |
| 作业创建 | cron_create(args) | cronjob(action="create", ...) |
| 作业更新 | cron_edit(args) | cronjob(action="update", ...) |
| Gateway 检测 | find_gateway_pids() | N/A |
| 交付参数 | --deliver "telegram:123" | deliver="telegram:123456" |
| 技能参数 | --skill / --skills | skill / skills |

---

## 作业执行安全

### HERMES_CRON_SESSION 环境变量

```python
源码位置: cron/scheduler.py:1018-1021

    # Mark this as a cron session so the approval system can apply cron_mode.
    os.environ["HERMES_CRON_SESSION"] = "1"
```

这触发 `tools/approval.py` 中的 cron 审批模式:

```python
源码位置: tools/approval.py:834-848

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

### 交付平台名验证

```python
源码位置: cron/scheduler.py:74-82

_KNOWN_DELIVERY_PLATFORMS = frozenset({
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant", "dingtalk", "feishu",
    ...
})
```

防止恶意 agent 通过伪造平台名枚举环境变量。

### 比较: cron 安全机制

| 安全机制 | 实现 | 作用 |
|----------|------|------|
| HERMES_CRON_SESSION | 环境变量标记 | 触发审批 cron_mode |
| _KNOWN_DELIVERY_PLATFORMS | frozenset 白名单 | 防止平台名枚举 |
| _scan_cron_prompt | prompt 扫描 | 防止危险 prompt |
| _validate_cron_script_path | 路径验证 | 防止路径遍历 |
| _resolve_cron_enabled_toolsets | 工具集过滤 | 限制 cron 可用工具 |
| workdir 串行化 | 分区并行 | 防止 TERMINAL_CWD 竞态 |
| _secure_dir (0700) | 目录权限 | 保护作业文件 |

---

## 汇总表

| 组件 | 行数 | 职责 |
|------|------|------|
| `cron/scheduler.py` | 1594 | tick() 主循环 + 文件锁 + ThreadPoolExecutor + run_job (no_agent/LLM 路径) |
| `cron/jobs.py` | 1050 | 作业 CRUD + jobs.json 存储 + 调度计算 + 状态追踪 + 文件权限 |
| `tools/cronjob_tools.py` | 662 | agent cron 管理 (7 操作 + prompt/script/context_from 验证) |
| `gateway/delivery.py` | 258 | DeliveryRouter — 目标解析 + 平台路由 + 截断 (4000 chars) |
| `hermes_cli/cron.py` | 307 | CLI cron 子命令 (list/create/edit/pause/resume/run/remove/status/tick) |

---

[← 11 — 技能与自进化系统](/zh-CN/chapters/11-skills-self-improvement) | [→ 13 — ...](/zh-CN/chapters/13-rl-training)