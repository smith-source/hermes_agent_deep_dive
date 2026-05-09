# 10 — Security Architecture: Seven-Layer Defense-in-Depth and Trust Model

## Source Files

| File | Lines | Core Responsibility |
|------|------|---------------------|
| `tools/url_safety.py` | 231 | L1 SSRF protection — private/internal/metadata IP interception |
| `tools/approval.py` | 1258 | L2 dangerous command approval — three-mode gating + hardline floor |
| `tools/mcp_tool.py` | 3175 | L3 credential isolation — MCP subprocess environment filtering |
| `tools/code_execution_tool.py` | 1621 | L3 credential isolation — sandbox API Key stripping |
| `tools/delegate_tool.py` | 2562 | L4 sub-agent security — depth limiting + toolset pruning |
| `Dockerfile` + `docker/entrypoint.sh` | 84+140 | L5 container security — non-root UID 10000 + gosu privilege dropping |
| `agent/redact.py` | 400 | L6 output redaction — RedactingFormatter + 30+ vendor regexes |
| `tools/osv_check.py` | 155 | L7 supply chain — OSV MAL-* queries |
| `tools/tirith_security.py` | 691 | L7 supply chain — Tirith pre-execution security scanning |
| `tools/website_policy.py` | 282 | Website URL blocklist policy |
| `agent/tool_guardrails.py` | 455 | Tool call loop guardrails |
| `SECURITY.md` | 85 | Trust model and security policy declaration |

> **One-line summary**: Hermes protects the operator from LLM behavior through seven layers of defense-in-depth (SSRF > approval > credential isolation > sub-agent security > container > redaction > supply chain) under a single-trust-domain assumption.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    User Interaction Layer (CLI / Gateway)│
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ approval │  │ tirith_sec   │  │ tool_guardrails    │  │
│  │ .py      │  │ .py          │  │ .py                │  │
│  │ (L2gate) │  │ (L7scan)     │  │ (loop guardrails)  │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────────┘  │
│       │               │                  │                │
│       ▼               ▼                  ▼                │
│  ┌──────────────────────────────────────────────────────┐│
│  │         check_all_command_guards() unified entry     ││
│  │   hardline → yolo → tirith → dangerous → smart/approve││
│  └────┬─────────────────────────────────────────────────┘│
└───────┼──────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│                    Execution Layer (terminal / code / delegate)│
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ url_saf  │  │ mcp_tool     │  │ delegate_tool      │  │
│  │ ety.py   │  │ (env filter) │  │ (depth+tool prune) │  │
│  │ (L1SSRF) │  │ (L3cred)     │  │ (L4sub-agent)      │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────────┘  │
│       │               │                  │                │
│       ▼               ▼                  ▼                │
│  ┌─────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ code_exe │  │ redact.py    │  │ Dockerfile         │  │
│  │ c_tool   │  │ (L6redact)   │  │ (L5container UID10000)│  │
│  │ (L3cred) │  │              │  │                    │  │
│  └────┬─────┘  └──────┬───────┘  └──────┬─────────────┘  │
│       │               │                  │                │
│       ▼               ▼                  ▼                │
│  ┌──────────────────────────────────────────────────────┐│
│  │               osv_check.py (L7 supply chain)          ││
│  └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## TL;DR

Hermes assumes a single trust domain (one trusted operator) and protects the operator from LLM behavior through seven layers of defense-in-depth. L1 SSRF protection intercepts private networks and cloud metadata endpoints (169.254.169.254 etc.); L2 approval system uses a hardline floor + three-mode gating (manual/smart/off) + auxiliary LLM risk assessment; L3 credential isolation filters environment variables separately for MCP subprocesses and sandbox subprocesses; L4 sub-agent security prohibits recursive delegation and limits depth to 1-3; L5 container security uses non-root UID 10000 + gosu privilege dropping; L6 output redaction covers 30+ vendor API Key prefixes; L7 supply chain security includes CI .pth scanning, OSV MAL-* queries, and Tirith pre-execution scanning. SECURITY.md explicitly declares trust boundaries and out-of-scope non-vulnerability types.

---

## L1 — SSRF Protection (tools/url_safety.py)

### Core Mechanism

`is_safe_url()` performs DNS resolution on each URL and checks the target IP, failing closed when resolution fails:

```python
Source location: tools/url_safety.py:155-231

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

### Always-Blocked Cloud Metadata Endpoints

```python
Source location: tools/url_safety.py:39-57

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

### CGNAT Range Explicit Blocking

Standard `ipaddress.is_private` does not cover 100.64.0.0/10 (RFC 6598 CGNAT); Hermes explicitly adds:

```python
Source location: tools/url_safety.py:70

_CGNAT_NETWORK = ipaddress.ip_network("100.64.0.0/10")
```

---

## L2 — Dangerous Command Approval (tools/approval.py)

### Hardline Floor

The hardline floor is an unbypassable baseline, covering rm -rf /, mkfs, dd to raw block devices, fork bombs, kill -1, shutdown/reboot, etc.:

```python
Source location: tools/approval.py:156-190

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

The `_CMDPOS` regex anchors the command start position, preventing false positives like "echo reboot":

```python
Source location: tools/approval.py:147-154

_CMDPOS = (
    r'(?:^|[;&|\n`]|\$\()'         # start position
    r'\s*'                          # optional whitespace
    r'(?:sudo\s+(?:-[^\s]+\s+)*)?'  # optional sudo with flags
    r'(?:env\s+(?:\w+=\S*\s+)*)?'   # optional env with VAR=VAL pairs
    r'(?:(?:exec|nohup|setsid|time)\s+)*'  # optional wrapper commands
    r'\s*'
)
```

### Three-Mode Gating

```python
Source location: tools/approval.py:716-719

def _get_approval_mode() -> str:
    """Read the approval mode from config. Returns 'manual', 'smart', or 'off'."""
    mode = _get_approval_config().get("mode", "manual")
    return _normalize_approval_mode(mode)
```

| Mode | Behavior | Applicable Scenario |
|------|----------|---------------------|
| `manual` | Prompt user for every dangerous command | Default, interactive use |
| `smart` | Auxiliary LLM automatically assesses risk; approve/deny/escalate | High-frequency operations, reduce interruptions |
| `off` | Skip all approval prompts (hardline still enforced) | cron/batch, YOLO |

### Smart Approval (Auxiliary LLM Risk Assessment)

```python
Source location: tools/approval.py:743-787

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

### Unified Entry: check_all_command_guards()

`check_all_command_guards()` merges Tirith and dangerous command detection into a single approval decision, preventing gateway force=True replay attacks that bypass one check but not the other:

```python
Source location: tools/approval.py:920-988

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

### Gateway Blocking Approval

Gateway uses `_ApprovalEntry` + `threading.Event` to implement blocking approval, preventing deadlocks from parallel sub-agents:

```python
Source location: tools/approval.py:380-387

class _ApprovalEntry:
    """One pending dangerous-command approval inside a gateway session."""
    __slots__ = ("event", "data", "result")
    def __init__(self, data: dict):
        self.event = threading.Event()
        self.data = data          # command, description, pattern_keys, ...
        self.result: Optional[str] = None  # "once"|"session"|"always"|"deny"
```

---

## L3 — Credential Isolation

### MCP Subprocess Environment Filtering (tools/mcp_tool.py)

MCP stdio subprocesses only receive safe baseline variables + user explicitly declared env blocks:

```python
Source location: tools/mcp_tool.py:251-292

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

Credentials in error messages are also redacted:

```python
Source location: tools/mcp_tool.py:256-301

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

### Sandbox API Key Stripping (tools/code_execution_tool.py)

`execute_code` subprocess uses dual whitelist + blacklist filtering:

```python
Source location: tools/code_execution_tool.py:1028-1056

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

Sandbox output is also redacted via `redact_sensitive_text()`:

```python
Source location: tools/code_execution_tool.py:1236-1242

from agent.redact import redact_sensitive_text
stdout_text = redact_sensitive_text(stdout_text)
stderr_text = redact_sensitive_text(stderr_text)
```

### Comparison: Credential Isolation Strategies

| Dimension | MCP Subprocess (mcp_tool.py) | Sandbox Subprocess (code_execution_tool.py) |
|-----------|------------------------------|---------------------------------------------|
| Filtering strategy | Whitelist (_SAFE_ENV_KEYS + XDG_* + user_env) | Whitelist prefixes + blacklist substrings + skill passthrough |
| SECRET detection | None (whitelist only) | Yes (_SECRET_SUBSTRINGS blacklist) |
| Error redaction | _sanitize_error (credential regex) | redact_sensitive_text (30+ vendor regexes) |
| User-declared env passthrough | Yes (server config env block) | Yes (terminal.env_passthrough + skill env_passthrough) |
| HOME isolation | None | Yes (get_subprocess_home() redirect) |

---

## L4 — Sub-Agent Security (tools/delegate_tool.py)

### Recursive Delegation Prohibition

```python
Source location: tools/delegate_tool.py:39-48

DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",  # no recursive delegation
    "clarify",        # no user interaction
    "memory",         # no writes to shared MEMORY.md
    "send_message",   # no cross-platform side effects
    "execute_code",   # children should reason step-by-step
])
```

### Depth Limiting

```python
Source location: tools/delegate_tool.py:128-132

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

### Sub-Agent Approval Callback

```python
Source location: tools/delegate_tool.py:68-74

def _subagent_auto_deny(command: str, description: str, **kwargs) -> str:
    """Auto-deny dangerous commands in subagent threads (safe default).
    Returns 'deny' so the subagent sees a refusal it can recover from, and
    never calls input() (which would deadlock the parent TUI)."""
    logger.warning(...)
    return "deny"
```

---

## L5 — Container Security

### Dockerfile Non-Root User

```dockerfile
Source location: Dockerfile:21

RUN useradd -u 10000 -m -d /opt/data hermes
```

### entrypoint.sh Privilege Dropping

```bash
Source location: docker/entrypoint.sh:12-55

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

## L6 — Output Redaction (agent/redact.py)

### 30+ Vendor API Key Prefixes

```python
Source location: agent/redact.py:67-103

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
    ...                                   # 30+ vendors total
]
```

### Multi-Layer Redaction Pipeline

```python
Source location: agent/redact.py:308-389

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

### mask_secret Tiered Redaction

```python
Source location: agent/redact.py:187-231

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

## L7 — Supply Chain Security

### OSV Malware Check (tools/osv_check.py)

Queries the OSV API before launching MCP servers via npx/uvx, only blocking confirmed MAL-* malicious advisories:

```python
Source location: tools/osv_check.py:26-62

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
Source location: tools/osv_check.py:131-155

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

### Tirith Pre-Execution Scanning (tools/tirith_security.py)

Tirith is a Rust-based command security scanner, run as a subprocess; exit codes determine behavior (0=allow, 1=block, 2=warn):

```python
Source location: tools/tirith_security.py:615-691

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

Tirith supports automatic installation (SHA-256 checksum + cosign provenance verification):

```python
Source location: tools/tirith_security.py:282-387

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

### CI Supply Chain Scanning

CI protections described in SECURITY.md:
- GitHub Actions pinned to full commit SHAs
- `supply-chain-audit.yml` blocks .pth files and base64+exec patterns

---

## Website URL Blocklist (tools/website_policy.py)

### Policy Loading and Caching

```python
Source location: tools/website_policy.py:131-199

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

### Access Check

```python
Source location: tools/website_policy.py:232-282

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

## Tool Call Loop Guardrails (agent/tool_guardrails.py)

### Detection Mechanism

`ToolCallGuardrailController` tracks three loop patterns:

```python
Source location: agent/tool_guardrails.py:19-59

IDEMPOTENT_TOOL_NAMES = frozenset({
    "read_file", "search_files", "web_search", "browser_snapshot", ...
})

MUTATING_TOOL_NAMES = frozenset({
    "terminal", "execute_code", "write_file", "patch", "delegate_task", ...
})
```

```python
Source location: agent/tool_guardrails.py:63-81

@dataclass(frozen=True)
class ToolCallGuardrailConfig:
    warnings_enabled: bool = True
    hard_stop_enabled: bool = False
    exact_failure_warn_after: int = 2      # Warn after 2 identical call+args failures
    exact_failure_block_after: int = 5     # Block after 5
    same_tool_failure_warn_after: int = 3  # Warn after 3 same-tool different-args failures
    same_tool_failure_halt_after: int = 8  # Halt after 8
    no_progress_warn_after: int = 2        # Warn after 2 idempotent tool same-result returns
    no_progress_block_after: int = 5       # Block after 5
```

---

## Trust Model (SECURITY.md)

### Core Assumption

> "The core assumption is that Hermes is a **personal agent** with one trusted operator."

| Trust Boundary | Declaration |
|----------------|-------------|
| Single tenant | System protects operator from LLM behavior, not from malicious co-tenants |
| Gateway session | Authorized callers (Telegram/Discord/Slack) are trusted; session key is routing only |
| Default execution | terminal.backend: local (direct host execution); container is opt-in |
| Skills vs MCP | Skills high trust (equivalent to host code); MCP low trust (environment filtering + OSV) |

### Out-of-Scope Non-Vulnerabilities

| Type | Reason |
|------|--------|
| Prompt injection (not bypassing approval/sandbox) | Does not constitute trust boundary crossing |
| Publicly exposed Gateway | Deployment configuration issue, not a code vulnerability |
| Write access to ~/.hermes/ | Operator's own files |
| terminal backend: local | Documented default behavior |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `tools/url_safety.py` | 231 | L1 SSRF protection — DNS-resolved IP check + cloud metadata always blocked |
| `tools/approval.py` | 1258 | L2 dangerous command approval — hardline floor + three-mode gating + smart approval |
| `tools/mcp_tool.py` (_build_safe_env) | ~40 | L3 MCP subprocess environment whitelist filtering |
| `tools/code_execution_tool.py` (env strip) | ~50 | L3 sandbox subprocess API Key blacklist + whitelist filtering |
| `tools/delegate_tool.py` (depth/blocklist) | ~50 | L4 sub-agent security — prohibit recursion + depth limiting + tool pruning |
| `Dockerfile` + `entrypoint.sh` | 224 | L5 container security — UID 10000 + gosu + tini |
| `agent/redact.py` | 400 | L6 output redaction — 30+ vendors + ENV/JSON/URL/JWT |
| `tools/osv_check.py` | 155 | L7 OSV MAL-* malware queries |
| `tools/tirith_security.py` | 691 | L7 Tirith pre-execution security scanning + auto-installation |
| `tools/website_policy.py` | 282 | URL blocklist policy (config + shared files) |
| `agent/tool_guardrails.py` | 455 | Tool call loop guardrails (exact/same-tool/no-progress) |
| `SECURITY.md` | 85 | Trust model declaration + non-vulnerability scope |

---

[← 09 — Context and Memory Engine](/en/chapters/09-context-memory) | [→ 11 — Skills and Self-Evolution System](/en/chapters/11-skills-self-improvement)