# 13 — RL Training Pipeline: Tinker-Atropos 集成的强化学习全流程编排


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `rl_cli.py` | 446 | RL 专用 CLI 入口 |
| `tools/rl_training_tool.py` | 1396 | 环境发现 + 训练生命周期管理 |
| `environments/hermes_base_env.py` | 714 | 抽象基类环境 |
| `environments/agentic_opd_env.py` | 1214 | On-Policy Distillation 环境 |
| `environments/web_research_env.py` | 719 | Web 研究奖励环境 |
| `environments/agent_loop.py` | 534 | Agent 循环执行器 |
| `environments/tool_context.py` | 473 | 工具上下文传递 |
| `trajectory_compressor.py` | 1508 | 轨迹后处理压缩 |
| `batch_runner.py` | 1287 | 批量并行执行 + 检查点 |
| `mini_swe_runner.py` | 736 | SWE-bench 评估适配器 |
| `toolset_distributions.py` | 364 | 工具集概率分布 |

**One-liner:** RL 训练管线将 Tinker-Atropos GRPO 框架与 Hermes-Agent 工具调用能力桥接，提供从环境发现到训练启动、轨迹压缩、批量执行的完整闭环。

---

## Architecture Overview

```
                    ┌──────────────────────────────────────────────┐
                    │              rl_cli.py (446L)               │
                    │  Fire CLI → AIAgent(RL_SYSTEM_PROMPT,       │
                    │              RL_TOOLSETS=["terminal","web","rl"])│
                    └──────────────────┬──────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────┐
                    │       tools/rl_training_tool.py (1396L)     │
                    │                                              │
                    │  ┌─ AST Scanner ──┐   ┌─ RunState ─────────┐│
                    │  │ _scan_environments│ │  run_id / status  ││
                    │  │ → EnvironmentInfo │ │  3 subprocesses   ││
                    │  └──────────────┘   └─────────────────────┘│
                    │                                              │
                    │  rl_list_environments → rl_select_environment│
                    │  → rl_get_current_config → rl_edit_config   │
                    │  → rl_start_training → rl_check_status      │
                    │  → rl_stop_training → rl_get_results        │
                    └──────────────────┬──────────────────────────┘
                                       │
              ┌────────────────────────▼────────────────────────────┐
              │          Tinker-Atropos Subprocess Spawn            │
              │                                                     │
              │  1. run-api        (Atropos trajectory API:8000)    │
              │  2. launch_training.py (Tinker trainer:8001+GRPO)  │
              │  3. env.py serve   (Atropos environment connect)   │
              └──────────────────────┬─────────────────────────────┘
                                     │
       ┌─────────────────────────────▼──────────────────────────────┐
       │            environments/ (3 benchmark envs)                │
       │                                                            │
       │  ┌─ HermesAgentBaseEnv ──────┐ ┌─ WebResearchEnv ────────┐│
       │  │  setup()                  │ │  FRAMES benchmark       ││
       │  │  get_next_item()          │ │  answer_correctness     ││
       │  │  format_prompt()          │ │  source_diversity        ││
       │  │  compute_reward()         │ │  efficiency penalty      ││
       │  └──────────────────────────┘ └──────────────────────────┘│
       │                                                            │
       │  ┌─ AgenticOPDEnv ──────────────────────────────────────┐│
       │  │  On-Policy Distillation: dense token-level signal     ││
       │  │  (assistant_turn, next_state) → LLM hint → teacher   ││
       │  │  logprobs vs student logprobs → per-token advantage   ││
       │  └──────────────────────────────────────────────────────┘│
       └────────────────────────────┬─────────────────────────────┘
                                    │
       ┌────────────────────────────▼─────────────────────────────┐
       │       Post-Processing Pipeline                           │
       │                                                          │
       │  batch_runner.py → trajectory_compressor.py              │
       │  (parallel exec)     (token-budget compression)          │
       │       │                     │                            │
       │  toolset_distributions.py ←─┘                            │
       │  (probabilistic tool sampling)                           │
       └──────────────────────────────────────────────────────────┘
```

---

## TL;DR

RL 训练管线通过专用 CLI (`rl_cli.py`) 启动带有 RL 系统提示和工具集的 AIAgent，调用 `rl_training_tool.py` 中定义的 8 个异步工具函数完成从环境发现、配置、训练启动到监控的完整流程。训练运行由三个子进程（API server、Trainer、Environment）组成，通过 AST 扫描自动发现 `BaseEnv` 子类。三个基准环境分别覆盖通用工具调用（HermesBaseEnv）、密集 token 级信号（AgenticOPDEnv）和多步 Web 研究（WebResearchEnv）。轨迹后处理通过 `trajectory_compressor.py` 在目标 token 预算内保护头尾回合、压缩中间内容。`batch_runner.py` 提供多进程并行 + 检查点恢复能力，`toolset_distributions.py` 控制工具集的概率采样策略。

---

## 1. RL CLI 入口 (`rl_cli.py`, 446 lines)

### 1.1 启动流程与配置

`rl_cli.py` 是 RL 训练的专用入口，使用 Fire CLI 提供 `--interactive`、`--list-environments`、`--check-server` 等模式。核心设计是将 RL 工作流与通用 CLI 分离，使用扩展的迭代上限和专用的系统提示。

**源码位置: rl_cli.py:65-174**

```python
DEFAULT_MODEL = "anthropic/claude-opus-4.5"
DEFAULT_BASE_URL = OPENROUTER_BASE_URL

RL_MAX_ITERATIONS = 200  # Allow many more iterations for long workflows

RL_SYSTEM_PROMPT = """You are an automated post-training engineer specializing in reinforcement learning for language models.

## Your Capabilities
1. **DISCOVER**: Use `rl_list_environments` to see available RL environments
2. **INSPECT**: Read environment files to understand how they work
3. **CONFIGURE**: Use `rl_select_environment` and `rl_edit_config` to set up training
4. **TEST**: Always use `rl_test_inference` before full training
5. **TRAIN**: Use `rl_start_training` to begin, `rl_check_status` to monitor
"""

RL_TOOLSETS = ["terminal", "web", "rl"]
```

### 1.2 环境检查与交互模式

`check_tinker_atropos()` 验证子模块存在性，`check_requirements()` 校验 API key 组合。交互模式支持 `status` 快捷命令查看活跃训练运行。

**源码位置: rl_cli.py:202-217**

```python
def check_tinker_atropos():
    """Check if tinker-atropos submodule is properly set up."""
    tinker_path = Path(__file__).parent / "tinker-atropos"
    if not tinker_path.exists():
        return False, "tinker-atropos submodule not found. Run: git submodule update --init"
    envs_path = tinker_path / "tinker_atropos" / "environments"
    env_files = list(envs_path.glob("*.py"))
    env_files = [f for f in env_files if not f.name.startswith("_")]
    return True, {"path": str(tinker_path), "environments_count": len(env_files)}
```

### 1.3 Agent 构建与运行

**源码位置: rl_cli.py:369-379**

```python
agent = AIAgent(
    base_url=base_url,
    api_key=api_key,
    model=model,
    max_iterations=max_iterations,
    enabled_toolsets=RL_TOOLSETS,
    save_trajectories=save_trajectories,
    verbose_logging=verbose,
    quiet_mode=False,
    ephemeral_system_prompt=RL_SYSTEM_PROMPT,
)
```

---

## 2. RL Training Tool (`tools/rl_training_tool.py`, 1396 lines)

### 2.1 锁定配置（基础设施参数）

训练工具的核心设计是"锁定字段"机制——基础设施参数（tokenizer、server URL、LoRA rank 等）不允许模型修改，确保训练配置在可控范围内。

**源码位置: tools/rl_training_tool.py:72-106**

```python
LOCKED_FIELDS = {
    "env": {
        "tokenizer_name": "Qwen/Qwen3-8B",
        "rollout_server_url": "http://localhost:8000",
        "use_wandb": True,
        "max_token_length": 8192,
        "max_num_workers": 2048,
        "worker_timeout": 3600,
        "total_steps": 2500,
        "steps_per_eval": 25,
        "max_batches_offpolicy": 3,
        "inference_weight": 1.0,
        "eval_limit_ratio": 0.1,
    },
    "openai": [
        {
            "model_name": "Qwen/Qwen3-8B",
            "base_url": "http://localhost:8001/v1",
            "api_key": "x",
            "weight": 1.0,
            "num_requests_for_eval": 256,
            "timeout": 3600,
            "server_type": "sglang",
        }
    ],
    "tinker": {
        "lora_rank": 32,
        "learning_rate": 0.00004,
        "max_token_trainer_length": 9000,
        "checkpoint_dir": "./temp/",
        "save_checkpoint_interval": 25,
    },
    "slurm": False,
    "testing": False,
}
```

### 2.2 状态管理数据结构

**源码位置: tools/rl_training_tool.py:116-140**

```python
@dataclass
class EnvironmentInfo:
    """Information about a discovered environment."""
    name: str
    class_name: str
    file_path: str
    description: str = ""
    config_class: str = "BaseEnvConfig"

@dataclass
class RunState:
    """State for a training run."""
    run_id: str
    environment: str
    config: Dict[str, Any]
    status: str = "pending"  # pending, starting, running, stopping, stopped, completed, failed
    error_message: str = ""
    wandb_project: str = ""
    wandb_run_name: str = ""
    start_time: float = 0.0
    # Process handles
    api_process: Optional[subprocess.Popen] = None
    trainer_process: Optional[subprocess.Popen] = None
    env_process: Optional[subprocess.Popen] = None
```

### 2.3 AST 扫描环境发现

使用 Python AST 解析扫描 `tinker-atropos/tinker_atropos/environments/` 目录，查找继承 `BaseEnv` 的类，提取 `name`、`env_config_cls` 和 docstring。

**源码位置: tools/rl_training_tool.py:158-216**

```python
def _scan_environments() -> List[EnvironmentInfo]:
    """Scan the environments directory for BaseEnv subclasses using AST."""
    environments = []
    if not ENVIRONMENTS_DIR.exists():
        return environments
    for py_file in ENVIRONMENTS_DIR.glob("*.py"):
        if py_file.name.startswith("_"):
            continue
        try:
            with open(py_file, "r") as f:
                tree = ast.parse(f.read())
            for node in ast.walk(tree):
                if isinstance(node, ast.ClassDef):
                    for base in node.bases:
                        base_name = ""
                        if isinstance(base, ast.Name):
                            base_name = base.id
                        elif isinstance(base, ast.Attribute):
                            base_name = base.attr
                        if base_name == "BaseEnv":
                            # Extract name, env_config_cls, docstring
                            ...
                            environments.append(EnvironmentInfo(
                                name=env_name,
                                class_name=node.name,
                                file_path=str(py_file),
                                description=description,
                                config_class=config_class,
                            ))
                            break
        except Exception as e:
            logger.warning("Could not parse %s: %s", py_file, e)
    return environments
```

### 2.4 动态配置字段发现

`_get_env_config_fields()` 动态导入环境模块，调用 `config_init()` 获取 Pydantic 配置类，提取每个字段的类型、默认值、描述和锁定状态。

**源码位置: tools/rl_training_tool.py:219-296**

```python
def _get_env_config_fields(env_file_path: str) -> Dict[str, Dict[str, Any]]:
    """Dynamically import an environment and extract its config fields."""
    try:
        spec = importlib.util.spec_from_file_location("env_module", env_file_path)
        module = importlib.util.module_from_spec(spec)
        sys.modules["env_module"] = module
        spec.loader.exec_module(module)

        # Find the BaseEnv subclass with config_init
        env_class = None
        for name, obj in vars(module).items():
            if isinstance(obj, type) and name != "BaseEnv":
                if hasattr(obj, "config_init") and callable(getattr(obj, "config_init")):
                    env_class = obj
                    break

        # Try calling config_init to get the actual config class
        config_class = None
        try:
            env_config, server_configs = env_class.config_init()
            config_class = type(env_config)
        except Exception:
            from atroposlib.envs.base import BaseEnvConfig
            config_class = BaseEnvConfig

        # Extract fields from the Pydantic model
        fields = {}
        for field_name, field_info in config_class.model_fields.items():
            is_locked = field_name in LOCKED_FIELD_NAMES
            locked_value = LOCKED_FIELDS.get("env", {}).get(field_name, default)
            current_value = make_serializable(locked_value) if is_locked else default

            fields[field_name] = {
                "type": type_name,
                "default": default,
                "description": description,
                "locked": is_locked,
                "current_value": current_value,
            }
        return fields
```

### 2.5 三进程训练启动

`_spawn_training_run()` 启动三个子进程：Atropos API server (run-api)、Tinker trainer (launch_training.py) 和环境 server。进程间有 5s → 30s → 90s 的等待序列，确保服务依次就绪。

**源码位置: tools/rl_training_tool.py:314-423**

```python
async def _spawn_training_run(run_state: RunState, config_path: Path):
    """Spawn the three processes needed for training."""
    run_id = run_state.run_id
    _ensure_logs_dir()

    # Step 1: Start the Atropos API server (run-api)
    run_state.api_process = subprocess.Popen(
        ["run-api"],
        stdout=api_log_file,
        stderr=subprocess.STDOUT,
        cwd=str(TINKER_ATROPOS_ROOT),
    )
    await asyncio.sleep(5)

    # Step 2: Start the Tinker trainer
    run_state.trainer_process = subprocess.Popen(
        [sys.executable, "launch_training.py", "--config", str(config_path)],
        stdout=trainer_log_file,
        stderr=subprocess.STDOUT,
        cwd=str(TINKER_ATROPOS_ROOT),
        env={**os.environ, "TINKER_API_KEY": os.getenv("TINKER_API_KEY", "")},
    )
    await asyncio.sleep(30)  # Wait for trainer to initialize

    # Step 3: Start the environment
    run_state.env_process = subprocess.Popen(
        [sys.executable, str(env_info.file_path), "serve", "--config", str(config_path)],
        stdout=env_log_file,
        stderr=subprocess.STDOUT,
        cwd=str(TINKER_ATROPOS_ROOT),
    )
    await asyncio.sleep(10)

    run_state.status = "running"
    asyncio.create_task(_monitor_training_run(run_state))
```

### 2.6 状态检查速率限制

`rl_check_status()` 对同一 run_id 强制 30 分钟最小间隔，避免频繁轮询浪费资源。

**源码位置: tools/rl_training_tool.py:815-830**

```python
async def rl_check_status(run_id: str) -> str:
    """Get status and metrics for a training run.
    RATE LIMITED: 30-minute minimum interval between checks."""
    now = time.time()
    last_check = _last_status_check.get(run_id, 0)
    if now - last_check < MIN_STATUS_CHECK_INTERVAL:
        remaining = int(MIN_STATUS_CHECK_INTERVAL - (now - last_check))
        return json.dumps({
            "status": _active_runs[run_id].status,
            "rate_limited": True,
            "next_check_in_seconds": remaining,
            "message": f"Please wait {remaining // 60} more minutes before checking again.",
        })
```

### 2.7 工具函数清单

| Tool Function | Lines | Purpose |
|---------------|-------|---------|
| `rl_list_environments` | 509-551 | AST 扫描环境列表 |
| `rl_select_environment` | 554-607 | 选择环境 + 加载配置 |
| `rl_get_current_config` | 613-660 | 显示当前配置（锁定/可编辑） |
| `rl_edit_config` | 660-710 | 修改可编辑字段 |
| `rl_start_training` | 710-812 | 三进程 spawn + UUID run_id |
| `rl_check_status` | 815-904 | 30min 速率限制的状态查询 |
| `rl_stop_training` | 904-936 | 反序终止进程 |
| `rl_get_results` | 936-980 | WandB metrics 获取 |
| `rl_test_inference` | 1019-1319 | 推理测试 |
| `rl_list_runs` | 980-1019 | 活跃运行列表 |

---

## 3. RL 环境体系 (`environments/`)

### 3.1 抽象基类 HermesAgentBaseEnv

所有 Hermes RL 环境继承 `HermesAgentBaseEnv`，它封装了 Atropos 集成管道：双模式运行（Phase 1 OpenAI server / Phase 2 ManagedServer）、按组工具集采样、Agent 循环编排、ToolContext 创建和 ScoredDataGroup 构建。子类只需实现 5 个方法。

**源码位置: environments/hermes_base_env.py:1-17**

```python
"""
HermesAgentBaseEnv -- Abstract Base Environment for Hermes-Agent + Atropos

Subclasses only need to implement:
    setup()           -- Load dataset, initialize state
    get_next_item()   -- Return the next item from the dataset
    format_prompt()   -- Convert a dataset item into the user message
    compute_reward()  -- Score the rollout (has full ToolContext access)
    evaluate()        -- Periodic evaluation
"""
```

### 3.2 配置类 HermesAgentEnvConfig

**源码位置: environments/hermes_base_env.py:78-157**

```python
class HermesAgentEnvConfig(BaseEnvConfig):
    """Configuration for hermes-agent Atropos environments."""
    # Mutually exclusive: enabled_toolsets OR distribution
    enabled_toolsets: Optional[List[str]] = Field(
        default=None,
        description="Explicit list of hermes toolsets to enable.",
    )
    distribution: Optional[str] = Field(
        default=None,
        description="Name of a toolset distribution from toolset_distributions.py.",
    )
    max_agent_turns: int = Field(default=30)
    system_prompt: Optional[str] = Field(default=None)
    agent_temperature: float = Field(default=1.0)
    terminal_backend: str = Field(default="local")
    terminal_timeout: int = Field(default=120)
    dataset_name: Optional[str] = Field(default=None)
    tool_pool_size: int = Field(default=128)
```

### 3.3 AgenticOPDEnv — On-Policy Distillation

首个支持密集 token 级训练信号的 Atropos 环境。核心思想来自 OpenClaw-RL（Princeton 2026）：每次 agent 收到 next-state 信号（工具结果、错误、测试结果），该信号包含关于上一步如何改进的 hindsight 信息。

**源码位置: environments/agentic_opd_env.py:1-58**

```python
"""
AgenticOPDEnv — On-Policy Distillation for Agentic Tool-Calling Tasks

Key idea (from OpenClaw-RL, Princeton 2026):
  Every time an agent receives a next-state signal (tool result, error trace,
  test verdict), that signal contains hindsight information about how the
  agent's PREVIOUS response could have been better. This environment:

  1. Runs standard agentic rollouts (tool-calling agent loop)
  2. Walks the conversation to find (assistant_turn, next_state) pairs
  3. Uses an LLM judge to extract "hints" from next-state signals
  4. Builds an enhanced prompt (original context + hint)
  5. Scores the student's response tokens under the enhanced distribution
  6. Packages the teacher's top-K predictions as distill_token_ids / distill_logprobs

The trainer then computes per-token advantages:
  A_t = teacher_logprob(token_t) - student_logprob(token_t)
  Positive → teacher approves this token (upweight)
  Negative → teacher disapproves (downweight)
"""
```

### 3.4 WebResearchEnv — 多步 Web 研究

**源码位置: environments/web_research_env.py:1-35**

```python
"""
WebResearchEnv — RL Environment for Multi-Step Web Research

Reward signals:
  - Answer correctness  (LLM judge, 0.0–1.0)
  - Source diversity    (used >=2 distinct domains)
  - Efficiency          (penalizes excessive tool calls)
  - Tool usage          (bonus for actually using web tools)

Dataset: FRAMES benchmark (Google, 2024) — multi-hop factual questions
"""
```

---

## 4. Trajectory Compressor (`trajectory_compressor.py`, 1508 lines)

### 4.1 压缩策略

轨迹压缩器遵循"保护头尾、压缩中间"策略，在目标 token 预算内保留训练信号质量。

**源码位置: trajectory_compressor.py:1-15**

```python
"""
Compression Strategy:
1. Protect first turns (system, human, first gpt, first tool)
2. Protect last N turns (final actions and conclusions)
3. Compress MIDDLE turns only, starting from 2nd tool response
4. Compress only as much as needed to fit under target
5. Replace compressed region with a single human summary message
6. Keep remaining tool calls intact (model continues working after summary)
"""
```

### 4.2 CompressionConfig

**源码位置: trajectory_compressor.py:83-119**

```python
class CompressionConfig:
    """Configuration for trajectory compression."""
    tokenizer_name: str = "moonshotai/Kimi-K2-Thinking"
    target_max_tokens: int = 15250
    summary_target_tokens: int = 750

    # Protected turns
    protect_first_system: bool = True
    protect_first_human: bool = True
    protect_first_gpt: bool = True
    protect_first_tool: bool = True
    protect_last_n_turns: int = 4

    # Summarization (OpenRouter)
    summarization_model: str = "google/gemini-3-flash-preview"
    temperature: float = 0.3
    max_retries: int = 3

    # Processing
    num_workers: int = 4
    max_concurrent_requests: int = 50
    per_trajectory_timeout: int = 300
```

### 4.3 TrajectoryCompressor 类

**源码位置: trajectory_compressor.py:332-430**

```python
class TrajectoryCompressor:
    """
    Compresses agent trajectories to fit within a target token budget.

    Compression strategy:
    1. Keep protected head turns (system, human, first gpt+tool)
    2. Keep protected tail turns (last N turns)
    3. From the compressible middle region, compress only as much as needed
    4. Replace compressed turns with a single human summary message
    """
    def __init__(self, config: CompressionConfig):
        self.config = config
        self.aggregate_metrics = AggregateMetrics()
        self._init_tokenizer()
        self._init_summarizer()
```

### 4.4 LLM 路由初始化

`_init_summarizer()` 使用集中的 provider router 处理认证和 provider 检测，支持自定义端点回退。

**源码位置: trajectory_compressor.py:374-416**

```python
def _init_summarizer(self):
    """Initialize LLM routing for summarization."""
    provider = self._detect_provider()
    if provider:
        self._llm_provider = provider
        self._use_call_llm = True
        from agent.auxiliary_client import resolve_provider_client
        client, _ = resolve_provider_client(provider, model=self.config.summarization_model)
    else:
        # Custom endpoint — raw base_url + api_key_env
        self._use_call_llm = False
        from openai import OpenAI
        self.client = OpenAI(api_key=api_key, base_url=...)
```

---

## 5. Batch Runner (`batch_runner.py`, 1287 lines)

### 5.1 BatchRunner 核心结构

**源码位置: batch_runner.py:515-590**

```python
class BatchRunner:
    """Manages batch processing of agent prompts with checkpointing and statistics."""

    def __init__(
        self,
        dataset_file: str,
        batch_size: int,
        run_name: str,
        distribution: str = "default",
        max_iterations: int = 10,
        model: str = "claude-opus-4-20250514",
        num_workers: int = 4,
        ephemeral_system_prompt: str = None,
        providers_allowed: List[str] = None,
        providers_ignored: List[str] = None,
        max_samples: int = None,
    ):
        # Validate distribution
        if not validate_distribution(distribution):
            raise ValueError(f"Unknown distribution: {distribution}")
```

### 5.2 检查点恢复机制

BatchRunner 支持 checkpoint-based fault tolerance：在每批处理完成后保存检查点，中断后可通过 `--resume` 从上次停止处继续。

### 5.3 轨迹格式

输出的轨迹使用 from/value pairs 格式，与 trajectory_compressor.py 兼容，确保后处理管线可以无缝衔接。

---

## 6. Mini SWE Runner (`mini_swe_runner.py`, 736 lines)

### 6.1 多环境执行

支持三种执行后端（local、docker、modal），输出 Hermes 格式轨迹，与 batch_runner 和 trajectory_compressor 管线兼容。

**源码位置: mini_swe_runner.py:1-27**

```python
"""
SWE Runner with Hermes Trajectory Format

Features:
- Uses Hermes-Agent's Docker, Modal, or Local environments for command execution
- Outputs trajectories in Hermes format (from/value pairs with XML)
- Compatible with the trajectory compression pipeline
- Supports batch processing from JSONL prompt files
"""
```

---

## 7. Toolset Distributions (`toolset_distributions.py`, 364 lines)

### 7.1 概率采样设计

每个分布定义工具集名称到选择概率的映射，用于 batch_runner 的每 prompt 工具集采样。

**源码位置: toolset_distributions.py:29-60**

```python
DISTRIBUTIONS = {
    "default": {
        "description": "All available tools, all the time",
        "toolsets": {
            "web": 100, "vision": 100, "image_gen": 100,
            "terminal": 100, "file": 100, "moa": 100, "browser": 100
        }
    },
    "image_gen": {
        "description": "Heavy focus on image generation with vision and web support",
        "toolsets": {
            "image_gen": 90, "vision": 90, "web": 55,
            "terminal": 45, "moa": 10
        }
    },
    "research": {
        "description": "Web research with vision analysis and reasoning",
        "toolsets": {
            "web": 90, "vision": 70, "terminal": 40, "moa": 30
        }
    },
    # ... more distributions
}
```

---

## Comparison Tables

### Table 1: RL Environment Comparison

| Feature | HermesAgentBaseEnv | AgenticOPDEnv | WebResearchEnv |
|---------|--------------------|---------------|----------------|
| Inheritance | BaseEnv → HermesAgentBaseEnv | HermesAgentBaseEnv | HermesAgentBaseEnv |
| Reward Type | Per-environment `compute_reward()` | Dense token-level (OPD) | Multi-signal composite |
| Training Signal | Sparse (end-of-trajectory) | Dense per-token advantage | Composite (correctness + diversity + efficiency) |
| Phase 2 Required | Optional | Yes (VLLM + ManagedServer) | Optional |
| Dataset | Configurable HF dataset | Coding tasks + built-in fallback | FRAMES benchmark (Google) |
| Tool Integration | Full Hermes toolset | Terminal + file tools | Web + browser tools |
| Key Innovation | ABC pattern for rapid env creation | On-Policy Distillation from OpenClaw-RL | Multi-source reward composition |

### Table 2: Training Process Comparison

| Component | Startup Delay | Log Output | Failure Detection |
|-----------|---------------|------------|-------------------|
| run-api (Atropos API) | 5s | `api_{run_id}.log` | `poll() is not None` |
| launch_training.py (Trainer) | 30s | `trainer_{run_id}.log` | `poll() is not None` |
| env.py serve (Environment) | 100s (90+10) | `env_{run_id}.log` | `poll() is not None` + monitor task |

### Table 3: Trajectory Processing Comparison

| Tool | Input Format | Output Format | Key Feature |
|------|-------------|---------------|-------------|
| batch_runner.py | JSONL (prompt field) | JSONL (from/value pairs) | Checkpoint resume + parallel workers |
| trajectory_compressor.py | JSONL (from/value pairs) | JSONL (compressed) | Head/tail protection + budget compression |
| mini_swe_runner.py | JSONL / single task | JSONL (from/value pairs) | Docker/Modal/Local execution |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `rl_cli.py` | 446 | RL 专用 CLI 入口，Fire CLI + 交互模式 |
| `tools/rl_training_tool.py` | 1396 | AST 环境发现 + 锁定配置 + 三进程训练生命周期 |
| `environments/hermes_base_env.py` | 714 | ABC 基类：5 方法 + Atropos 集成管道 |
| `environments/agentic_opd_env.py` | 1214 | OPD 环境：密集 token 级训练信号 |
| `environments/web_research_env.py` | 719 | Web 研究环境：多信号复合奖励 |
| `environments/agent_loop.py` | 534 | Agent 循环执行器编排 |
| `environments/tool_context.py` | 473 | 工具上下文传递给奖励函数 |
| `trajectory_compressor.py` | 1508 | 轨迹压缩：保护头尾 + LLM 概要中间 |
| `batch_runner.py` | 1287 | 多进程并行 + 检查点恢复 + 统计聚合 |
| `mini_swe_runner.py` | 736 | SWE-bench 评估适配器 |
| `toolset_distributions.py` | 364 | 工具集概率分布定义 |

---

[<< 12 — 定时任务与调度](/zh-CN/chapters/12-cron-scheduling) | [14 — 配置系统 >>](/zh-CN/chapters/14-configuration-system)