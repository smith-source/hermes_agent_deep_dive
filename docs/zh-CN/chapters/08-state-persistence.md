# 08 — 状态与持久化: 从会话存储到 Kanban 任务追踪的多层持久化体系

[← 07-前端交互界面](/zh-CN/chapters/07-frontend-interfaces) | [→ 09-上下文与记忆引擎](/zh-CN/chapters/09-context-memory)

---

## 源码文件

| 文件 | 行数 | 说明 |
|------|------|------|
| `hermes_state.py` | 2,669 | SessionDB — SQLite + WAL + FTS5 全文搜索, schema v11 |
| `hermes_cli/kanban_db.py` | 4,087 | Kanban DB — Task/Run/Comment/Event 多表, circuit breaker |
| `cron/jobs.py` | 1,050 | Cron job 存储 — jobs.json, schedule 解析, next_run 计算 |
| `tools/checkpoint_manager.py` | 1,638 | Checkpoint — Git shadow repo 自动快照 |
| `tools/process_registry.py` | 1,434 | Process Registry — spawn/wait/kill 进程生命周期管理 |

---

## 一句话总结

Hermes 通过五个独立的持久化子系统 (SessionDB, Kanban DB, Cron Jobs, CheckpointManager, ProcessRegistry) 管理 AI 会话历史、任务追踪、定时调度、文件快照和进程状态, 各子系统根据数据特性选择不同存储引擎 (SQLite, JSON, Git shadow repo, in-memory + detached)。

---

## Architecture Overview

```
                    ┌───────────────────────────────────────┐
                    │         ~/.hermes/                    │
                    │                                       │
   ┌────────────────┼──────────┐  ┌────────────────────────┼──────────┐
   │   state.db     │ SQLite   │  │   kanban.db            │ SQLite   │
   │                │ WAL      │  │                        │ WAL      │
   │  sessions      │          │  │  tasks                 │          │
   │  messages      │          │  │  task_links            │          │
   │  state_meta    │          │  │  task_comments          │          │
   │  messages_fts  │ FTS5     │  │  task_events           │          │
   │  messages_fts_trigram │ FTS5 tri │  │  task_runs             │          │
   │  schema_version │ (v11)    │  │  kanban_notify_subs    │          │
   └────────────────┘          │  └────────────────────────┘          │
                               │                                       │
   ┌───────────────────────────┼──────────┐  ┌─────────────────────────┼──────────┐
   │   cron/jobs.json          │ JSON     │  │   .hermes-checkpoints/  │ Git      │
   │                           │          │  │   (shadow repo)         │ shadow   │
   │  jobs[]                   │          │  │                         │          │
   │  updated_at               │          │  │  per-project git repo   │          │
   │                           │          │  │  with checkpoint tags   │          │
   └───────────────────────────┘          │  └─────────────────────────┘          │
                                          │                                       │
   ┌──────────────────────────────────────┼──────────────────────────────────────┐
   │   ProcessRegistry                    │ in-memory + detached session probe   │
   │                                      │                                      │
   │  ProcessSession[]                    │ spawn, poll, wait, kill              │
   │  local PTY / env subprocess          │                                      │
   └──────────────────────────────────────┘                                      │
                                          └───────────────────────────────────────┘
```

---

## TL;DR

Hermes 的状态持久化由五个独立子系统构成。SessionDB (`hermes_state.py`, 2,669 行) 用 SQLite + WAL 模式存储会话/消息, 配合 FTS5 双表 (unicode61 + trigram) 实现中英文全文搜索, schema v11 含 4 个表 + 6 个触发器。Kanban DB (`kanban_db.py`, 4,087 行) 用 SQLite 管理 Task/Run/Comment/Event 四表, 带 circuit breaker 防止连续失败雪崩。Cron Jobs (`jobs.py`, 1,050 行) 用 JSON 文件存储定时任务, atomic write + fsync 保证持久性。CheckpointManager (`checkpoint_manager.py`, 1,638 行) 用 Git shadow repo 按目录做文件快照, 每 turn dedup, per-project 独立 repo。ProcessRegistry (`process_registry.py`, 1,434 行) 管理 spawn/wait/kill 的进程生命周期, 支持 local PTY 和 env subprocess 两种 spawn 模式。

---

## 1. SessionDB — 会话与消息的持久化核心

### 1.1 Schema v11 定义

SessionDB 使用 schema version 11, 包含 4 个数据表 + 2 个 FTS5 虚拟表 + 6 个同步触发器:

源码位置: hermes_state.py:36-156

```python
SCHEMA_VERSION = 11

SCHEMA_SQL = """
CREATE TABLE IF NOT EXISTS schema_version (
    version INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    user_id TEXT,
    model TEXT,
    model_config TEXT,
    system_prompt TEXT,
    parent_session_id TEXT,
    started_at REAL NOT NULL,
    ended_at REAL,
    end_reason TEXT,
    message_count INTEGER DEFAULT 0,
    tool_call_count INTEGER DEFAULT 0,
    input_tokens INTEGER DEFAULT 0,
    output_tokens INTEGER DEFAULT 0,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,
    reasoning_tokens INTEGER DEFAULT 0,
    billing_provider TEXT,
    billing_base_url TEXT,
    billing_mode TEXT,
    estimated_cost_usd REAL,
    actual_cost_usd REAL,
    cost_status TEXT,
    cost_source TEXT,
    pricing_version TEXT,
    title TEXT,
    api_call_count INTEGER DEFAULT 0,
    FOREIGN KEY (parent_session_id) REFERENCES sessions(id)
);

CREATE TABLE IF NOT EXISTS messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL REFERENCES sessions(id),
    role TEXT NOT NULL,
    content TEXT,
    tool_call_id TEXT,
    tool_calls TEXT,
    tool_name TEXT,
    timestamp REAL NOT NULL,
    token_count INTEGER,
    finish_reason TEXT,
    reasoning TEXT,
    reasoning_content TEXT,
    reasoning_details TEXT,
    codex_reasoning_items TEXT,
    codex_message_items TEXT
);

CREATE TABLE IF NOT EXISTS state_meta (
    key TEXT PRIMARY KEY,
    value TEXT
);
```

### 1.2 关键索引

SessionDB 定义了 4 个核心索引，覆盖所有高频查询路径:

源码位置: hermes_state.py:97-100

```sql
CREATE INDEX IF NOT EXISTS idx_sessions_source ON sessions(source);
CREATE INDEX IF NOT EXISTS idx_sessions_parent ON sessions(parent_session_id);
CREATE INDEX IF NOT EXISTS idx_sessions_started ON sessions(started_at DESC);
CREATE INDEX IF NOT EXISTS idx_messages_session ON messages(session_id, timestamp);
```

| 索引名 | 表 | 列 | 用途 |
|--------|-----|-----|------|
| `idx_sessions_source` | sessions | source | 按平台来源筛选会话 |
| `idx_sessions_parent` | sessions | parent_session_id | 父子会话关系查询 (/resume, /branch) |
| `idx_sessions_started` | sessions | started_at DESC | 时间排序 + 最近会话列表 |
| `idx_messages_session` | messages | (session_id, timestamp) | 单会话消息时间线查询 |

此外，schema migration 过程中会按需创建辅助索引（如 `idx_sessions_title_unique`、`idx_telegram_dm_topic_bindings_user`），这些不在初始 SCHEMA_SQL 中，而是在 _reconcile_columns 运行时根据表结构补充。

### 1.3 FTS5 全文搜索 — 双表策略

SessionDB 实现了两个 FTS5 虚拟表以支持中文/日文/韩文 (CJK) 子串搜索。`messages_fts` 使用默认 unicode61 tokenizer (适合英文), `messages_fts_trigram` 使用 trigram tokenizer (适合 CJK):

源码位置: hermes_state.py:103-156

```python
FTS_SQL = """
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts USING fts5(
    content
);

CREATE TRIGGER IF NOT EXISTS messages_fts_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts(rowid, content) VALUES (
        new.id,
        COALESCE(new.content, '') || ' ' || COALESCE(new.tool_name, '') || ' ' || COALESCE(new.tool_calls, '')
    );
END;

CREATE TRIGGER IF NOT EXISTS messages_fts_delete AFTER DELETE ON messages BEGIN
    DELETE FROM messages_fts WHERE rowid = old.id;
END;

CREATE TRIGGER IF NOT EXISTS messages_fts_update AFTER UPDATE ON messages BEGIN
    DELETE FROM messages_fts WHERE rowid = old.id;
    INSERT INTO messages_fts(rowid, content) VALUES (
        new.id,
        COALESCE(new.content, '') || ' ' || COALESCE(new.tool_name, '') || ' ' || COALESCE(new.tool_calls, '')
    );
END;
"""

FTS_TRIGRAM_SQL = """
CREATE VIRTUAL TABLE IF NOT EXISTS messages_fts_trigram USING fts5(
    content,
    tokenize='trigram'
);

CREATE TRIGGER IF NOT EXISTS messages_fts_trigram_insert AFTER INSERT ON messages BEGIN
    INSERT INTO messages_fts_trigram(rowid, content) VALUES (
        new.id,
        COALESCE(new.content, '') || ' ' || COALESCE(new.tool_name, '') || ' ' || COALESCE(new.tool_calls, '')
    );
END;
... (delete/update triggers analogous)
"""
```

设计理由: unicode61 tokenizer 将 CJK 字符拆分为独立单字 token, 破坏了短语匹配; trigram tokenizer 创建重叠的 3 字节序列, 使得子串查询对任何书写系统 (CJK, Thai 等) 都原生生效。

### 1.3 WAL 模式 + 应用层 Jitter 重试

SessionDB 在多进程共享 (gateway + CLI + worktree agents) 场景下, WAL write-lock 争夺会导致 TUI 冻结。SQLite 内置的 deterministic sleep 会产生 convoy 效应。解决方案是短 timeout (1s) + 应用层随机 jitter 重试:

源码位置: hermes_state.py:159-238

```python
class SessionDB:
    """SQLite-backed session storage with FTS5 search."""

    _WRITE_MAX_RETRIES = 15
    _WRITE_RETRY_MIN_S = 0.020   # 20ms
    _WRITE_RETRY_MAX_S = 0.150   # 150ms
    _CHECKPOINT_EVERY_N_WRITES = 50

    def __init__(self, db_path: Path = None):
        self.db_path = db_path or DEFAULT_DB_PATH
        self._lock = threading.Lock()
        self._write_count = 0
        self._conn = sqlite3.connect(
            str(self.db_path),
            check_same_thread=False,
            timeout=1.0,          # Short timeout — app-level retry handles contention
            isolation_level=None, # Autocommit: we manage transactions ourselves
        )
        self._conn.row_factory = sqlite3.Row
        self._conn.execute("PRAGMA journal_mode=WAL")
        self._conn.execute("PRAGMA foreign_keys=ON")
        self._init_schema()

    def _execute_write(self, fn: Callable[[sqlite3.Connection], T]) -> T:
        """Execute a write transaction with BEGIN IMMEDIATE and jitter retry.

        BEGIN IMMEDIATE acquires the WAL write lock at transaction start,
        not at commit time. On 'database is locked', release the Python lock,
        sleep random 20-150ms, and retry — breaking the convoy pattern.
        """
        last_err: Optional[Exception] = None
        for attempt in range(self._WRITE_MAX_RETRIES):
            try:
                with self._lock:
                    self._conn.execute("BEGIN IMMEDIATE")
                    try:
                        result = fn(self._conn)
                        self._conn.commit()
                    except BaseException:
                        try:
                            self._conn.rollback()
                        except Exception:
                            pass
                        raise
                self._write_count += 1
                ...
```

### 1.4 Schema Migration 系统 — 声明式列协商

SessionDB 实现 additive migration 系统 — 采用 Beets/sqlite-utils 的声明式协商模式: SCHEMA_SQL 是唯一权威定义, `_reconcile_columns()` 在每次启动时比对声明列与实际列, 自动 ADD 缺失列。新增列只需修改 SCHEMA_SQL, 不需要版本号迁移块。

源码位置: hermes_state.py:297-396

#### _parse_schema_columns — 用 SQLite 本身解析 DDL

`_parse_schema_columns()` 使用 in-memory SQLite 连接解析 SCHEMA_SQL, 通过 `PRAGMA table_info()` 提取每个表的列元数据 (类型、NOT NULL、默认值)。这避免了 regex 解析 DDL 的边缘情况 (DEFAULT 表达式含逗号、inline REFERENCES、CHECK 约束等):

```python
@staticmethod
def _parse_schema_columns(schema_sql: str) -> Dict[str, Dict[str, str]]:
    """Extract expected columns per table from SCHEMA_SQL.

    Uses an in-memory SQLite database to parse the SQL — SQLite itself
    handles all syntax (DEFAULT expressions with commas, inline
    REFERENCES, CHECK constraints, etc.) so there are zero regex
    edge cases.  The in-memory DB is opened, the schema DDL is
    executed, and PRAGMA table_info extracts the column metadata.

    Adding a column to SCHEMA_SQL is all that's needed; the
    reconciliation loop picks it up automatically.
    """
    ref = sqlite3.connect(":memory:")
    try:
        ref.executescript(schema_sql)
        table_columns: Dict[str, Dict[str, str]] = {}
        for (tbl,) in ref.execute(
            "SELECT name FROM sqlite_master "
            "WHERE type='table' AND name NOT LIKE 'sqlite_%'"
        ).fetchall():
            cols: Dict[str, str] = {}
            for row in ref.execute(f'PRAGMA table_info("{tbl}")').fetchall():
                col_name = row[1]
                col_type = row[2] or ""
                notnull = row[3]
                default = row[4]
                pk = row[5]
                parts = [col_type] if col_type else []
                if notnull and not pk:
                    parts.append("NOT NULL")
                if default is not None:
                    parts.append(f"DEFAULT {default}")
                cols[col_name] = " ".join(parts)
            table_columns[tbl] = cols
        return table_columns
    finally:
        ref.close()
```

#### _reconcile_columns — 声明式列自动补充

`_reconcile_columns()` 对每个表比对 `_parse_schema_columns()` 输出的声明列集与 `PRAGMA table_info()` 返回的实际列集, 自动执行 `ALTER TABLE ADD COLUMN` 补缺:

```python
def _reconcile_columns(self, cursor: sqlite3.Cursor) -> None:
    """Ensure live tables have every column declared in SCHEMA_SQL.

    Follows the Beets/sqlite-utils pattern: the CREATE TABLE definition
    in SCHEMA_SQL is the single source of truth for the desired schema.
    On every startup this method diffs the live columns (via PRAGMA
    table_info) against the declared columns, and ADDs any that are
    missing.

    This makes column additions a declarative operation — just add
    the column to SCHEMA_SQL and it appears on the next startup.
    Version-gated migration blocks are no longer needed for ADD COLUMN.
    """
    expected = self._parse_schema_columns(SCHEMA_SQL)
    for table_name, declared_cols in expected.items():
        try:
            rows = cursor.execute(
                f'PRAGMA table_info("{table_name}")'
            ).fetchall()
        except sqlite3.OperationalError:
            continue
        live_cols = set()
        for row in rows:
            name = row[1] if isinstance(row, (tuple, list)) else row["name"]
            live_cols.add(name)

        for col_name, col_type in declared_cols.items():
            if col_name not in live_cols:
                safe_name = col_name.replace('"', '""')
                try:
                    cursor.execute(
                        f'ALTER TABLE "{table_name}" ADD COLUMN "{safe_name}" {col_type}'
                    )
                except sqlite3.OperationalError as exc:
                    logger.debug("reconcile %s.%s: %s", table_name, col_name, exc)
```

#### _init_schema — 启动时完整流程

`_init_schema()` 依次执行: (1) CREATE TABLE/FTS SQL → (2) _reconcile_columns → (3) schema_version 更新。列添加从此是声明式操作, 不再需要版本号迁移块:

```python
def _init_schema(self):
    """Create tables and FTS if they don't exist, reconcile columns.

    Schema management follows the declarative reconciliation pattern
    (Beets, sqlite-utils): SCHEMA_SQL is the single source of truth.
    On existing databases, _reconcile_columns() diffs live columns
    against SCHEMA_SQL and ADDs any missing ones.  This eliminates
    the version-gated migration chain for column additions, making
    it impossible for reordered or inserted migrations to skip columns.

    The schema_version table is retained for future data migrations
    (transforming existing rows) which cannot be handled declaratively.
    """
    cursor = self._conn.cursor()
    self._conn.executescript(SCHEMA_SQL)
    self._conn.executescript(FTS_SQL)
    self._conn.executescript(FTS_TRIGRAM_SQL)
    self._reconcile_columns(cursor)
    cursor.execute("SELECT version FROM schema_version")
    row = cursor.fetchone()
    if not row:
        cursor.execute("INSERT INTO schema_version (version) VALUES (?)", (SCHEMA_VERSION,))
    elif row[0] < SCHEMA_VERSION:
        cursor.execute("UPDATE schema_version SET version = ?", (SCHEMA_VERSION,))
```

### 1.5 FTS5 查询消毒

用户搜索输入需要消毒以防 FTS5 语法注入:

源码位置: hermes_state.py:1625

```python
def _sanitize_fts5_query(query: str) -> str:
    """Sanitize a user search string for safe FTS5 query."""
    ...
```

---

## 2. Kanban DB — 任务追踪与 Circuit Breaker

### 2.1 数据模型

Kanban DB 定义 4 个数据类和 6 个表:

源码位置: kanban_db.py:556-838

```python
class Task:
    """In-memory view of a row from the ``tasks`` table."""
    id: str
    title: str
    body: Optional[str]
    assignee: Optional[str]
    status: str
    priority: int
    created_by: Optional[str]
    created_at: int
    started_at: Optional[int]
    completed_at: Optional[int]
    workspace_kind: str          # 'scratch' (default)
    workspace_path: Optional[str]
    claim_lock: Optional[str]
    claim_expires: Optional[int]
    tenant: Optional[str]
    result: Optional[str] = None
    idempotency_key: Optional[str] = None
    consecutive_failures: int = 0    # Unified counter: spawn, timeout, crash
    worker_pid: Optional[int] = None
    last_failure_error: Optional[str] = None
    max_runtime_seconds: Optional[int] = None
    last_heartbeat_at: Optional[int] = None
    current_run_id: Optional[int] = None
    workflow_template_id: Optional[str] = None
    current_step_key: Optional[str] = None
    skills: Optional[list] = None   # Force-loaded skills for worker

class Run:
    """Historical attempt record for a task."""
    ...

class Comment:
    """User comment on a task."""
    id: int
    task_id: str
    author: str
    body: str
    created_at: int

class Event:
    """Structured event log for a task."""
    id: int
    task_id: str
    run_id: Optional[int]
    kind: str
    payload: Optional[str]
    created_at: int
```

### 2.2 表结构

Kanban DB 包含 6 个表, `tasks` 是主表, `task_runs` 记录每次尝试的历史:

源码位置: kanban_db.py:740-838

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id                   TEXT PRIMARY KEY,
    title                TEXT NOT NULL,
    body                 TEXT,
    assignee             TEXT,
    status               TEXT NOT NULL,
    priority             INTEGER DEFAULT 0,
    created_by           TEXT,
    created_at           INTEGER NOT NULL,
    started_at           INTEGER,
    completed_at         INTEGER,
    workspace_kind       TEXT NOT NULL DEFAULT 'scratch',
    workspace_path       TEXT,
    claim_lock           TEXT,
    claim_expires        INTEGER,
    tenant               TEXT,
    result               TEXT,
    idempotency_key      TEXT,
    consecutive_failures INTEGER NOT NULL DEFAULT 0,
    worker_pid           INTEGER,
    last_failure_error   TEXT,
    max_runtime_seconds  INTEGER,
    last_heartbeat_at    INTEGER,
    current_run_id       INTEGER,
    workflow_template_id TEXT,
    current_step_key     TEXT,
    skills               TEXT
);

CREATE TABLE IF NOT EXISTS task_links (
    parent_id  TEXT NOT NULL,
    child_id   TEXT NOT NULL,
    PRIMARY KEY (parent_id, child_id)
);

CREATE TABLE IF NOT EXISTS task_comments (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id    TEXT NOT NULL,
    author     TEXT NOT NULL,
    body       TEXT NOT NULL,
    created_at INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS task_events (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id    TEXT NOT NULL,
    run_id     INTEGER,
    kind       TEXT NOT NULL,
    payload    TEXT,
    created_at INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS task_runs (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    task_id             TEXT NOT NULL,
    profile             TEXT,
    step_key            TEXT,
    status              TEXT NOT NULL,
    -- status: running | done | blocked | crashed | timed_out | failed | released
    claim_lock          TEXT,
    ...
);

CREATE TABLE IF NOT EXISTS kanban_notify_subs (
    ...
);
```

### 2.3 Circuit Breaker — 连续失败防护

`Task.consecutive_failures` 是统一计数器, 在 spawn 失败、timeout、crash 时递增, 仅在成功完成时重置为 0。Circuit breaker 在 `_record_task_failure` 中触发:

源码位置: kanban_db.py:556-598

```
# Unified consecutive-failure counter. Incremented on spawn
# failure, timeout, or crash; reset only on successful completion.
# The circuit breaker in _record_task_failure trips when this
# exceeds DEFAULT_FAILURE_LIMIT consecutive non-successes.
consecutive_failures: int = 0
```

---

## 3. Cron Jobs — 定时任务调度

### 3.1 JSON 存储与 Atomic Write

Cron jobs 使用 `~/.hermes/cron/jobs.json` 存储, 通过 atomic write (mkstemp + fsync + rename) 保证写入原子性:

源码位置: cron/jobs.py:341-386

```python
JOBS_FILE = ...

def load_jobs() -> List[Dict[str, Any]]:
    """Load all jobs from storage."""
    ensure_dirs()
    if not JOBS_FILE.exists():
        return []
    try:
        with open(JOBS_FILE, 'r', encoding='utf-8') as f:
            data = json.load(f)
            return data.get("jobs", [])
    except json.JSONDecodeError:
        # Retry with strict=False to handle bare control chars
        try:
            with open(JOBS_FILE, 'r', encoding='utf-8') as f:
                data = json.loads(f.read(), strict=False)
                jobs = data.get("jobs", [])
                if jobs:
                    save_jobs(jobs)  # Auto-repair
                    logger.warning("Auto-repaired jobs.json")
                return jobs
        except Exception as e:
            raise RuntimeError(f"Cron database corrupted and unrepairable: {e}") from e

def save_jobs(jobs: List[Dict[str, Any]]):
    """Save all jobs to storage."""
    ensure_dirs()
    fd, tmp_path = tempfile.mkstemp(dir=str(JOBS_FILE.parent), suffix='.tmp', prefix='.jobs_')
    try:
        with os.fdopen(fd, 'w', encoding='utf-8') as f:
            json.dump({"jobs": jobs, "updated_at": _hermes_now().isoformat()}, f, indent=2)
            f.flush()
            os.fsync(f.fileno())
        atomic_replace(tmp_path, JOBS_FILE)
        _secure_file(JOBS_FILE)
    except BaseException:
        try: os.unlink(tmp_path)
        except OSError: pass
        raise
```

### 3.2 Schedule 解析

`parse_schedule()` 支持多种 cron 表达式格式, 并计算 next_run:

源码位置: cron/jobs.py:124-340

```python
def parse_schedule(schedule: str) -> Dict[str, Any]:
    """Parse a cron schedule expression into structured dict."""
    ...

def compute_next_run(schedule: Dict[str, Any], last_run_at: Optional[str] = None) -> Optional[str]:
    """Compute the next run timestamp for a schedule."""
    ...
```

### 3.3 Job 生命周期管理

Cron jobs 的 CRUD 操作和运行追踪:

| 函数 | 行数 | 职责 |
|------|------|------|
| `create_job()` | 422-577 | 创建新 job, 支持 prompt/script/mode 等参数 |
| `get_job()` | 578-586 | 按 ID 获取单个 job |
| `list_jobs()` | 587-594 | 列出所有 jobs (含 disabled) |
| `update_job()` | 595-642 | 更新 job 属性 |
| `pause_job()` | 643-655 | 暂停 job |
| `resume_job()` | 656-674 | 恢复暂停 job |
| `trigger_job()` | 675-691 | 手动触发 job |
| `remove_job()` | 692-702 | 删除 job |
| `mark_job_run()` | 703-775 | 记录运行结果 (success/error) |
| `advance_next_run()` | 776-804 | 推进 next_run 时间戳 |
| `get_due_jobs()` | 805-817 | 获取应运行的 jobs |
| `save_job_output()` | 907-938 | 保存 job 输出日志 |

---

## 4. CheckpointManager — Git Shadow Repo 文件快照

### 4.1 设计原理

CheckpointManager 在每 turn 开始时 dedup, 在文件变更工具调用前 ensure_checkpoint。它使用 Git shadow repo (每个项目目录一个独立 repo) 做 add+commit 快照, 并限制每个目录最多 20 个快照、总大小 500MB:

源码位置: checkpoint_manager.py:573-632

```python
class CheckpointManager:
    """Manages automatic filesystem checkpoints.

    Designed to be owned by AIAgent. Call ``new_turn()`` at the start of
    each conversation turn and ``ensure_checkpoint(dir, reason)`` before
    any file-mutating tool call. The manager deduplicates so at most one
    snapshot is taken per directory per turn.

    Parameters
    ----------
    enabled : bool — Master switch (from config / CLI flag).
    max_snapshots : int — Keep at most this many checkpoints per directory.
    max_total_size_mb : int — Hard ceiling on total store size.
    max_file_size_mb : int — Skip adding any single file larger than this.
    """

    def __init__(
        self,
        enabled: bool = False,
        max_snapshots: int = 20,
        max_total_size_mb: int = 500,
        max_file_size_mb: int = 10,
    ):
        self.enabled = enabled
        self.max_snapshots = max(1, int(max_snapshots))
        self.max_total_size_mb = max(0, int(max_total_size_mb))
        self.max_file_size_mb = max(0, int(max_file_size_mb))
        self._checkpointed_dirs: Set[str] = set()
        self._git_available: Optional[bool] = None  # lazy probe
```

### 4.2 Turn Lifecycle

Checkpoint 在每 turn 开始重置 dedup set, 只在第一个 ensure_checkpoint 调用时实际执行:

```python
def new_turn(self) -> None:
    """Reset per-turn dedup. Call at the start of each agent iteration."""
    self._checkpointed_dirs.clear()

def ensure_checkpoint(self, working_dir: str, reason: str = "auto") -> bool:
    """Take a checkpoint if enabled and not already done this turn.

    Returns True if a checkpoint was taken, False otherwise.
    Never raises — all errors are silently logged.
    """
    if not self.enabled:
        return False
    if self._git_available is None:
        self._git_available = shutil.which("git") is not None
    ...
```

### 4.3 Shadow Repo 管理

每个项目目录对应一个独立的 shadow Git repo, 存储在 `.hermes-checkpoints/` 下:

| 函数 | 行数 | 职责 |
|------|------|------|
| `_store_path()` | 204 | 返回 checkpoint store 根路径 |
| `_project_hash()` | 198 | 项目目录哈希, 用于隔离 shadow repo |
| `_init_store()` | 387 | 初始化 store + 注册项目 |
| `_init_shadow_repo()` | 546 | 初始化 shadow repo (git init + git add) |
| `_take()` | 838 | 实际执行 git add + commit |
| `list_checkpoints()` | 655 | 列出指定目录的所有 checkpoints |
| `diff()` | 709 | 显示 checkpoint 的 diff |
| `restore()` | 759 | 从 checkpoint 恢复文件 |

---

## 5. ProcessRegistry — 进程生命周期管理

### 5.1 数据模型

`ProcessSession` dataclass 记录每个 spawned 进程的完整状态, `ProcessRegistry` 管理所有活跃和已完成的进程:

源码位置: process_registry.py:88-135

```python
@dataclass
class ProcessSession:
    """State for a single spawned process."""
    session_id: str
    task_id: Optional[str]
    command: Optional[str]
    pid: Optional[int]
    env_pid: Optional[int]
    status: str  # "running", "completed", "failed", "killed"
    exit_code: Optional[int]
    ...

class ProcessRegistry:
    """Manages spawned process sessions."""
    def __init__(self):
        ...
```

### 5.2 两种 Spawn 模式

ProcessRegistry 支持本地 PTY spawn 和 env subprocess spawn:

| 方法 | 行数 | 职责 |
|------|------|------|
| `spawn_local()` | 459-570 | 本地 PTY spawn — 直接 fork+exec, 实时 stdout/stderr 读取 |
| `spawn_via_env()` | 571-652 | Env subprocess spawn — 通过环境配置启动, env poller 监控 |
| `_reader_loop()` | 653-680 | 本地进程 stdout reader 线程 |
| `_env_poller_loop()` | 681-733 | Env 进程状态轮询线程 |
| `_pty_reader_loop()` | 734-765 | PTY 进程输出读取线程 |

### 5.3 进程操作 API

ProcessRegistry 提供完整的进程生命周期操作:

| 方法 | 行数 | 职责 |
|------|------|------|
| `poll()` | 875-906 | 查询进程状态 (非阻塞) |
| `read_log()` | 906-937 | 读取进程输出日志 |
| `wait()` | 937-1014 | 等待进程完成 (支持超时) |
| `kill_process()` | 1014-1074 | 终止进程 (SIGTERM then SIGKILL) |
| `write_stdin()` | 1074-1101 | 向进程 stdin 写入数据 |
| `submit_stdin()` | 1101-1105 | 向 stdin 提交数据 + EOF |
| `close_stdin()` | 1105-1128 | 关闭 stdin pipe |
| `list_sessions()` | 1128-... | 列出所有进程会话 (按 task_id 过滤) |
| `_move_to_finished()` | 765-793 | 将完成进程从 active 移到 finished |

---

## Comparison Tables

### 表 1: 存储引擎对比

| 维度 | SessionDB | Kanban DB | Cron Jobs | CheckpointManager | ProcessRegistry |
|------|-----------|-----------|-----------|-------------------|-----------------|
| 存储引擎 | SQLite (WAL) | SQLite (WAL) | JSON file | Git shadow repo | In-memory + detached |
| 文件位置 | ~/.hermes/state.db | ~/.hermes/kanban.db | ~/.hermes/cron/jobs.json | ~/.hermes-checkpoints/ | N/A (进程态) |
| 并发策略 | app-level jitter retry | 同 SessionDB | atomic write + rename | per-project repo | thread-safe dict |
| 写入原子性 | BEGIN IMMEDIATE | 同 SessionDB | mkstemp + fsync + rename | git add + commit | N/A |
| 查询能力 | FTS5 全文搜索 | SQL query | JSON parse | git log | dict lookup |
| 数据生命周期 | 永久 (可 archive) | 可 archive | 可 remove | max 20 snapshots/目录 | 进程退出后移 finished |

### 表 2: Schema 对比 (SessionDB vs Kanban DB)

| 维度 | SessionDB sessions 表 | Kanban DB tasks 表 |
|------|----------------------|---------------------|
| 主键 | id (TEXT) | id (TEXT) |
| 状态跟踪 | ended_at, end_reason | status, started_at, completed_at |
| 计量统计 | message_count, tool_call_count, input/output_tokens | priority, consecutive_failures |
| 计费信息 | billing_provider, estimated_cost_usd, actual_cost_usd | 无 |
| 父子关系 | parent_session_id (FK) | task_links (多对多) |
| 元数据 | title, model, model_config | workspace_kind, workspace_path, tenant |
| 失败追踪 | 无 | consecutive_failures, last_failure_error |
| 运行追踪 | 无 (flat) | current_run_id (指向 task_runs) |
| 扩展列 | codex_reasoning_items, codex_message_items | skills, workflow_template_id |

### 表 3: 持久化写入策略对比

| 维度 | SessionDB | Kanban DB | Cron Jobs | CheckpointManager |
|------|-----------|-----------|-----------|-------------------|
| 写入锁 | BEGIN IMMEDIATE + Python Lock | 同 SessionDB | mkstemp (独占 tmp 文件) | git lock (per-repo) |
| 重试策略 | jitter: 20-150ms, 15 retries | 同 SessionDB | 无重试 (atomic replace) | 无重试 (silent failure) |
| WAL checkpoint | 每 50 次写入 PASSIVE | 同 SessionDB | N/A | N/A |
| 数据修复 | schema migration (additive) | additive ALTER TABLE | strict=False auto-repair | git fsck (implicit) |
| 备份/恢复 | session resume | task archive | jobs.json backup | git checkout |

### 表 4: FTS5 双表策略对比

| 维度 | messages_fts (unicode61) | messages_fts_trigram (trigram) |
|------|--------------------------|-------------------------------|
| Tokenizer | unicode61 (默认) | trigram |
| 适用语言 | 英文, 空格分隔语言 | CJK, Thai, 任何无空格语言 |
| 索引内容 | content + tool_name + tool_calls | 同上 |
| 同步机制 | INSERT/DELETE/UPDATE 触发器 | 同上 (独立触发器) |
| 查询语法 | 标准 FTS5 (短语, AND, OR) | 子串匹配 (无需特殊语法) |
| 索引大小 | 较小 | 较大 (重叠 token) |
| 性能 | 更快 (less token overlap) | 较慢 (更多 token) |

### 表 5: 进程 Spawn 模式对比

| 维度 | spawn_local (PTY) | spawn_via_env |
|------|-------------------|---------------|
| 进程类型 | 本地命令 | 环境配置的 subprocess |
| 输出读取 | _pty_reader_loop (实时) | _env_poller_loop (轮询) |
| 输入写入 | write_stdin / PTY | submit_stdin / pipe |
| 监控方式 | PID 检查 + PTY read | env 状态轮询 |
| 适用场景 | 工具执行 (shell commands) | 长运行服务 (gateway, worker) |
| 进程分离 | _is_host_pid_alive | _refresh_detached_session |

---

## Summary Table

| 组件名 | 行数 | 职责 |
|--------|------|------|
| `hermes_state.py` SessionDB | 2,669 | SQLite+WAL 会话/消息存储, FTS5 双表全文搜索, schema v11 迁移, jitter retry |
| `hermes_cli/kanban_db.py` Task | 4,087 | 任务数据模型, circuit breaker, 从_row 解析, skills JSON |
| `hermes_cli/kanban_db.py` 表定义 | 同上 | 6 表: tasks, task_links, task_comments, task_events, task_runs, notify_subs |
| `cron/jobs.py` load/save | 1,050 | JSON atomic write, auto-repair, schedule 解析, next_run 计算 |
| `cron/jobs.py` lifecycle | 同上 | create/get/list/update/pause/resume/trigger/remove/mark/advance |
| `tools/checkpoint_manager.py` | 1,638 | Git shadow repo 快照, turn dedup, per-project repo, list/diff/restore |
| `tools/process_registry.py` ProcessSession | 1,434 | 进程状态 dataclass |
| `tools/process_registry.py` ProcessRegistry | 同上 | spawn_local/spawn_via_env, poll/wait/kill, reader/poller loops |

---

[← 07-前端交互界面](/zh-CN/chapters/07-frontend-interfaces) | [→ 09-上下文与记忆引擎](/zh-CN/chapters/09-context-memory)