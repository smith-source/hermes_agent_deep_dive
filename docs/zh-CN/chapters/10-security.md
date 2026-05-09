
# 10 — 安全体系: 七层纵深防御与信任模型

## 源码文件

| 文件 | 行数 | 核心职责 |
|------|------|----------|
| `tools/url_safety.py` | 231 | L1 SSRF 防护 — 私有/内部/元数据 IP 拦截 |
| `tools/approval.py` | 1258 | L2 危险命令审批 — 三模式门控 + 硬线封锁 |
| `tools/mcp_tool.py` | 3175 | L3 凭证隔离 — MCP 子进程环境过滤 |
| `tools/code_execution_tool.py` | 1621 | L3 凭证隔离 — 沙盒 API Key 剻除 |
| `tools/delegate_tool.py` | 2562 | L4 子代理安全 — 深度限制 + 工具集裁剪 |
| `Dockerfile` + `docker/entrypoint.sh` | 84+140 | L5 容器安全 — 非root UID 10000 + gosu 权限下放 |
| `agent/redact.py` | 400 | L6 输出脱敏 — RedactingFormatter + 30+ 供应商正则 |
| `tools/osv_check.py` | 155 | L7 供应链 — OSV MAL-* 查询 |
| `tools/tirith_security.py` | 691 | L7 供应链 — Tirith 预执行安全扫描 |
| `tools/website_policy.py` | 282 | 网站 URL 黑名单策略 |
| `agent/tool_guardrails.py` | 455 | 工具调用循环护栏 |
| `SECURITY.md` | 85 | 信任模型与安全策略声明 |

> **一句话总结**: Hermes 通过七层纵深防御 (SSRf > 审批 > 凭证隔离 > 子代理安全 > 容器 > 脱敏 > 供应链) 在单信任域假设下保护操作者免受 LLM 行为侵害。

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    用户交互层 (CLI / Gateway)            │
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ approval │  │ tirith_sec   │  │ tool_guardrails    │  │
│  │ .py      │  │ .py          │  │ .py                │  │
│  │ (L2门控) │  │ (L7扫描)     │  │ (循环护栏)          │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────────┘  │
│       │               │                  │                │
│       ▼               ▼                  ▼                │
│  ┌──────────────────────────────────────────────────────┐│
│  │         check_all_command_guards() 统一入口           ││
│  │   hardline → yolo → tirith → dangerous → smart/approve││
│  └────┬─────────────────────────────────────────────────┘│
└───────┼──────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│                    执行层 (terminal / code / delegate)    │
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ url_saf  │  │ mcp_tool     │  │ delegate_tool      │  │
│  │ ety.py   │  │ (env过滤)    │  │ (深度+工具裁剪)     │  │
│  │ (L1SSRF) │  │ (L3凭证)     │  │ (L4子代理)          │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────────┘  │
│       │               │                  │                │
│       ▼               ▼                  ▼                │
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ code_exe │  │ redact.py    │  │ Dockerfile         │  │
│  │ c_tool   │  │ (L6脱敏)     │  │ (L5容器UID10000)   │  │
│  │ (L3凭证) │  │              │  │                    │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────────┘  │
│       │               │                  │                │
│       ▼               ▼                  ▼                │
│  ┌──────────────────────────────────────────────────────┐│
│  │               osv_check.py (L7 供应链)                ││
│  └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## TL;DR

Hermes 假设单信任域 (一个可信操作者), 通过七层纵深防御保护操作者免受 LLM 行为侵害。L1 SSRF 防护拦截私有网络和云元数据端点 (169.254.169.254 等); L2 审批系统使用硬线封锁 + 三模式门控 (manual/smart/off) + 辅助 LLM 风险评估; L3 凭证隔离对 MCP 子进程和沙盒子进程分别过滤环境变量; L4 子代理安全禁止递归委托并限制深度为 1-3; L5 容器安全使用非 root UID 10000 + gosu 权限下放; L6 输出脱敏覆盖 30+ 供应商 API Key 前缀; L7 供应链安全包含 CI .pth 扫描、OSV MAL-* 查询和 Tirith 预执行扫描。SECURITY.md 明确声明了信任边界和不在范围内的非漏洞类型。

---

## L1 — SSRF 防护 (tools/url_safety.py)

### 核心机制

`is_safe_url()` 对每个 URL 执行 DNS 解析后检查目标 IP, 失败时关闭 (fail-closed):

```python
源码位置: tools/url_safety.py:155-231

def is_safe_url(url: str) -> bool:
    """Return True if the URL target is not a private/internal address.
    Fails closed: DNS errors and unexpected exceptions block the request.
    """
    try:
        parsed = urlparse(url)
        hostname = (parsed.hostname or "").strip().lower().rstrip(".")
        scheme = (parsed.scheme or "").strip().lower()
        if not hostname:
            return False
        # Block known internal hostnames — ALWAYS, even with toggle on
        if hostname in _BLOCKED_HOSTNAMES:
            logger.warning("Blocked request to internal hostname: %s", hostname)
            return False
        allow_all_private = _global_allow_private_urls()
        allow_private_ip = _allows_private_ip_resolution(hostname, scheme)
        try:
            addr_info = socket.getaddrinfo(hostname, None, socket.AF_UNSPEC, socket.SOCK_STREAM)
        except socket.gaierror:
            logger.warning("Blocked request — DNS resolution failed for: %s", hostname)
            return False
        for family, _, _, _, sockaddr in addr_info:
            ip_str = sockaddr[0]
            ip = ipaddress.ip_address(ip_str)
            # Always block cloud metadata IPs and link-local
            if ip in _ALWAYS_BLOCKED_IPS or any(ip in net for net in _ALWAYS_BLOCKED_NETWORKS):
                logger.warning("Blocked request to cloud metadata address: %s -> %s", hostname, ip_str)
                return False
            if not allow_all_private and not allow_private_ip and _is_blocked_ip(ip):
                logger.warning("Blocked request to private/internal address: %s -> %s", hostname, ip_str)
                return False
        return True
    except Exception as exc:
        logger.warning("Blocked request — URL safety check error for %s: %s", url, exc)
        return False
```

### 始终封锁的云元数据端点

```python
源码位置: tools/url_safety.py:39-57

_BLOCKED_HOSTNAMES = frozenset({
    "metadata.google.internal",
    "metadata.goog",
})

_ALWAYS_BLOCKED_IPS = frozenset({
    ipaddress.ip_address("169.254.169.254"),  # AWS/GCP/Azure/DO/Oracle metadata
    ipaddress.ip_address("169.254.170.2"),     # AWS ECS task metadata (task IAM creds)
    ipaddress.ip_address("169.254.169.253"),   # Azure IMDS wire server
    ipaddress.ip_address("fd00:ec2::254"),     # AWS metadata (IPv6)
    ipaddress.ip_address("100.100.100.200"),   # Alibaba Cloud metadata
})

_ALWAYS_BLOCKED_NETWORKS = (
    ipaddress.ip_network("169.254.0.0/16"),    # Entire link-local range
)
```

### CGNAT 网段显式封锁

标准 `ipaddress.is_private` 不覆盖 100.64.0.0/10 (RFC 6598 CGNAT), Hermes 显式添加:

```python
源码位置: tools/url_safety.py:70

_CGNAT_NETWORK = ipaddress.ip_network("100.64.0.0/10")
```

---

## L2 — 危险命令审批 (tools/approval.py)

### 硬线封锁 (Hardline Floor)

硬线封锁是不可绕过的底线, 覆盖 rm -rf /、mkfs、dd 到裸设备、fork bomb、kill -1、shutdown/reboot 等:

```python
源码位置: tools/approval.py:156-190

HARDLINE_PATTERNS = [
    (r'\brm\s+(-[^\s]*\s+)*(/|/\*|/ \*)(\s|$)', "recursive delete of root filesystem"),
    (r'\bmkfs(\.[a-z0-9]+)?\b', "format filesystem (mkfs)"),
    (r'\bdd\b[^\n]*\bof=/dev/(sd|nvme|hd|mmcblk|vd|xvd)[a-z0-9]*', "dd to raw block device"),
    (r':\(\)\s*\{\s*:\s*\|\s*:\s*&\s*\}\s*;\s*:', "fork bomb"),
    (r'\bkill\s+(-[^\s]+\s+)*-1\b', "kill all processes"),
    (_CMDPOS + r'(shutdown|reboot|halt|poweroff)\b', "system shutdown/reboot"),
    _CMDPOS + r'init\s+[06]\b', "init 0/6 (shutdown/reboot)"),
    _CMDPOS + r'systemctl\s+(poweroff|reboot|halt|kexec)\b', "systemctl poweroff/reboot"),
]
```

`_CMDPOS` 正则锚定命令起始位置, 防止 "echo reboot" 等误报:

```python
源码位置: tools/approval.py:147-154

_CMDPOS = (
    r'(?:^|[;&|\n`]|\$\()'         # start position
    r'\s*'                          # optional whitespace
    r'(?:sudo\s+(?:-[^\s]+\s+)*)?'  # optional sudo with flags
    r'(?:env\s+(?:\w+=\S*\s+)*)?'   # optional env with VAR=VAL pairs
    r'(?:(?:exec|nohup|setsid|time)\s+)*'  # optional wrapper commands
    r'\s*'
)
```

### 三模式门控

```python
源码位置: tools/approval.py:716-719

def _get_approval_mode() -> str:
    """Read the approval mode from config. Returns 'manual', 'smart', or 'off'."""
    mode = _get_approval_config().get("mode", "manual")
    return _normalize_approval_mode(mode)
```

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `manual` | 每次危险命令均提示用户 | 默认, 交互式使用 |
| `smart` | 辅助 LLM 自动评估风险; approve/deny/escalate | 高频操作, 减少打扰 |
| `off` | 跳过所有审批提示 (硬线仍然生效) | cron/batch, YOLO |

### Smart Approval (辅助 LLM 风险评估)

```python
源码位置: tools/approval.py:743-787

def _smart_approve(command: str, description: str) -> str:
    """Use the auxiliary LLM to assess risk and decide approval.
    Returns 'approve' if the LLM determines the command is safe,
    'deny' if genuinely dangerous, or 'escalate' if uncertain.
    """
    from agent.auxiliary_client import call_llm
    prompt = f"""You are a security reviewer for an AI coding agent...
    Command: {command}
    Flagged reason: {description}
    Assess the ACTUAL risk of this command...
    Respond with exactly one word: APPROVE, DENY, or ESCALATE"""
    response = call_llm(task="approval", messages=[...], temperature=0, max_tokens=16)
    answer = (response.choices[0].message.content or "").strip().upper()
    if "APPROVE" in answer:
        return "approve"
    elif "DENY" in answer:
        return "deny"
    else:
        return "escalate"
```

### 统一入口: check_all_command_guards()

`check_all_command_guards()` 将 Tirith 和危险命令检测合并为单一审批决策, 防止 gateway force=True 重放绕过其中一个检查:

```python
源码位置: tools/approval.py:920-988

def check_all_command_guards(command, env_type, approval_callback=None) -> dict:
    # Skip containers
    if env_type in ("docker", "singularity", "modal", "daytona", "vercel_sandbox"):
        return {"approved": True, "message": None}
    # Hardline floor first
    is_hardline, hardline_desc = detect_hardline_command(command)
    if is_hardline:
        return _hardline_block_result(hardline_desc)
    # YOLO / mode=off bypass
    if ... or approval_mode == "off":
        return {"approved": True, "message": None}
    # Phase 1: Gather findings from both checks
    tirith_result = check_command_security(command)  # L7
    is_dangerous, pattern_key, description = detect_dangerous_command(command)  # L2
    # Phase 2: Decide — combined approval or smart/auto
    ...
```

### Gateway 阻塞式审批

Gateway 使用 `_ApprovalEntry` + `threading.Event` 实现阻塞式审批, 防止并行子代理的死锁:

```python
源码位置: tools/approval.py:380-387

class _ApprovalEntry:
    """One pending dangerous-command approval inside a gateway session."""
    __slots__ = ("event", "data", "result")
    def __init__(self, data: dict):
        self.event = threading.Event()
        self.data = data          # command, description, pattern_keys, ...
        self.result: Optional[str] = None  # "once"|"session"|"always"|"deny"
```

---

## L3 — 凭证隔离

### MCP 子进程环境过滤 (tools/mcp_tool.py)

MCP stdio 子进程仅接收安全基线变量 + 用户显式声明的 env 块:

```python
源码位置: tools/mcp_tool.py:251-292

_SAFE_ENV_KEYS = frozenset({
    "PATH", "HOME", "USER", "LANG", "LC_ALL", "TERM", "SHELL", "TMPDIR",
})

def _build_safe_env(user_env: Optional[dict]) -> dict:
    """Build a filtered environment dict for stdio subprocesses.
    Only passes through safe baseline variables and XDG_* from the current
    process environment, plus any variables explicitly specified by the user
    in the server config. This prevents accidentally leaking secrets like
    API keys, tokens, or credentials to MCP server subprocesses."""
    env = {}
    for key, value in os.environ.items():
        if key in _SAFE_ENV_KEYS or key.startswith("XDG_"):
            env[key] = value
    if user_env:
        env.update(user_env)
    return env
```

错误消息中的凭证也被脱敏:

```python
源码位置: tools/mcp_tool.py:256-301

_CREDENTIAL_PATTERN = re.compile(
    r"(?:"
    r"ghp_[A-Za-z0-9_]{1,255}"           # GitHub PAT
    r"|sk-[A-Za-z0-9_]{1,255}"           # OpenAI-style key
    r"|Bearer\s+\S+"                      # Bearer token
    r"|token=[^\s&,;\"']{1,255}"
    r"|key=[^\s&,;\"']{1,255}"
    r"|API_KEY=[^\s&,;\"']{1,255}"
    r"|password=[^\s&,;\"']{1,255}"
    r"|secret=[^\s&,;\"']{1,255}"
    r")",
    re.IGNORECASE,
)

def _sanitize_error(text: str) -> str:
    """Strip credential-like patterns from error text before returning to LLM."""
    return _CREDENTIAL_PATTERN.sub("[REDACTED]", text)
```

### 沙盒 API Key 剻除 (tools/code_execution_tool.py)

`execute_code` 子进程使用白名单 + 黑名单双过滤:

```python
源码位置: tools/code_execution_tool.py:1028-1056

_SAFE_ENV_PREFIXES = ("PATH", "HOME", "USER", "LANG", "LC_", "TERM",
                      "TMPDIR", "TMP", "TEMP", "SHELL", "LOGNAME",
                      "XDG_", "PYTHONPATH", "VIRTUAL_ENV", "CONDA",
                      "HERMES_")
_SECRET_SUBSTRINGS = ("KEY", "TOKEN", "SECRET", "PASSWORD", "CREDENTIAL",
                      "PASSWD", "AUTH")
child_env = {}
for k, v in os.environ.items():
    if _is_passthrough(k):
        child_env[k] = v
        continue
    # Block vars with secret-like names
    if any(s in k.upper() for s in _SECRET_SUBSTRINGS):
        continue
    # Allow vars with known safe prefixes
    if any(k.startswith(p) for p in _SAFE_ENV_PREFIXES):
        child_env[k] = v
```

沙盒输出也通过 `redact_sensitive_text()` 脱敏:

```python
源码位置: tools/code_execution_tool.py:1236-1242

from agent.redact import redact_sensitive_text
stdout_text = redact_sensitive_text(stdout_text)
stderr_text = redact_sensitive_text(stderr_text)
```

### 比较: 凭证隔离策略

| 维度 | MCP 子进程 (mcp_tool.py) | 沙盒子进程 (code_execution_tool.py) |
|------|--------------------------|--------------------------------------|
| 过滤策略 | 白名单 (_SAFE_ENV_KEYS + XDG_* + user_env) | 白名单前缀 + 黑名单子串 + skill passthrough |
| SECRET 检测 | 无 (仅白名单) | 有 (_SECRET_SUBSTRINGS 黑名单) |
| 错误脱敏 | _sanitize_error (凭证正则) | redact_sensitive_text (30+ 供应商正则) |
| 用户声明的 env 透传 | 是 (server config env block) | 是 (terminal.env_passthrough + skill env_passthrough) |
| HOME 隔离 | 无 | 是 (get_subprocess_home() 重定向) |

---

## L4 — 子代理安全 (tools/delegate_tool.py)

### 递归委托禁止

```python
源码位置: tools/delegate_tool.py:39-48

DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",  # no recursive delegation
    "clarify",        # no user interaction
    "memory",         # no writes to shared MEMORY.md
    "send_message",   # no cross-platform side effects
    "execute_code",   # children should reason step-by-step
])
```

### 深度限制

```python
源码位置: tools/delegate_tool.py:128-132

MAX_DEPTH = 1  # flat by default: parent (0) -> child (1)
_MIN_SPAWN_DEPTH = 1
_MAX_SPAWN_DEPTH_CAP = 3

def _get_max_spawn_depth() -> int:
    """Read delegation.max_spawn_depth from config, clamped to [1, 3].
    depth 0 = parent agent. max_spawn_depth = N means agents at depths
    0..N-1 can spawn; depth N is the leaf floor."""
    cfg = _load_config()
    val = cfg.get("max_spawn_depth")
    ...
    clamped = max(_MIN_SPAWN_DEPTH, min(_MAX_SPAWN_DEPTH_CAP, ival))
    return clamped
```

### 子代理审批回调

```python
源码位置: tools/delegate_tool.py:68-74

def _subagent_auto_deny(command: str, description: str, **kwargs) -> str:
    """Auto-deny dangerous commands in subagent threads (safe default).
    Returns 'deny' so the subagent sees a refusal it can recover from, and
    never calls input() (which would deadlock the parent TUI)."""
    logger.warning(...)
    return "deny"
```

---

## L5 — 容器安全

### Dockerfile 非 root 用户

```dockerfile
源码位置: Dockerfile:21

RUN useradd -u 10000 -m -d /opt/data hermes
```

### entrypoint.sh 权限下放

```bash
源码位置: docker/entrypoint.sh:12-55

if [ "$(id -u)" = "0" ]; then
    # Remap hermes UID/GID to match host ownership
    if [ -n "$HERMES_UID" ] && [ "$HERMES_UID" != "$(id -u hermes)" ]; then
        usermod -u "$HERMES_UID" hermes
    fi
    # Fix ownership of the data volume
    chown -R hermes:hermes "$HERMES_HOME" 2>/dev/null || \
        echo "Warning: chown failed (rootless container?)"
    echo "Dropping root privileges"
    exec gosu hermes "$0" "$@"
fi
```

---

## L6 — 输出脱敏 (agent/redact.py)

### 30+ 供应商 API Key 前缀

```python
源码位置: agent/redact.py:67-103

_PREFIX_PATTERNS = [
    r"sk-[A-Za-z0-9_-]{10,}",           # OpenAI / OpenRouter / Anthropic (sk-ant-*)
    r"ghp_[A-Za-z0-9]{10,}",            # GitHub PAT (classic)
    r"github_pat_[A-Za-z0-9_]{10,}",    # GitHub PAT (fine-grained)
    r"gho_[A-Za-z0-9]{10,}",            # GitHub OAuth access token
    r"xox[baprs]-[A-Za-z0-9-]{10,}",    # Slack tokens
    r"AIza[A-Za-z0-9_-]{30,}",          # Google API keys
    r"AKIA[A-Z0-9]{16}",                # AWS Access Key ID
    r"sk_live_[A-Za-z0-9]{10,}",        # Stripe secret key (live)
    r"SG\.[A-Za-z0-9_-]{10,}",          # SendGrid API key
    r"hf_[A-Za-z0-9]{10,}",             # HuggingFace token
    r"r8_[A-Za-z0-9]{10,}",             # Replicate API token
    r"gsk_[A-Za-z0-9]{10,}",            # Groq Cloud API key
    ...                                   # 共 30+ 供应商
]
```

### 多层脱敏管道

```python
源码位置: agent/redact.py:308-389

def redact_sensitive_text(text, *, force=False, code_file=False) -> str:
    if not (force or _REDACT_ENABLED):
        return text
    # Known prefixes (sk-, ghp_, etc.)
    text = _PREFIX_RE.sub(lambda m: _mask_token(m.group(1)), text)
    # ENV assignments (skip for code files)
    if not code_file:
        text = _ENV_ASSIGN_RE.sub(_redact_env, text)
        text = _JSON_FIELD_RE.sub(_redact_json, text)
    # Authorization headers
    text = _AUTH_HEADER_RE.sub(...)
    # Telegram bot tokens
    text = _TELEGRAM_RE.sub(_redact_telegram, text)
    # Private key blocks
    text = _PRIVATE_KEY_RE.sub("[REDACTED PRIVATE KEY]", text)
    # Database connection string passwords
    text = _DB_CONNSTR_RE.sub(...)
    # JWT tokens (eyJ...)
    text = _JWT_RE.sub(...)
    # URL userinfo
    text = _redact_url_userinfo(text)
    # URL query params
    text = _redact_url_query_params(text)
    # Discord mentions, E.164 phone numbers
    ...
    return text
```

### mask_secret 分级脱敏

```python
源码位置: agent/redact.py:187-231

def mask_secret(value, *, head=4, tail=4, floor=12, placeholder="***", empty="") -> str:
    """Mask a secret for display, preserving head and tail characters.
    Short tokens (< floor chars) are fully masked. Longer tokens preserve
    the first `head` (default 4) and last `tail` (default 4) characters for debuggability."""
    if not value:
        return empty
    if len(value) < floor:
        return placeholder
    return f"{value[:head]}...{value[-tail:]}"
```

---

## L7 — 供应链安全

### OSV 恶意软件检查 (tools/osv_check.py)

在 npx/uvx 启动 MCP 服务器前查询 OSV API, 仅阻断 MAL-* 确认恶意通告:

```python
源码位置: tools/osv_check.py:26-62

def check_package_for_malware(command: str, args: list) -> Optional[str]:
    """Check if an MCP server package has known malware advisories.
    Inspects the command (npx/uvx) and args to infer the package name.
    Returns an error message string if malware is found, or None if clean/unknown.
    Returns None (allow) on network errors or unrecognized commands."""
    ecosystem = _infer_ecosystem(command)
    if not ecosystem:
        return None  # not npx/uvx — skip
    package, version = _parse_package_from_args(args, ecosystem)
    if not package:
        return None
    try:
        malware = _query_osv(package, ecosystem, version)
    except Exception as exc:
        logger.debug("OSV check failed for %s/%s (allowing): %s", ecosystem, package, exc)
        return None  # Fail-open
    if malware:
        ids = ", ".join(m["id"] for m in malware[:3])
        return f"BLOCKED: Package '{package}' ({ecosystem}) has known malware advisories: {ids}."
    return None
```

```python
源码位置: tools/osv_check.py:131-155

def _query_osv(package, ecosystem, version=None) -> list:
    """Query the OSV API for MAL-* advisories. Returns list of malware vulns."""
    payload = {"package": {"name": package, "ecosystem": ecosystem}}
    data = json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(_OSV_ENDPOINT, data=data, headers={...}, method="POST")
    with urllib.request.urlopen(req, timeout=_TIMEOUT) as resp:
        result = json.loads(resp.read())
    vulns = result.get("vulns", [])
    return [v for v in vulns if v.get("id", "").startswith("MAL-")]
```

### Tirith 预执行扫描 (tools/tirith_security.py)

Tirith 是 Rust 编写的命令安全扫描器, 作为子进程运行, 退出码决定行为 (0=allow, 1=block, 2=warn):

```python
源码位置: tools/tirith_security.py:615-691

def check_command_security(command: str) -> dict:
    """Run tirith security scan on a command.
    Exit code determines action (0=allow, 1=block, 2=warn)."""
    cfg = _load_security_config()
    if not cfg["tirith_enabled"]:
        return {"action": "allow", "findings": [], "summary": ""}
    tirith_path = _resolve_tirith_path(cfg["tirith_path"])
    timeout = cfg["tirith_timeout"]
    fail_open = cfg["tirith_fail_open"]
    result = subprocess.run(
        [tirith_path, "check", "--json", "--non-interactive",
         "--shell", "posix", "--", command],
        capture_output=True, text=True, timeout=timeout,
    )
    exit_code = result.returncode
    if exit_code == 0:  action = "allow"
    elif exit_code == 1:  action = "block"
    elif exit_code == 2:  action = "warn"
    else:
        if fail_open: return {"action": "allow", ...}
        return {"action": "block", ...}
    findings = []
    try:
        data = json.loads(result.stdout)
        findings = data.get("findings", [])[:_MAX_FINDINGS]
        summary = (data.get("summary", "") or "")[:_MAX_SUMMARY_LEN]
    except json.JSONDecodeError:
        pass
    return {"action": action, "findings": findings, "summary": summary}
```

Tirith 支持自动安装 (SHA-256 校验 + cosign provenance 验证):

```python
源码位置: tools/tirith_security.py:282-387

def _install_tirith(*, log_failures=True) -> tuple[str | None, str]:
    """Download and install tirith to $HERMES_HOME/bin/tirith.
    Verifies provenance via cosign and SHA-256 checksum."""
    ...
    # Cosign provenance verification
    if shutil.which("cosign"):
        cosign_result = _verify_cosign(checksums_path, sig_path, cert_path)
        if cosign_result is True:
            cosign_verified = True
        elif cosign_result is False:
            return None, "cosign_verification_failed"
    # SHA-256 checksum verification
    if not _verify_checksum(archive_path, checksums_path, archive_name):
        return None, "checksum_failed"
```

### CI 供应链扫描

SECURITY.md 描述的 CI 防护:
- GitHub Actions pinned to full commit SHAs
- `supply-chain-audit.yml` 阻断 .pth 文件和 base64+exec 模式

---

## 网站 URL 黑名单 (tools/website_policy.py)

### 策略加载与缓存

```python
源码位置: tools/website_policy.py:131-199

def load_website_blocklist(config_path=None) -> Dict[str, Any]:
    """Load and return the parsed website blocklist policy.
    Results are cached for _CACHE_TTL_SECONDS (30s) to avoid re-reading config.yaml."""
    ...
    # Load from config.yaml security.website_blocklist section
    policy = _load_policy_config(config_path)
    # Parse domains + shared_files
    rules = []
    for raw_rule in raw_domains:
        normalized = _normalize_rule(raw_rule)
        rules.append({"pattern": normalized, "source": "config"})
    for shared_file in raw_shared_files:
        for normalized in _iter_blocklist_file_rules(path):
            rules.append({"pattern": normalized, "source": str(path)})
    result = {"enabled": enabled, "rules": rules}
    return result
```

### 访问检查

```python
源码位置: tools/website_policy.py:232-282

def check_website_access(url, config_path=None) -> Optional[Dict[str, str]]:
    """Check whether a URL is allowed by the website blocklist policy.
    Returns None if access is allowed, or a dict with block metadata if blocked.
    Never raises on policy errors — logs a warning and returns None (fail-open)."""
    host = _extract_host_from_urlish(url)
    policy = load_website_blocklist(config_path)
    if not policy.get("enabled"):
        return None
    for rule in policy.get("rules", []):
        pattern = rule.get("pattern", "")
        if _match_host_against_rule(host, pattern):
            return {"url": url, "host": host, "rule": pattern, "source": rule.get("source", "config"),
                    "message": f"Blocked by website policy: '{host}' matched rule '{pattern}'"}
    return None
```

---

## 工具调用循环护栏 (agent/tool_guardrails.py)

### 检测机制

`ToolCallGuardrailController` 跟踪三种循环模式:

```python
源码位置: agent/tool_guardrails.py:19-59

IDEMPOTENT_TOOL_NAMES = frozenset({
    "read_file", "search_files", "web_search", "browser_snapshot", ...
})

MUTATING_TOOL_NAMES = frozenset({
    "terminal", "execute_code", "write_file", "patch", "delegate_task", ...
})
```

```python
源码位置: agent/tool_guardrails.py:63-81

@dataclass(frozen=True)
class ToolCallGuardrailConfig:
    warnings_enabled: bool = True
    hard_stop_enabled: bool = False
    exact_failure_warn_after: int = 2      # 相同调用相同参数失败 2 次后警告
    exact_failure_block_after: int = 5     # 5 次后阻断
    same_tool_failure_warn_after: int = 3  # 同一工具不同参数失败 3 次后警告
    same_tool_failure_halt_after: int = 8  # 8 次后停机
    no_progress_warn_after: int = 2        # 幂等工具返回相同结果 2 次后警告
    no_progress_block_after: int = 5       # 5 次后阻断
```

---

## 信任模型 (SECURITY.md)

### 核心假设

> "The core assumption is that Hermes is a **personal agent** with one trusted operator."

| 信任边界 | 声明 |
|----------|------|
| 单租户 | 系统保护操作者免受 LLM 行为侵害, 不防恶意同租户 |
| Gateway 会话 | 授权调用者 (Telegram/Discord/Slack) 等信任, session key 仅路由 |
| 默认执行 | terminal.backend: local (直接主机执行), 容器为 opt-in |
| Skills vs MCP | Skills 高信任 (等同主机代码), MCP 低信任 (环境过滤 + OSV) |

### 不在范围内的非漏洞

| 类型 | 原因 |
|------|------|
| Prompt injection (未绕过审批/沙盒) | 不构成信任边界穿越 |
| 公网暴露 Gateway | 部署配置问题, 不是代码漏洞 |
| 对 ~/.hermes/ 的写访问 | 操作者自有文件 |
| terminal backend: local | 文档化默认行为 |

---

## 汇总表

| 组件 | 行数 | 职责 |
|------|------|------|
| `tools/url_safety.py` | 231 | L1 SSRF 防护 — DNS 解析后 IP 检查 + 云元数据始终封锁 |
| `tools/approval.py` | 1258 | L2 危险命令审批 — 硬线封锁 + 三模式门控 + smart approval |
| `tools/mcp_tool.py` (_build_safe_env) | ~40 | L3 MCP 子进程环境白名单过滤 |
| `tools/code_execution_tool.py` (env strip) | ~50 | L3 沙盒子进程 API Key 黑名单 + 白名单过滤 |
| `tools/delegate_tool.py` (depth/blocklist) | ~50 | L4 子代理安全 — 禁递归 + 深度限制 + 工具裁剪 |
| `Dockerfile` + `entrypoint.sh` | 224 | L5 容器安全 — UID 10000 + gosu + tini |
| `agent/redact.py` | 400 | L6 输出脱敏 — 30+ 供应商 + ENV/JSON/URL/JWT |
| `tools/osv_check.py` | 155 | L7 OSV MAL-* 恶意软件查询 |
| `tools/tirith_security.py` | 691 | L7 Tirith 预执行安全扫描 + 自动安装 |
| `tools/website_policy.py` | 282 | URL 黑名单策略 (配置 + 共享文件) |
| `agent/tool_guardrails.py` | 455 | 工具调用循环护栏 (精确/同工具/无进展) |
| `SECURITY.md` | 85 | 信任模型声明 + 非漏洞范围 |

---

[← 09 — 上下文与记忆引擎](/zh-CN/chapters/09-context-memory) | [→ 11 — 技能与自进化系统](/zh-CN/chapters/11-skills-self-improvement)