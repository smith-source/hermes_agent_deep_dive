
# 11 — 技能与自进化系统: 25 内置 + 15 可选技能 + Curator 自审查闭环

## 源码文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `tools/skills_hub.py` | 3229 | SkillSource ABC + 6 实现适配器 + hub 状态管理 |
| `tools/skills_tool.py` | 1533 | 技能查看/加载/列表/执行 — agent 工具层 |
| `tools/skills_guard.py` | 932 | 安全扫描器 — 6 类威胁 + 信任分级安装策略 |
| `tools/skill_manager_tool.py` | 929 | 技能 CRUD — create/edit/patch/delete/write_file/remove_file |
| `tools/skills_sync.py` | 431 | 清单式同步 — bundled 技能 → 用户目录 |
| `agent/skill_commands.py` | 501 | /skills 斜杠命令处理 |
| `agent/skill_utils.py` | 473 | 报名空间解析/平台匹配/禁用逻辑 |
| `agent/curator.py` | 1674 | Curator 自审查 — 状态机 + LLM 评审 + 合并/裁剪 |
| `skills/` (目录) | 25 个子目录 | 25 内置技能 (每个含 SKILL.md) |
| `optional-skills/` (目录) | 15 个子目录 | 15 可选技能 (未激活) |

> **一句话总结**: Hermes 通过 SkillSource ABC 适配 6 种来源的技能注册表, 经 SkillsGuard 安全扫描后才安装, Curator 以状态机 + LLM 评审闭环驱动技能的自我合并/裁剪/归档进化。

## 架构总览

```
┌───────────────────────────────────────────────────────────────┐
│                    技能发现与获取层                             │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────┐ │
│  │GitHub    │  │WellKnown  │  │Url       │  │HermesIndex   │ │
│  │Source    │  │Source     │  │Source    │  │Source        │ │
│  │(5 taps) │  │(.well-    │  │(direct   │  │(hermes.json) │ │
│  │          │  │ known/)   │  │ HTTP)    │  │              │ │
│  └────┬─────┘  └────┬──────┘  └────┬─────┘  └────┬─────────┘ │
│       │             │              │              │            │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐  │            │
│  │ClawHub  │  │Claude     │  │Optional      │  │            │
│  │Source   │  │Marketplace│  │SkillSource   │  │            │
│  │(claw.ai │  │Source     │  │(repo opt-    │  │            │
│  │ API)    │  │(3 repos)  │  │ ional-skills)│  │            │
│  └────┬─────┘  └────┬──────┘  └────┬─────────┘  │            │
│       │             │              │              │            │
│       ▼             ▼              ▼              ▼            │
│  ┌────────────────────────────────────────────────────────────┐│
│  │        create_source_router() → parallel_search_sources() ││
│  └────┬──────────────────────────────────────────────────────┘│
└───────┼───────────────────────────────────────────────────────┘
        │ SkillBundle
        ▼
┌───────────────────────────────────────────────────────────────┐
│                    安全扫描与安装层                             │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐               │
│  │ quarantine│  │skills_   │  │HubLockFile   │               │
│  │ (临时存放) │  │guard.py  │  │+ TapsManager │               │
│  │          │  │(6类威胁)  │  │+ audit.log   │               │
│  └────┬─────┘  └────┬──────┘  └────┬─────────┘               │
│       │             │              │                           │
│       ▼             ▼              ▼                           │
│  ┌────────────────────────────────────────────────────────────┐│
│  │  should_allow_install() → trust_level × verdict → allow/  ││
│  │  block/ask                                                ││
│  └────┬──────────────────────────────────────────────────────┘│
│       │ install_from_quarantine() → ~/.hermes/skills/          │
└───────┼───────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                    技能使用与进化层                             │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐               │
│  │skills_   │  │skill_     │  │curator.py    │               │
│  │tool.py   │  │manager_   │  │(状态机:      │               │
│  │(view/list│  │tool.py    │  │ fresh→stale  │               │
│  │ /execute)│  │(CRUD)     │  │ →archived)   │               │
│  └────┬─────┘  └────┬──────┘  └────┬─────────┘               │
│       │             │              │                           │
│       ▼             ▼              ▼                           │
│  ┌────────────────────────────────────────────────────────────┐│
│  │        skill_view() → SKILL.md → 系统提示注入              ││
│  │        skill_manage() → create/patch → Curator 评审       ││
│  └────┬──────────────────────────────────────────────────────┘│
└───────┼───────────────────────────────────────────────────────┘
```

## TL;DR

Hermes 技能系统通过 SkillSource ABC 定义统一的技能发现接口, 由 6 个适配器实现 (GitHubSource 从 5 个官方 tap 搜索, WellKnownSkillSource 通过 /.well-known/skills/ 协议发现, UrlSource 直接下载 HTTP SKILL.md, ClawHubSource/ClaudeMarketplaceSource/LobeHubSource 查询第三方市场, OptionalSkillSource 从仓库 optional-skills/ 目录加载). 所有外部技能下载后进入 quarantine 目录, 经 SkillsGuard 扫描 6 类威胁模式 (exfiltration/injection/destructive/persistence/network/obfuscation) 后按信任分级策略决定是否允许安装. SkillsTool 负责技能的查看/加载/执行, SkillsManagerTool 提供 7 种 CRUD 操作, skills_sync.py 使用清单式同步将 bundled 技能复制到用户目录. Curator 以状态机 (fresh → stale → archived) 和 LLM 评审闭环驱动技能的自我合并/裁剪/归档进化.

---

## SkillSource ABC 与 6 个适配器

### SkillSource ABC

```python
源码位置: tools/skills_hub.py:252-277

class SkillSource(ABC):
    """Abstract base for all skill registry adapters."""

    @abstractmethod
    def search(self, query: str, limit: int = 10) -> List[SkillMeta]:
        """Search for skills matching a query string."""
        ...

    @abstractmethod
    def fetch(self, identifier: str) -> Optional[SkillBundle]:
        """Download a skill bundle by identifier."""
        ...

    @abstractmethod
    def inspect(self, identifier: str) -> Optional[SkillMeta]:
        """Fetch metadata for a skill without downloading all files."""
        ...

    @abstractmethod
    def source_id(self) -> str:
        """Unique identifier for this source (e.g. 'github', 'clawhub')."""
        ...

    def trust_level_for(self, identifier: str) -> str:
        """Determine trust level for a skill from this source."""
        return "community"
```

### 数据模型: SkillMeta 与 SkillBundle

```python
源码位置: tools/skills_hub.py:63-85

@dataclass
class SkillMeta:
    """Minimal metadata returned by search results."""
    name: str
    description: str
    source: str           # "official", "github", "clawhub", "claude-marketplace", "lobehub"
    identifier: str       # source-specific ID (e.g. "openai/skills/skill-creator")
    trust_level: str      # "builtin" | "trusted" | "community"
    repo: Optional[str] = None
    path: Optional[str] = None
    tags: List[str] = field(default_factory=list)
    extra: Dict[str, Any] = field(default_factory=dict)

@dataclass
class SkillBundle:
    """A downloaded skill ready for quarantine/scanning/installation."""
    name: str
    files: Dict[str, Union[str, bytes]]   # relative_path -> file content
    source: str
    identifier: str
    trust_level: str
    metadata: Dict[str, Any] = field(default_factory=dict)
```

### GitHubSource (5 个官方 tap)

```python
源码位置: tools/skills_hub.py:284-300

class GitHubSource(SkillSource):
    """Fetch skills from GitHub repos via the Contents API."""

    DEFAULT_TAPS = [
        {"repo": "openai/skills", "path": "skills/"},
        {"repo": "anthropics/skills", "path": "skills/"},
        {"repo": "VoltAgent/awesome-agent-skills", "path": "skills/"},
        {"repo": "garrytan/gstack", "path": ""},
        {"repo": "MiniMax-AI/cli", "path": "skill/"},
    ]

    def __init__(self, auth: GitHubAuth, extra_taps=None):
        self.auth = auth
        self.taps = list(self.DEFAULT_TAPS)
        if extra_taps:
            self.taps.extend(extra_taps)

    def trust_level_for(self, identifier: str) -> str:
        """openai/skills and anthropics/skills → 'trusted'; everything else → 'community'."""
        repo = identifier.split("/", 2)[0:2]
        if f"{repo[0]}/{repo[1]}" in TRUSTED_REPOS:
            return "trusted"
        return "community"
```

### WellKnownSkillSource

```python
源码位置: tools/skills_hub.py:708-835

class WellKnownSkillSource(SkillSource):
    """Fetch skills from /.well-known/skills/ endpoints.

    Uses /.well-known/skills/index.json as the registry — mirrors
    the W3C well-known convention for service discovery."""

    def source_id(self) -> str:
        return "well-known"

    def trust_level_for(self, identifier: str) -> str:
        return "community"
```

### UrlSource (直接 HTTP SKILL.md)

```python
源码位置: tools/skills_hub.py:938-977

class UrlSource(SkillSource):
    """Fetch a single-file SKILL.md skill directly from an HTTP(S) URL.
    The identifier IS the URL. Only single-file skills are supported.
    Trust level is always 'community' and the same security scan runs."""

    def source_id(self) -> str:
        return "url"

    def trust_level_for(self, identifier: str) -> str:
        return "community"

    # Search is meaningless for a direct URL — skip (return empty).
    def search(self, query: str, limit: int = 10) -> List[SkillMeta]:
        return []

    def _matches(self, identifier: str) -> bool:
        """Return True iff this source should handle identifier.
        We claim bare HTTP(S) URLs that end in .md."""
        if not isinstance(identifier, str):
            return False
        ident = identifier.strip()
        if not ident.lower().startswith(("http://", "https://")):
            return False
        if "/.well-known/skills/" in ident:
            return False  # Don't steal well-known URLs.
        return ident.lower().endswith(".md")
```

### SkillsShSource (skills.sh 市场注册表)

SkillsShSource 通过 skills.sh 网站 API 搜索技能, 并从底层 GitHub 仓库获取技能内容。它充当一个聚合层: skills.sh 提供搜索索引和元数据, 实际文件下载委托给 GitHubSource:

```python
源码位置: tools/skills_hub.py:1108-1580

class SkillsShSource(SkillSource):
    """Discover skills via skills.sh and fetch content from the underlying GitHub repo."""

    BASE_URL = "https://skills.sh"
    SEARCH_URL = f"{BASE_URL}/api/search"

    def __init__(self, auth: GitHubAuth):
        self.auth = auth
        self.github = GitHubSource(auth=auth)

    def source_id(self) -> str:
        return "skills-sh"

    def trust_level_for(self, identifier: str) -> str:
        return self.github.trust_level_for(self._normalize_identifier(identifier))

    def search(self, query: str, limit: int = 10) -> List[SkillMeta]:
        """Search skills.sh API; fall back to HTML scraping on API failure."""
        # API search path
        resp = httpx.get(self.SEARCH_URL, params={"q": query, "limit": limit}, timeout=20)
        ...
        # HTML scraping fallback: parse skills.sh page for skill links
        items = self._featured_skills(limit) if not query.strip() else ...

    def fetch(self, identifier: str) -> Optional[SkillBundle]:
        """Fetch via GitHubSource after resolving identifier to canonical form."""
        canonical = self._normalize_identifier(identifier)
        detail = self._fetch_detail_page(canonical)
        for candidate in self._candidate_identifiers(canonical):
            bundle = self.github.fetch(candidate)
            if bundle:
                return bundle
        return None
```

搜索策略优先使用 skills.sh JSON API (`/api/search?q=`), API 不可用时降级为 HTML 页面解析 (正则提取 skill 链接)。获取内容时, 先解析详情页获取底层 GitHub 仓库标识符, 然后委托 GitHubSource.fetch() 下载实际文件。缓存机制与 GitHubSource 共享 `_read_index_cache` / `_write_index_cache`, 搜索结果缓存 1 小时。

### OptionalSkillSource (仓库 optional-skills/)

```python
源码位置: tools/skills_hub.py:2324-2349

class OptionalSkillSource(SkillSource):
    """Official optional skills shipped with the repo (not activated by default)."""

    def __init__(self):
        self._optional_dir = Path(__file__).parent.parent / "optional-skills"

    def source_id(self) -> str:
        return "optional"

    def trust_level_for(self, identifier: str) -> str:
        return "builtin"  # Shipped with repo, same trust level as built-in skills
```

### 源路由器与并行搜索

```python
源码位置: tools/skills_hub.py:3092-3118

def create_source_router(auth: Optional[GitHubAuth] = None) -> List[SkillSource]:
    """Build the ordered list of source adapters used by search/install."""
    sources = []
    if auth:
        sources.append(GitHubSource(auth))
    sources.append(WellKnownSkillSource())
    sources.append(UrlSource())
    sources.append(ClawHubSource())
    sources.append(ClaudeMarketplaceSource(auth))
    sources.append(LobeHubSource())
    sources.append(OptionalSkillSource())
    sources.append(HermesIndexSource(auth))
    return sources
```

```python
源码位置: tools/skills_hub.py:3129-3209

def parallel_search_sources(query, sources, ...):
    """Search all sources in parallel via ThreadPoolExecutor."""
    with ThreadPoolExecutor(max_workers=...) as pool:
        futures = {pool.submit(_search_one_source, src, query, limit): src for src in sources}
        results = []
        for future in as_completed(futures, timeout=search_timeout):
            results.extend(future.result())
    return sorted(results, key=_sort_key)
```

### 比较: SkillSource 适配器

| 适配器 | source_id | 信任等级 | 搜索能力 | 获取方式 | 数据来源 |
|--------|-----------|----------|----------|----------|----------|
| GitHubSource | "github" | trusted (openai/anthropics) / community | 完整 (Contents API + tree) | 下载文件到 SkillBundle | 5 DEFAULT_TAPS + user taps |
| WellKnownSkillSource | "well-known" | community | 通过 /.well-known/skills/index.json | HTTP GET SKILL.md | W3C well-known 协议 |
| UrlSource | "url" | community | 无 (直接 URL) | HTTP GET SKILL.md | 用户提供的 URL |
| SkillsShSource | "skills-sh" | trusted (openai/anthropics) / community (委托 GitHubSource) | skills.sh API + HTML fallback | 委托 GitHubSource.fetch | skills.sh 注册表 + GitHub 仓库 |
| ClawHubSource | "clawhub" | community | claw.ai API | HTTP GET | Claw Hub 市场 |
| ClaudeMarketplaceSource | "claude-marketplace" | trusted (openai/anthropics) / community | GitHub marketplace index | HTTP GET | 3 个 KNOWN_MARKETPLACES |
| LobeHubSource | "lobehub" | community | lobehub API | HTTP GET | LobeHub 市场 |
| OptionalSkillSource | "optional" | builtin | 本地目录扫描 | 文件系统复制 | optional-skills/ 目录 |
| HermesIndexSource | "hermes-index" | trusted/community | hermes.json 索引 | GitHub API | 官方索引 |

---

## Hub 状态管理

### 目录结构与文件

```python
源码位置: tools/skills_hub.py:46-56

HERMES_HOME = get_hermes_home()
SKILLS_DIR = HERMES_HOME / "skills"
HUB_DIR = SKILLS_DIR / ".hub"
LOCK_FILE = HUB_DIR / "lock.json"        # 安装来源追踪
QUARANTINE_DIR = HUB_DIR / "quarantine"  # 下载后临时存放
AUDIT_LOG = HUB_DIR / "audit.log"        # 安装/卸载审计日志
TAPS_FILE = HUB_DIR / "taps.json"        # 用户自定义 tap 列表
INDEX_CACHE_DIR = HUB_DIR / "index-cache" # 远程索引缓存 (1h TTL)
```

### Quarantine → Scan → Install 流程

```python
源码位置: tools/skills_hub.py:2693-2759

def quarantine_bundle(bundle: SkillBundle) -> Path:
    """Write a skill bundle to the quarantine directory for scanning."""
    skill_name = _validate_skill_name(bundle.name)  # 防止路径遍历
    validated_files = []
    for rel_path, file_content in bundle.files.items():
        safe_rel_path = _validate_bundle_rel_path(rel_path)  # 防止 .. 和绝对路径
        validated_files.append((safe_rel_path, file_content))
    dest = QUARANTINE_DIR / skill_name
    if dest.exists():
        shutil.rmtree(dest)
    dest.mkdir(parents=True)
    for rel_path, file_content in validated_files:
        file_dest = dest.joinpath(*rel_path.split("/"))
        ...
    return dest

def install_from_quarantine(quarantine_path, skill_name, category, bundle, scan_result) -> Path:
    """Move a scanned skill from quarantine into the skills directory."""
    safe_skill_name = _validate_skill_name(skill_name)
    safe_category = _validate_category_name(category) if category else ""
    quarantine_resolved = quarantine_path.resolve()
    quarantine_root = QUARANTINE_DIR.resolve()
    if not quarantine_resolved.is_relative_to(quarantine_root):
        raise ValueError(f"Unsafe quarantine path: {quarantine_path}")  # 防路径遍历
    install_dir = SKILLS_DIR / safe_category / safe_skill_name
    shutil.move(str(quarantine_path), str(install_dir))
```

### 路径安全验证

```python
源码位置: tools/skills_hub.py:88-110

def _normalize_bundle_path(path_value, *, field_name, allow_nested) -> str:
    """Normalize and validate bundle-controlled paths before touching disk."""
    normalized = raw.replace("\\", "/")
    path = PurePosixPath(normalized)
    parts = [part for part in path.parts if part not in ("", ".")]
    if normalized.startswith("/") or path.is_absolute():
        raise ValueError(f"Unsafe {field_name}: {path_value}")
    if not parts or any(part == ".." for part in parts):
        raise ValueError(f"Unsafe {field_name}: {path_value}")
    if re.fullmatch(r"[A-Za-z]:", parts[0]):
        raise ValueError(f"Unsafe {field_name}: {path_value}")  # Windows drive letter
    if not allow_nested and len(parts) != 1:
        raise ValueError(f"Unsafe {field_name}: {path_value}")
    return "/".join(parts)
```

### HubLockFile 与审计日志

```python
源码位置: tools/skills_hub.py:2549-2603

class HubLockFile:
    """Track provenance of installed hub skills."""
    def __init__(self, path=LOCK_FILE):
    def record_install(self, name, source, identifier, trust_level, content_hash, installed_at, category=""):
    def record_uninstall(self, name):
    def get_installed(self, name):
    def list_installed(self):

class TapsManager:
    """Manage user-added tap repositories."""
    def add(self, repo, path="skills/"):
    def remove(self, repo):
    def list_taps(self):
```

```python
源码位置: tools/skills_hub.py:2660-2680

def append_audit_log(action, skill_name, source, ...):
    """Append an entry to the hub audit log (~/.hermes/skills/.hub/audit.log)."""
```

---

## SkillsGuard 安全扫描 (tools/skills_guard.py)

### 信任分级安装策略

```python
源码位置: tools/skills_guard.py:39-53

TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #                  safe      caution    dangerous
    "builtin":       ("allow",  "allow",   "allow"),
    "trusted":       ("allow",  "allow",   "block"),
    "community":     ("allow",  "block",   "block"),
    "agent-created": ("allow",  "allow",   "ask"),
}
```

### 6 类威胁模式

```python
源码位置: tools/skills_guard.py:86-300+

THREAT_PATTERNS = [
    # ── Exfiltration: shell commands leaking secrets ──
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)',
     "env_exfil_curl", "critical", "exfiltration", ...),
    (r'wget\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)',
     "env_exfil_wget", "critical", "exfiltration", ...),

    # ── Exfiltration: reading credential stores ──
    (r'\$HOME/\.ssh|\~/\.ssh', "ssh_dir_access", "high", "exfiltration", ...),
    (r'\$HOME/\.hermes/\.env|\~/\.hermes/\.env',
     "hermes_env_access", "critical", "exfiltration", ...),

    # ── Exfiltration: programmatic env access ──
    (r'os\.environ\b(?!\s*\.get\s*\(\s*["\']PATH)',
     "python_os_environ", "high", "exfiltration", ...),
    (r'process\.env\[', "node_process_env", "high", "exfiltration", ...),

    # ── Prompt injection ──
    (r'ignore\s+(?:\w+\s+)*(previous|all|above|prior)\s+instructions',
     "prompt_injection_ignore", "critical", "injection", ...),
    (r'do\s+not\s+(?:\w+\s+)*tell\s+(?:\w+\s+)*the\s+user',
     "deception_hide", "critical", "injection", ...),
    (r'output\s+(?:\w+\s+)*(system|initial)\s+prompt',
     "leak_system_prompt", "high", "injection", ...),

    # ── Destructive operations ──
    (r'rm\s+-rf\s+/', "destructive_root_rm", "critical", "destructive", ...),

    # ── Persistence ──
    (r'crontab\s+', "cron_persistence", "high", "persistence", ...),
    (r'\.bashrc\b', "shell_rc_persistence", "high", "persistence", ...),

    # ── Network ──
    (r'reverse.*shell|bind.*shell', "reverse_shell", "critical", "network", ...),

    # ── Obfuscation ──
    (r'base64\s*\(-?\s*b', "base64_decode_obfuscation", "high", "obfuscation", ...),
]
```

### 扫描流程与决策

```python
源码位置: tools/skills_guard.py:599-646

def scan_skill(skill_path: Path, source: str = "community") -> ScanResult:
    """Scan a skill directory for threat patterns.
    Returns a ScanResult with verdict: safe/caution/dangerous."""
    trust = _resolve_trust_level(source)
    findings = _check_structure(skill_dir) + scan_file(...)
    verdict = _determine_verdict(findings)
    summary = _build_summary(name, source, trust, verdict, findings)
    return ScanResult(skill_name=name, source=source, trust_level=trust,
                      verdict=verdict, findings=findings, summary=summary)

def should_allow_install(result: ScanResult, force: bool = False) -> Tuple[bool, str]:
    """Determine if a skill should be allowed to install based on trust × verdict."""
    policy = INSTALL_POLICY.get(result.trust_level, INSTALL_POLICY["community"])
    action = policy[VERDICT_INDEX.get(result.verdict, 2)]
    if action == "allow":
        return True, ""
    if action == "ask":
        return True, f"Review recommended: {result.summary}"  # agent-created: soft block
    if force and action in ("block", "ask"):
        return True, f"Force-installed despite {result.verdict} verdict"
    return False, f"Blocked: {result.summary}"
```

---

## SkillsTool (tools/skills_tool.py)

### 技能查看与加载

```python
源码位置: tools/skills_tool.py:849-909

def skill_view(name, file_path=None, task_id=None, preprocess=True) -> str:
    """View the content of a skill or a specific file within a skill directory.
    Qualified names like 'plugin:skill' resolve to plugin-provided skills."""
    # Qualified name dispatch (plugin skills)
    if ":" in name:
        namespace, bare = parse_qualified_name(name)
        pm = get_plugin_manager()
        plugin_skill_md = pm.find_plugin_skill(name)
        if plugin_skill_md is not None:
            return _serve_plugin_skill(name, plugin_skill_md, preprocess)
    # Local skill lookup
    skills = _find_all_skills()
    matched = [s for s in skills if s["name"] == name]
    ...
```

### 技能列表

```python
源码位置: tools/skills_tool.py:674-745

def skills_list(category=None, task_id=None) -> str:
    """Return a formatted list of available skills, optionally filtered by category."""
    skills = _find_all_skills(skip_disabled=True)
    if category:
        skills = [s for s in skills if s.get("category") == category]
    skills = _sort_skills(skills)
    ...
```

### 技能准备就绪检查

```python
源码位置: tools/skills_tool.py:126-168

class SkillReadinessStatus(str, Enum):
    READY = "ready"          # 所有前置条件满足
    MISSING_ENV = "missing_env"  # 缺少必需环境变量
    MISSING_TOOL = "missing_tool"  # 缺少必需工具
    MISSING_PLATFORM = "missing_platform"  # 平台不匹配

def check_skills_requirements() -> bool:
    """Check if all loaded skills have their prerequisites met."""
```

### 比较: SkillsTool 接口

| 操作 | SkillsTool (skill_view) | SkillsManagerTool (skill_manage) | hermes_cli skills |
|------|------------------------|----------------------------------|-------------------|
| 查看技能内容 | skill_view(name) | N/A | hermes skills view |
| 列出技能 | skills_list(category) | N/A | hermes skills list |
| 创建技能 | N/A | skill_manage(action="create") | hermes skills create |
| 编辑技能 | N/A | skill_manage(action="edit") | hermes skills edit |
| 补丁修改 | N/A | skill_manage(action="patch") | hermes skills patch |
| 删除技能 | N/A | skill_manage(action="delete") | hermes skills remove |
| 写入文件 | N/A | skill_manage(action="write_file") | N/A |
| 删除文件 | N/A | skill_manage(action="remove_file") | N/A |

---

## SkillsManagerTool (tools/skill_manager_tool.py)

### 7 种 CRUD 操作

```python
源码位置: tools/skill_manager_tool.py:711-788

def skill_manage(action, name, content=None, category=None, file_path=None,
                 file_content=None, old_string=None, new_string=None,
                 replace_all=False, absorbed_into=None) -> str:
    """Manage user-created skills. Dispatches to the appropriate action handler."""
    if action == "create":
        result = _create_skill(name, content, category)
    elif action == "edit":
        result = _edit_skill(name, content)
    elif action == "patch":
        result = _patch_skill(name, old_string, new_string, file_path, replace_all)
    elif action == "delete":
        result = _delete_skill(name, absorbed_into=absorbed_into)
    elif action == "write_file":
        result = _write_file(name, file_path, file_content)
    elif action == "remove_file":
        result = _remove_file(name, file_path)
    else:
        result = {"success": False, "error": f"Unknown action '{action}'"}
```

### 安全扫描 (agent-created 技能)

```python
源码位置: tools/skill_manager_tool.py:59-77

def _guard_agent_created_enabled() -> bool:
    """Check if the skills.guard_agent_created config is enabled (off by default)."""
    ...

def _security_scan_skill(skill_dir: Path) -> Optional[str]:
    """Run SkillsGuard scan on an agent-created skill directory.
    Returns an error message if the scan blocks installation, or None if allowed."""
    from tools.skills_guard import scan_skill, should_allow_install
    result = scan_skill(skill_dir, source="agent-created")
    allowed, reason = should_allow_install(result)
    if not allowed:
        return f"Security scan blocked skill creation: {reason}"
    return None
```

### Curator 语义遥测

```python
源码位置: tools/skill_manager_tool.py:769-786

# Curator telemetry: bump patch_count on edit/patch/write_file
try:
    from tools.skill_usage import bump_patch, forget, mark_agent_created
    from tools.skill_provenance import is_background_review
    if action == "create":
        if is_background_review():
            mark_agent_created(name)
    elif action in ("patch", "edit", "write_file", "remove_file"):
        bump_patch(name)
    elif action == "delete":
        forget(name)
except Exception:
    pass
```

### 比较: SkillsManagerTool 操作

| 操作 | 输入要求 | 安全检查 | Curator 语义 |
|------|----------|----------|-------------|
| create | name + content (SKILL.md 全文) | _security_scan_skill (agent-created 时) | mark_agent_created (background review 时) |
| edit | name + content (SKILL.md 全文) | _security_scan_skill | bump_patch |
| patch | name + old_string + new_string | _security_scan_skill | bump_patch |
| delete | name + absorbed_into (可选) | N/A | forget |
| write_file | name + file_path + file_content | 路径验证 | bump_patch |
| remove_file | name + file_path | 路径验证 | bump_patch |

---

## Skills Sync (tools/skills_sync.py)

### 清单式同步逻辑

```python
源码位置: tools/skills_sync.py:1-22

"""
Skills Sync -- Manifest-based seeding and updating of bundled skills.
Manifest format (v2): each line is 'skill_name:origin_hash'
Update logic:
  - NEW skills (not in manifest): copied, origin hash recorded.
  - EXISTING skills (user copy matches origin hash): safe to update from bundled.
  - EXISTING skills (user copy differs from origin hash): user customized it → SKIP.
  - DELETED by user (in manifest, absent from user dir): respected, not re-added.
  - REMOVED from bundled (in manifest, gone from repo): cleaned from manifest.
"""
```

---

## Curator 自审查闭环 (agent/curator.py)

### 状态机: fresh → stale → archived

```python
源码位置: agent/curator.py:131-177

def _load_config() -> Dict[str, Any]:
    """Load curator config from config.yaml (curator section)."""

def is_enabled() -> bool:
    """Check if curator is enabled (default: True)."""

def get_interval_hours() -> int:
    """Get review interval in hours (default: 168 = 7 days)."""

def get_min_idle_hours() -> float:
    """Get minimum idle hours before running (default: 2.0)."""

def get_stale_after_days() -> int:
    """Get days before marking agent-created skills as stale (default: 30)."""

def get_archive_after_days() -> int:
    """Get days before archiving stale skills (default: 90)."""

def apply_automatic_transitions(now=None) -> Dict[str, int]:
    """Apply automatic state transitions:
    - fresh → stale (after stale_after_days without use)
    - stale → archived (after archive_after_days)
    - stale → fresh (if skill was recently used)"""
```

### LLM 评审流程

```python
源码位置: agent/curator.py:1278-1360

def run_curator_review(on_summary=None, synchronous=False, dry_run=False) -> Dict[str, Any]:
    """Execute a single curator review pass.
    Steps:
      1. Apply automatic state transitions (pure, no LLM).
      2. If there are agent-created skills, spawn a forked AIAgent that runs
         the LLM review prompt against the current candidate list.
      3. Update .curator_state with last_run_at and a one-line summary.
      4. Invoke on_summary with a user-visible description."""
    # Pre-mutation snapshot
    snap = curator_backup.snapshot_skills(reason="pre-curator-run")
    counts = apply_automatic_transitions(now=start)

    # Persist state before the LLM pass
    state = load_state()
    state["last_run_at"] = start.isoformat()
    state["run_count"] = int(state.get("run_count", 0)) + 1
    save_state(state)

    def _llm_pass():
        """Spawn AIAgent with curator review prompt."""
        ...
```

### 合并/裁剪分类

```python
源码位置: agent/curator.py:491-613

def _classify_removed_skills(removed_names, all_skills, ...):
    """Classify removed skills as consolidated or pruned.
    - consolidated: merged into another skill (absorbed_into declared)
    - pruned: genuinely deleted without replacement"""
```

---

## 技能定义格式 (SKILL.md)

### 内置技能示例 (dogfood)

```yaml
源码位置: skills/dogfood/SKILL.md:1-9

---
name: dogfood
description: "Exploratory QA of web apps: find bugs, evidence, reports."
version: 1.0.0
metadata:
  hermes:
    tags: [qa, testing, browser, web, dogfood]
    related_skills: []
---
```

技能正文包含:
- Prerequisites (必需工具/环境)
- Inputs (用户提供什么)
- Workflow (5 个阶段: Plan → Explore → Collect Evidence → Categorize → Report)
- Tools Reference (browser_navigate, browser_snapshot, browser_click 等)
- Tips (最佳实践)

### 技能目录结构

```
skills/
├── dogfood/SKILL.md               # QA 测试技能
├── email/himalaya/SKILL.md         # Himalaya 邮件客户端技能
├── red-teaming/godmode/SKILL.md    # 红队渗透技能
├── devops/webhook-subscriptions/   # DevOps webhook 技能
├── productivity/airtable/          # Airtable 集成技能
├── ... (25 个子目录, 含内置技能)
optional-skills/
├── autonomous-ai-agents/           # 自主 AI 代理 (未激活)
├── blockchain/                     # 区块链技能 (未激活)
├── security/                       # 安全技能 (未激活)
├── ... (15 个子目录, 含可选技能)
```

---

## 汇总表

| 组件 | 行数 | 职责 |
|------|------|------|
| `tools/skills_hub.py` | 3229 | SkillSource ABC + 8 适配器 + Hub 状态管理 (quarantine/lock/audit/taps) |
| `tools/skills_tool.py` | 1533 | 技能查看/列表/加载/执行 — skill_view + skills_list |
| `tools/skills_guard.py` | 932 | 安全扫描器 — 6 类威胁 + 4 级信任 × 3 级裁决安装策略 |
| `tools/skill_manager_tool.py` | 929 | 技能 CRUD — 7 种操作 + 安全扫描 + Curator 语义 |
| `tools/skills_sync.py` | 431 | 清单式同步 — v2 manifest + 哈希追踪 + 用户修改保护 |
| `agent/skill_commands.py` | 501 | /skills 斜杠命令路由 |
| `agent/skill_utils.py` | 473 | 报名空间解析 + 平台匹配 + 禁用逻辑 |
| `agent/curator.py` | 1674 | 自审查闭环 — 状态机 + LLM 评审 + 合并/裁剪分类 |
| `skills/` (25 子目录) | ~25 技能 | 25 内置技能 (每个含 SKILL.md + 可选 references/) |
| `optional-skills/` (15 子目录) | ~15 技能 | 15 可选技能 (未激活, trust=builtin) |

---

[← 10 — 安全体系](/zh-CN/chapters/10-security) | [→ 12 — 定时任务与调度](/zh-CN/chapters/12-cron-scheduling)