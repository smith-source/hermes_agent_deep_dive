# 13 — RL Training Pipeline: Full Reinforcement Learning Workflow Orchestration via Tinker-Atropos Integration


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `rl_cli.py` | 446 | RL-specific CLI entry point |
| `tools/rl_training_tool.py` | 1396 | Environment discovery + training lifecycle management |
| `environments/hermes_base_env.py` | 714 | Abstract base class environment |
| `environments/agentic_opd_env.py` | 1214 | On-Policy Distillation environment |
| `environments/web_research_env.py` | 719 | Web research reward environment |
| `environments/agent_loop.py` | 534 | Agent loop executor |
| `environments/tool_context.py` | 473 | Tool context passing |
| `trajectory_compressor.py` | 1508 | Trajectory post-processing compression |
| `batch_runner.py` | 1287 | Batch parallel execution + checkpointing |
| `mini_swe_runner.py` | 736 | SWE-bench evaluation adapter |
| `toolset_distributions.py` | 364 | Toolset probability distributions |

**One-liner:** The RL training pipeline bridges the Tinker-Atropos GRPO framework with Hermes-Agent's tool-calling capabilities, providing a complete closed loop from environment discovery to training launch, trajectory compression, and batch execution.

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

The RL training pipeline launches an AIAgent with RL system prompt and toolsets via a dedicated CLI (`rl_cli.py`), invoking 8 async tool functions defined in `rl_training_tool.py` to complete the full workflow from environment discovery, configuration, training launch to monitoring. A training run consists of three subprocesses (API server, Trainer, Environment), with `BaseEnv` subclasses automatically discovered via AST scanning. Three benchmark environments cover general tool-calling (HermesBaseEnv), dense token-level signals (AgenticOPDEnv), and multi-step web research (WebResearchEnv). Trajectory post-processing uses `trajectory_compressor.py` to protect head/tail turns and compress middle content within a target token budget. `batch_runner.py` provides multi-process parallel execution + checkpoint recovery, and `toolset_distributions.py` controls probabilistic toolset sampling strategies.

---

## 1. RL CLI Entry Point (`rl_cli.py`, 446 lines)

### 1.1 Startup Flow and Configuration

`rl_cli.py` is the dedicated entry point for RL training, using Fire CLI to provide `--interactive`, `--list-environments`, `--check-server` modes. The core design separates the RL workflow from the general CLI, using extended iteration limits and a dedicated system prompt.

**Source location: rl_cli.py:65-174**

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

### 1.2 Environment Check and Interactive Mode

`check_tinker_atropos()` verifies the submodule exists, `check_requirements()` validates API key combinations. Interactive mode supports `status` shortcut command to view active training runs.

**Source location: rl_cli.py:202-217**

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

### 1.3 Agent Construction and Execution

**Source location: rl_cli.py:369-379**

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

### 2.1 Locked Configuration (Infrastructure Parameters)

The core design of the training tool is a "locked fields" mechanism — infrastructure parameters (tokenizer, server URL, LoRA rank, etc.) are not allowed to be modified by the model, ensuring training configuration remains within controllable bounds.

**Source location: tools/rl_training_tool.py:72-106**

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

### 2.2 State Management Data Structures

**Source location: tools/rl_training_tool.py:116-140**

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

### 2.3 AST Scanning Environment Discovery

Uses Python AST parsing to scan the `tinker-atropos/tinker_atropos/environments/` directory, finding classes that inherit `BaseEnv` and extracting `name`, `env_config_cls`, and docstring.

**Source location: tools/rl_training_tool.py:158-216**

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

### 2.4 Dynamic Configuration Field Discovery

`_get_env_config_fields()` dynamically imports the environment module, calls `config_init()` to obtain the Pydantic config class, and extracts each field's type, default value, description, and locked status.

**Source location: tools/rl_training_tool.py:219-296**

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

### 2.5 Three-Process Training Launch

`_spawn_training_run()` launches three subprocesses: Atropos API server (run-api), Tinker trainer (launch_training.py), and environment server. Inter-process wait sequence is 5s → 30s → 90s, ensuring services become ready in order.

**Source location: tools/rl_training_tool.py:314-423**

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

### 2.6 Status Check Rate Limiting

`rl_check_status()` enforces a 30-minute minimum interval for the same run_id, preventing frequent polling that wastes resources.

**Source location: tools/rl_training_tool.py:815-830**

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

### 2.7 Tool Function List

| Tool Function | Lines | Purpose |
|---------------|-------|---------|
| `rl_list_environments` | 509-551 | AST scan environment list |
| `rl_select_environment` | 554-607 | Select environment + load config |
| `rl_get_current_config` | 613-660 | Display current config (locked/editable) |
| `rl_edit_config` | 660-710 | Modify editable fields |
| `rl_start_training` | 710-812 | Three-process spawn + UUID run_id |
| `rl_check_status` | 815-904 | 30min rate-limited status query |
| `rl_stop_training` | 904-936 | Reverse-order process termination |
| `rl_get_results` | 936-980 | WandB metrics retrieval |
| `rl_test_inference` | 1019-1319 | Inference testing |
| `rl_list_runs` | 980-1019 | Active runs list |

---

## 3. RL Environment System (`environments/`)

### 3.1 Abstract Base Class HermesAgentBaseEnv

All Hermes RL environments inherit from `HermesAgentBaseEnv`, which encapsulates the Atropos integration pipeline: dual-mode execution (Phase 1 OpenAI server / Phase 2 ManagedServer), per-group toolset sampling, agent loop orchestration, ToolContext creation, and ScoredDataGroup construction. Subclasses only need to implement 5 methods.

**Source location: environments/hermes_base_env.py:1-17**

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

### 3.2 Configuration Class HermesAgentEnvConfig

**Source location: environments/hermes_base_env.py:78-157**

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

The first Atropos environment supporting dense token-level training signals. The core idea comes from OpenClaw-RL (Princeton 2026): every time an agent receives a next-state signal (tool result, error, test result), that signal contains hindsight information about how the previous step could have been improved.

**Source location: environments/agentic_opd_env.py:1-58**

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

### 3.4 WebResearchEnv — Multi-Step Web Research

**Source location: environments/web_research_env.py:1-35**

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

### 4.1 Compression Strategy

The trajectory compressor follows a "protect head/tail, compress middle" strategy, preserving training signal quality within a target token budget.

**Source location: trajectory_compressor.py:1-15**

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

**Source location: trajectory_compressor.py:83-119**

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

### 4.3 TrajectoryCompressor Class

**Source location: trajectory_compressor.py:332-430**

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

### 4.4 LLM Routing Initialization

`_init_summarizer()` uses a centralized provider router for authentication and provider detection, with custom endpoint fallback support.

**Source location: trajectory_compressor.py:374-416**

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

### 5.1 BatchRunner Core Structure

**Source location: batch_runner.py:515-590**

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

### 5.2 Checkpoint Recovery Mechanism

BatchRunner supports checkpoint-based fault tolerance: saves checkpoints after each batch, enabling `--resume` to continue from the last stop point after an interruption.

### 5.3 Trajectory Format

Output trajectories use from/value pairs format, compatible with trajectory_compressor.py, ensuring the post-processing pipeline can connect seamlessly.

---

## 6. Mini SWE Runner (`mini_swe_runner.py`, 736 lines)

### 6.1 Multi-Environment Execution

Supports three execution backends (local, docker, modal), outputs Hermes-format trajectories compatible with the batch_runner and trajectory_compressor pipeline.

**Source location: mini_swe_runner.py:1-27**

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

### 7.1 Probabilistic Sampling Design

Each distribution defines a mapping from toolset name to selection probability, used for per-prompt toolset sampling in batch_runner.

**Source location: toolset_distributions.py:29-60**

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
| `rl_cli.py` | 446 | RL-specific CLI entry point, Fire CLI + interactive mode |
| `tools/rl_training_tool.py` | 1396 | AST environment discovery + locked config + three-process training lifecycle |
| `environments/hermes_base_env.py` | 714 | ABC base class: 5 methods + Atropos integration pipeline |
| `environments/agentic_opd_env.py` | 1214 | OPD environment: dense token-level training signals |
| `environments/web_research_env.py` | 719 | Web research environment: multi-signal composite reward |
| `environments/agent_loop.py` | 534 | Agent loop executor orchestration |
| `environments/tool_context.py` | 473 | Tool context passing to reward functions |
| `trajectory_compressor.py` | 1508 | Trajectory compression: protect head/tail + LLM summary for middle |
| `batch_runner.py` | 1287 | Multi-process parallel + checkpoint recovery + statistics aggregation |
| `mini_swe_runner.py` | 736 | SWE-bench evaluation adapter |
| `toolset_distributions.py` | 364 | Toolset probability distribution definitions |

---

[<< 12 — Cron Scheduling](/en/chapters/12-cron-scheduling) | [14 — Configuration System >>](/en/chapters/14-configuration-system)