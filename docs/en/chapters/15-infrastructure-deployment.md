# 15 — Infrastructure & Deployment: Multi-Mode Containerized Deployment and Declarative Configuration


---

## Source Files

| File | Lines | Role |
|------|-------|------|
| `Dockerfile` | 83 | Multi-stage build + gosu + tini + non-root UID |
| `docker-compose.yml` | 59 | Gateway + dashboard dual service orchestration |
| `docker/entrypoint.sh` | 139 | Privilege drop + directory initialization + skill sync |
| `nix/nixosModules.nix` | 980 | Declarative configuration + native/container dual mode |
| `scripts/hermes-gateway` | 416 | systemd/launchd service installation |
| `.github/workflows/tests.yml` | 83 | Test CI |
| `.github/workflows/lint.yml` | ~170 | Code linting CI |
| `.github/workflows/docker-publish.yml` | ~240 | Docker image publishing |
| `.github/workflows/nix.yml` | ~130 | Nix build CI |
| `.github/workflows/osv-scanner.yml` | ~80 | Dependency vulnerability scanning |
| `.github/workflows/supply-chain-audit.yml` | ~120 | Supply chain audit |
| `setup-hermes.sh` | 399 | Developer quick setup |
| `scripts/install.sh` | 1587 | Termux/macOS/Linux installation script |

**One-liner:** The infrastructure layer uses a multi-stage Dockerfile build (gosu privilege drop + tini zombie reap + UID 10000 + Playwright), docker-compose provides gateway/dashboard dual service orchestration, entrypoint.sh handles first-time directory initialization + skill sync + optional dashboard launch, the NixOS module supports declarative configuration and native/container dual-mode execution, systemd/launchd service scripts enable production deployment, and 7+ CI workflows cover testing/linting/image publishing/Nix building/vulnerability scanning/supply chain auditing.

---

## Architecture Overview

```
                    ┌───────────────────────────────────────────────────┐
                    │            Docker Build Pipeline                  │
                    │                                                   │
                    │  Stage 1: uv_source (astral-sh/uv:python3.13)    │
                    │  Stage 2: gosu_source (tianon/gosu:1.19)         │
                    │  Stage 3: debian:13.4 (final image)              │
                    │    ├── apt install (tini, npm, python3, ripgrep) │
                    │    ├── useradd -u 10000 hermes                    │
                    │    ├── COPY gosu + uv from stages 1-2             │
                    │    ├── npm install + Playwright chromium          │
                    │    ├── COPY source + web build + TUI build        │
                    │    ├── uv venv + pip install -e ".[all]"         │
                    │    ├── VOLUME /opt/data                           │
                    │    └────────────────────────────────────────────── │
                    │    ENTRYPOINT: tini -g → entrypoint.sh            │
                    └───────────────────────────────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────┐
                    │       docker/entrypoint.sh (139L)            │
                    │                                              │
                    │  if root:                                    │
                    │    ├── usermod/groupmod HERMES_UID/GID       │
                    │    ├── chown -R $HERMES_HOME                 │
                    │    ├── chown config.yaml (host-editable)     │
                    │    └── exec gosu hermes "$0" "$@"            │
                    │                                              │
                    │  as hermes:                                  │
                    │    ├── source .venv/bin/activate             │
                    │    ├── mkdir -p cron/sessions/logs/...       │
                    │    ├── cp .env.example → .env (if missing)  │
                    │    ├── cp cli-config.yaml → config.yaml     │
                    │    ├── cp SOUL.md (if missing)              │
                    │    ├── python3 "$INSTALL_DIR/tools/skills_sync.py" │
                    │    ├── HERMES_DASHBOARD=1 → background dash │
                    │    └── exec hermes "$@"                     │
                    └──────────────────────────────────────────────┘
                                       │
       ┌───────────────────────────────▼────────────────────────────────┐
       │                    docker-compose.yml                          │
       │                                                                │
       │  gateway:  hermes-agent → gateway run  (network_mode: host)   │
       │  dashboard: hermes-agent → dashboard   (network_mode: host)   │
       │  volumes:  ~/.hermes:/opt/data                                │
       │  env:      HERMES_UID / HERMES_GID                            │
       └────────────────────────────────────────────────────────────────┘
                                       │
       ┌───────────────────────────────▼────────────────────────────────┐
       │                Alternative Deployment Modes                    │
       │                                                                │
       │  ┌─ NixOS Module ──────────────────────────────────────────┐ │
       │  │  services.hermes-agent = {                              │ │
       │  │    enable = true;                                       │ │
       │  │    container.enable = false → native systemd            │ │
       │  │    container.enable = true  → OCI container             │ │
       │  │    settings.model = "anthropic/claude-sonnet-4";       │ │
       │  │    environmentFiles = [ sops secrets ];                │ │
       │  │  }                                                      │ │
       │  └────────────────────────────────────────────────────────┘ │
       │                                                                │
       │  ┌─ systemd/launchd ────┐  ┌─ install.sh ────────┐          │
       │  │ scripts/hermes-gateway│  │ Termux/macOS/Linux  │          │
       │  │ install/start/stop   │  │ 1587 lines          │          │
       │  └──────────────────────┘  └──────────────────────┘          │
       └────────────────────────────────────────────────────────────────┘
```

---

## TL;DR

The deployment system is built around a multi-stage Dockerfile: Stages 1-2 extract the uv and gosu binaries respectively, Stage 3 installs all dependencies on debian:13.4 (tini zombie reaping + npm + Playwright + Python venv). At runtime, the container starts as root, and entrypoint.sh drops privileges to the hermes user (UID 10000) via `usermod + gosu`, initializes the `~/.hermes` directory structure, copies .env/config.yaml/SOUL.md seed files, executes skills_sync.py to sync skills, and optionally launches a dashboard side-process. docker-compose provides gateway + dashboard dual service orchestration. The NixOS module supports declarative configuration (`lib.recursiveUpdate` deep merge) and native/container dual-mode. CI covers 7+ workflows: tests/lint/docker-publish/nix/osv-scanner/supply-chain-audit/deploy-site.

---

## 1. Dockerfile (83 lines)

### 1.1 Multi-Stage Build Design

Three-stage build minimizes final image size: Stage 1 extracts uv, Stage 2 extracts gosu, Stage 3 combines the full runtime.

**Source location: Dockerfile:1-24**

```dockerfile
FROM ghcr.io/astral-sh/uv:0.11.6-python3.13-trixie AS uv_source
FROM tianon/gosu:1.19-trixie AS gosu_source
FROM debian:13.4

# Disable Python stdout buffering
ENV PYTHONUNBUFFERED=1
# Playwright browsers outside volume mount
ENV PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/.playwright

# System deps: tini reaps orphaned zombie processes (MCP stdio, git, bun)
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    build-essential curl nodejs npm python3 ripgrep ffmpeg gcc \
    python3-dev libffi-dev procps git openssh-client docker-cli tini && \
    rm -rf /var/lib/apt/lists/*

# Non-root user: UID 10000, overridable via HERMES_UID
RUN useradd -u 10000 -m -d /opt/data hermes

COPY --chmod=0755 --from=gosu_source /gosu /usr/local/bin/
COPY --chmod=0755 --from=uv_source /usr/local/bin/uv /usr/local/bin/uvx /usr/local/bin/
```

### 1.2 Layer-Cached Dependency Installation

**Source location: Dockerfile:36-56**

```dockerfile
# Layer-cached dependency install — copy only package manifests first
COPY package.json package-lock.json ./
COPY web/package.json web/package-lock.json web/
COPY ui-tui/package.json ui-tui/package-lock.json ui-tui/
COPY ui-tui/packages/hermes-ink/ ui-tui/packages/hermes-ink/

# npm_config_install_links=false forces symlinks for file: deps
ENV npm_config_install_links=false

RUN npm install --prefer-offline --no-audit && \
    npx playwright install --with-deps chromium --only-shell && \
    (cd web && npm install --prefer-offline --no-audit) && \
    (cd ui-tui && npm install --prefer-offline --no-audit) && \
    npm cache clean --force
```

### 1.3 Python venv and Runtime

**Source location: Dockerfile:74-83**

```dockerfile
# Python virtualenv
RUN uv venv && \
    uv pip install --no-cache-dir -e ".[all]"

# Runtime
ENV HERMES_WEB_DIST=/opt/hermes/hermes_cli/web_dist
ENV HERMES_HOME=/opt/data
ENV PATH="/opt/data/.local/bin:${PATH}"
VOLUME [ "/opt/data" ]
ENTRYPOINT [ "/usr/bin/tini", "-g", "--", "/opt/hermes/docker/entrypoint.sh" ]
```

---

## 2. Docker Compose (59 lines)

### 2.1 Dual Service Orchestration

Gateway and Dashboard share the same image, using `network_mode: host` to avoid port mapping overhead.

**Source location: docker-compose.yml:21-59**

```yaml
services:
  gateway:
    build: .
    image: hermes-agent
    container_name: hermes
    restart: unless-stopped
    network_mode: host
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - HERMES_UID=${HERMES_UID:-10000}
      - HERMES_GID=${HERMES_GID:-10000}
    command: ["gateway", "run"]

  dashboard:
    image: hermes-agent
    container_name: hermes-dashboard
    restart: unless-stopped
    network_mode: host
    depends_on:
      - gateway
    volumes:
      - ~/.hermes:/opt/data
    environment:
      - HERMES_UID=${HERMES_UID:-10000}
      - HERMES_GID=${HERMES_GID:-10000}
    # Localhost-only. Remote access via ssh -L 9119:localhost:9119
    command: ["dashboard", "--host", "127.0.0.1", "--no-open"]
```

---

## 3. Docker Entrypoint (`docker/entrypoint.sh`, 139 lines)

### 3.1 Root → hermes Privilege Drop

When started as root (Docker default), the entrypoint remaps the hermes user UID/GID to host values via `usermod/groupmod`, fixes volume ownership, then drops privileges via `exec gosu hermes`.

**Source location: docker/entrypoint.sh:12-55**

```bash
if [ "$(id -u)" = "0" ]; then
    if [ -n "$HERMES_UID" ] && [ "$HERMES_UID" != "$(id -u hermes)" ]; then
        echo "Changing hermes UID to $HERMES_UID"
        usermod -u "$HERMES_UID" hermes
    fi
    if [ -n "$HERMES_GID" ] && [ "$HERMES_GID" != "$(id -g hermes)" ]; then
        groupmod -o -g "$HERMES_GID" hermes 2>/dev/null || true
    fi

    # Fix ownership of the data volume
    actual_hermes_uid=$(id -u hermes)
    needs_chown=false
    if [ -n "$HERMES_UID" ] && [ "$HERMES_UID" != "10000" ]; then
        needs_chown=true
    elif [ "$(stat -c %u "$HERMES_HOME" 2>/dev/null)" != "$actual_hermes_uid" ]; then
        needs_chown=true
    fi
    if [ "$needs_chown" = true ]; then
        chown -R hermes:hermes "$HERMES_HOME" 2>/dev/null || \
            echo "Warning: chown failed (rootless container?) — continuing anyway"
    fi

    # Ensure config.yaml readable even if host-edited
    if [ -f "$HERMES_HOME/config.yaml" ]; then
        chown hermes:hermes "$HERMES_HOME/config.yaml" 2>/dev/null || true
        chmod 640 "$HERMES_HOME/config.yaml" 2>/dev/null || true
    fi

    echo "Dropping root privileges"
    exec gosu hermes "$0" "$@"
fi
```

### 3.2 Directory Initialization and Seed Files

**Source location: docker/entrypoint.sh:57-87**

```bash
# Running as hermes from here
source "${INSTALL_DIR}/.venv/bin/activate"

# Create essential directory structure
mkdir -p "$HERMES_HOME"/{cron,sessions,logs,hooks,memories,skills,skins,plans,workspace,home}

# .env — seed from example if missing
if [ ! -f "$HERMES_HOME/.env" ]; then
    cp "$INSTALL_DIR/.env.example" "$HERMES_HOME/.env"
fi

# config.yaml — seed from example if missing
if [ ! -f "$HERMES_HOME/config.yaml" ]; then
    cp "$INSTALL_DIR/cli-config.yaml.example" "$HERMES_HOME/config.yaml"
fi

# SOUL.md — seed from docker-specific template
if [ ! -f "$HERMES_HOME/SOUL.md" ]; then
    cp "$INSTALL_DIR/docker/SOUL.md" "$HERMES_HOME/SOUL.md"
fi

# Sync bundled skills (manifest-based, preserves user edits)
if [ -d "$INSTALL_DIR/skills" ]; then
    python3 "$INSTALL_DIR/tools/skills_sync.py"
fi
```

### 3.3 Dashboard Side-Process

**Source location: docker/entrypoint.sh:89-122**

```bash
# HERMES_DASHBOARD=1 → background dashboard side-process
case "${HERMES_DASHBOARD:-}" in
    1|true|TRUE|True|yes|YES|Yes)
        dash_host="${HERMES_DASHBOARD_HOST:-0.0.0.0}"
        dash_port="${HERMES_DASHBOARD_PORT:-9119}"
        dash_args=(--host "$dash_host" --port "$dash_port" --no-open)
        # Non-localhost requires --insecure (dashboard exposes API keys)
        if [ "$dash_host" != "127.0.0.1" ] && [ "$dash_host" != "localhost" ]; then
            dash_args+=(--insecure)
        fi
        (
            stdbuf -oL -eL hermes dashboard "${dash_args[@]}" 2>&1 \
                | sed -u 's/^/[dashboard] /'
        ) &
        ;;
esac
```

### 3.4 Final exec

**Source location: docker/entrypoint.sh:124-139**

```bash
# If first positional arg is an executable on PATH → run it directly
# (needed by launcher which runs `sleep infinity` sandbox containers)
# Otherwise → wrap with `hermes` subcommand
if [ $# -gt 0 ] && command -v "$1" >/dev/null 2>&1; then
    exec "$@"
fi
exec hermes "$@"
```

---

## 4. NixOS Module (`nix/nixosModules.nix`, 980 lines)

### 4.1 Declarative Configuration

The NixOS module supports declaring configuration via `services.hermes-agent.settings` as a Nix attrset, automatically converted to YAML and deep-merged with `configFile`.

**Source location: nix/nixosModules.nix:26-48**

```nix
{ inputs, ... }: {
  flake.nixosModules.default = { config, lib, pkgs, ... }:

  let
    cfg = config.services.hermes-agent;
    # Deep-merge config type (from 0xrsydn/nix-hermes-agent)
    deepConfigType = lib.types.mkOptionType {
      name = "hermes-config-attrs";
      description = "Hermes YAML config (attrset), merged deeply.";
      merge = _loc: defs: lib.foldl' lib.recursiveUpdate { } (map (d: d.value) defs);
    };

    # Generate config.yaml from Nix attrset (YAML is superset of JSON)
    configJson = builtins.toJSON cfg.settings;
    generatedConfigFile = pkgs.writeText "hermes-config.yaml" configJson;
    configFile = if cfg.configFile != null then cfg.configFile else generatedConfigFile;

    configMergeScript = pkgs.callPackage ./configMergeScript.nix { };
  in ...
```

### 4.2 Native vs Container Dual Mode

**Source location: nix/nixosModules.nix:1-17**

```nix
# Two modes:
#   container.enable = false (default) → native systemd service
#   container.enable = true            → OCI container (persistent writable layer)
#
# Container mode: hermes runs from /nix/store bind-mounted read-only into a
# plain Ubuntu container. The writable layer (apt/pip/npm installs) persists
# across restarts and agent updates.
```

### 4.3 Container Entry Point Provisioning

The entrypoint in container mode provisions the toolchain on first boot (apt nodejs + curl uv + Python venv), with all installation results persisted in the writable layer.

**Source location: nix/nixosModules.nix:79-149**

```bash
# First boot: provisioning agent tools...
if [ ! -f /var/lib/hermes-tools-provisioned ] && command -v apt-get >/dev/null 2>&1; then
    apt-get update -qq
    apt-get install -y -qq sudo curl ca-certificates gnupg
    # Node 22 via NodeSource — Ubuntu 24.04 ships Node 18 (EOL)
    curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
        | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
    apt-get install -y -qq nodejs
    touch /var/lib/hermes-tools-provisioned
fi
# uv (Python manager) — not in Ubuntu repos
if ! command -v uv >/dev/null 2>&1 && command -v curl >/dev/null 2>&1; then
    ...
fi
```

---

## 5. Systemd/Launchd Service (`scripts/hermes-gateway`, 416 lines)

### 5.1 Service Installation

Supports dual-platform service installation for systemd (Linux) and launchd (macOS).

**Source location: scripts/hermes-gateway:1-23**

```python
"""
Hermes Gateway - Standalone messaging platform integration.

Usage:
    ./scripts/hermes-gateway install   # Install as systemd service
    ./scripts/hermes-gateway start     # Start the service
    ./scripts/hermes-gateway stop      # Stop the service
    ./scripts/hermes-gateway restart   # Restart the service
    ./scripts/hermes-gateway status    # Check service status
    ./scripts/hermes-gateway uninstall # Remove the service
"""
```

---

## 6. CI Workflows

### 6.1 Tests CI

**Source location: .github/workflows/tests.yml:1-83**

```yaml
name: Tests
on:
  push:
    branches: [main]
    paths-ignore: ['**/*.md', 'docs/**']
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv venv .venv --python 3.11 && uv pip install -e ".[all,dev]"
      - run: python -m pytest tests/ -q -n auto
        env:
          OPENROUTER_API_KEY: ""  # Prevent real API calls

  e2e:
    runs-on: ubuntu-latest
    steps:
      - run: python -m pytest tests/e2e/ -v
```

### 6.2 Workflow List

| Workflow | File | Purpose | Trigger |
|----------|------|---------|---------|
| Tests | `tests.yml` | pytest + e2e | push/PR to main |
| Lint | `lint.yml` | ruff + mypy + shellcheck | push/PR to main |
| Docker Publish | `docker-publish.yml` | Multi-arch image build + GHCR push | tag release |
| Nix Build | `nix.yml` | Nix flake check + build | push/PR to main |
| Nix Lockfile Fix | `nix-lockfile-fix.yml` | Auto-update flake lock | weekly schedule |
| OSV Scanner | `osv-scanner.yml` | Dependency vulnerability scan | push/PR + daily |
| Supply Chain Audit | `supply-chain-audit.yml` | PyPI/npm dependency audit | push/PR + daily |
| Deploy Site | `deploy-site.yml` | Docs site deployment | push to main |
| Skills Index | `skills-index.yml` | Update skills catalog | push to main |
| Contributor Check | `contributor-check.yml` | New contributor welcome | PR opened |
| Docs Site Checks | `docs-site-checks.yml` | Link/format validation | push/PR |

---

## 7. Developer Setup (`setup-hermes.sh`, 399 lines)

### 7.1 Quick Setup Script

Developer-oriented first-time setup script that installs dependencies + configures environment + validates installation.

---

## 8. Install Script (`scripts/install.sh`, 1587 lines)

### 8.1 Multi-Platform Installation

Supports installation on three platforms: Termux (Android), macOS, and Linux, including dependency detection + package manager selection + pip/uv installation paths.

---

## Comparison Tables

### Table 1: Deployment Mode Comparison

| Mode | Mechanism | Config Method | Update Method | Isolation | Use Case |
|------|-----------|--------------|---------------|-----------|----------|
| Docker (compose) | docker-compose up | ~/.hermes/config.yaml + .env | docker compose pull + up | Container | Production host |
| Docker (single) | docker run | HERMES_UID + env vars | docker pull + restart | Container | Quick test |
| NixOS Native | systemd unit | Nix attrset → config.yaml | nixos-rebuild switch | Process | NixOS server |
| NixOS Container | podman/docker | Nix attrset → .env in container | Container recreation | OCI container | NixOS with Ubuntu deps |
| systemd service | scripts/hermes-gateway install | ~/.hermes/config.yaml | systemctl restart | Process | Linux server |
| launchd service | scripts/hermes-gateway install | ~/.hermes/config.yaml | launchctl load | Process | macOS |
| Developer | setup-hermes.sh | Project .env + config.yaml | git pull + pip install | None | Local dev |

### Table 2: Entrypoint Phase Comparison

| Phase | Runs As | Actions | Failure Handling |
|-------|---------|---------|-----------------|
| UID remapping | root | usermod/groupmod HERMES_UID/GID | Skip if rootless Podman |
| Volume ownership | root | chown -R hermes:hermes HERMES_HOME | chown fails → warn + continue |
| Config permissions | root | chown + chmod 640 config.yaml | Silently skip on error |
| Privilege drop | root → hermes | exec gosu hermes "$0" "$@" | Fatal if gosu missing |
| Directory seeding | hermes | mkdir + cp .env/config.yaml/SOUL.md | Skip if file exists |
| Skills sync | hermes | python3 "$INSTALL_DIR/tools/skills_sync.py" | Best-effort |
| Dashboard launch | hermes | Background process with --insecure | Process tree cleanup on stop |
| Final exec | hermes | exec hermes "$@" or exec "$@" | Container exit |

### Table 3: CI Workflow Comparison

| Workflow | Runner | Key Tools | Timeout | Security |
|----------|--------|-----------|---------|----------|
| tests.yml | ubuntu-latest | pytest, uv, Python 3.11 | 20 min | API keys empty |
| lint.yml | ubuntu-latest | ruff, mypy, shellcheck | ~10 min | Read-only |
| docker-publish.yml | ubuntu-latest | docker buildx, GHCR | ~30 min | SHA-pinned actions |
| nix.yml | ubuntu-latest | nix flake check | ~15 min | Read-only |
| osv-scanner.yml | ubuntu-latest | google/osv-scanner | ~5 min | Daily schedule |
| supply-chain-audit.yml | ubuntu-latest | pip-audit, npm audit | ~10 min | Daily schedule |

---

## Summary Table

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `Dockerfile` | 83 | Multi-stage build + gosu + tini + non-root UID 10000 |
| `docker-compose.yml` | 59 | Gateway + dashboard dual service orchestration |
| `docker/entrypoint.sh` | 139 | Privilege drop + directory initialization + seed files + skill sync |
| `nix/nixosModules.nix` | 980 | Declarative configuration + native/container dual mode |
| `scripts/hermes-gateway` | 416 | systemd/launchd service installation management |
| `.github/workflows/` | ~800 | 7+ CI workflows (tests/lint/images/Nix/security) |
| `setup-hermes.sh` | 399 | Developer quick setup script |
| `scripts/install.sh` | 1587 | Termux/macOS/Linux multi-platform installation |

---

[<< 14 — Configuration System](/en/chapters/14-configuration-system) | [16 — Plugin System >>](/en/chapters/16-plugin-system)