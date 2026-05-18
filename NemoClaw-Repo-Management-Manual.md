# NemoClaw: Repo-Based Installation & Management Manual

> **Scope:** Manage NemoClaw entirely from the local Git clone at `/home/engineer/workspace/NemoClaw` — no `curl | bash` remote installer, no `npm install -g` from registry. Everything runs from source.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Initial Installation from Repo](#2-initial-installation-from-repo)
3. [Verify Installation](#3-verify-installation)
4. [Running NemoClaw (Onboarding)](#4-running-nemoclaw-onboarding)
5. [Updating NemoClaw](#5-updating-nemoclaw)
6. [Rebuilding After Code Changes](#6-rebuilding-after-code-changes)
7. [Uninstalling NemoClaw](#7-uninstalling-nemoclaw)
8. [Reinstalling (Clean Slate)](#8-reinstalling-clean-slate)
9. [Managing the Sandbox Lifecycle](#9-managing-the-sandbox-lifecycle)
10. [Managing Network Policies](#10-managing-network-policies)
11. [Development & Debugging](#11-development--debugging)
12. [Quick Reference Card](#12-quick-reference-card)
13. [Troubleshooting](#13-troubleshooting)

---

## 1. Prerequisites

| Requirement | Minimum | Check Command |
|-------------|---------|---------------|
| Node.js | ≥ 22.16.0 | `node --version` |
| npm | ≥ 10 | `npm --version` |
| Docker | Running daemon | `docker info` |
| Git | Any recent | `git --version` |
| OpenShell | Installed | `which openshell` |
| uv (for docs only) | Any | `which uv` |

The repo lives at:
```
/home/engineer/workspace/NemoClaw
```

All commands below assume this as the working directory unless stated otherwise.

---

## 2. Initial Installation from Repo

The repo's `install.sh` is the **single command** for installation. When run from inside the repo, it detects the source checkout and automatically:

1. Verifies/installs Node.js (skips if already present)
2. Fixes npm permissions if needed
3. Runs `npm install` + `npm run build:cli` (compiles TypeScript CLI)
4. Runs `cd nemoclaw && npm install && npm run build` (compiles plugin)
5. Runs `npm link` (creates global `nemoclaw` symlink pointing back into this repo)
6. Runs the interactive onboarding wizard (gateway, inference providers, sandbox)

### Install (interactive — recommended for first time)
```bash
cd /home/engineer/workspace/NemoClaw
bash scripts/install.sh
```

The onboarding wizard will walk you through:
- Platform detection and preflight checks (hardware, container runtime)
- OpenShell gateway setup
- Inference provider selection (NVIDIA NIM, Anthropic, OpenAI, Ollama, vLLM, etc.)
- API key/credential entry
- Network policy selection
- Sandbox creation
- Agent launch

### Install (non-interactive)
```bash
cd /home/engineer/workspace/NemoClaw
NEMOCLAW_NON_INTERACTIVE=1 \
NEMOCLAW_PROVIDER=anthropic \
NEMOCLAW_POLICY_MODE=suggested \
NEMOCLAW_SANDBOX_NAME=openclaw \
NEMOCLAW_ACCEPT_THIRD_PARTY_SOFTWARE=1 \
  bash scripts/install.sh --non-interactive --yes-i-accept-third-party-software
```

### Build-only (no onboarding)
If you only want the `nemoclaw` CLI available without setting up a sandbox:
```bash
cd /home/engineer/workspace/NemoClaw
NEMOCLAW_INSTALLING=1 npm install --ignore-scripts \
  && npm run build:cli \
  && cd nemoclaw && npm install --ignore-scripts && npm run build && cd .. \
  && npm link
```

> **How `npm link` works:** It creates a global symlink that points back into this repo's `bin/nemoclaw.js`, which loads `dist/nemoclaw`. This means any future rebuild of `dist/` immediately updates the live CLI — no re-linking needed.

---

## 3. Verify Installation

```bash
# CLI resolves?
which nemoclaw

# Version?
nemoclaw --version

# Symlink points to your repo? (not some other install)
ls -la "$(which nemoclaw)"
```

---

## 4. Re-onboarding (Reconfigure Without Reinstalling)

If NemoClaw is already built and linked, and you just want to reconfigure the gateway, change providers, or create a new sandbox:

```bash
# Re-run onboarding only (does NOT rebuild)
nemoclaw onboard
```

To re-run the full installer (rebuild + onboard):
```bash
cd /home/engineer/workspace/NemoClaw
bash scripts/install.sh
```

---

## 5. Updating NemoClaw

Since the global `nemoclaw` symlink points into your repo's `bin/nemoclaw.js` → `dist/`, any rebuild of `dist/` immediately updates the live CLI. No re-linking or re-installing needed.

### Standard Update (most common — rebuild only)
```bash
cd /home/engineer/workspace/NemoClaw
git pull origin main
NEMOCLAW_INSTALLING=1 npm install --ignore-scripts \
  && npm run build:cli \
  && cd nemoclaw && npm install --ignore-scripts && npm run build && cd ..
```

This is sufficient for **code changes, dependency updates, and blueprint/policy edits**. The running sandbox is untouched.

### When to re-run `install.sh` (rare)

Only needed when the update changes **infrastructure** that lives outside the repo:

| Change in the pull | Why `install.sh` is needed |
|---------------------|---------------------------|
| Sandbox Docker image pin changed (`blueprint.yaml`) | Sandbox container needs rebuilding |
| OpenShell version constraints changed | Gateway may need upgrading |
| New inference provider added to blueprint | Provider needs registering with the gateway |
| You want to re-onboard (new sandbox, change provider) | Onboarding wizard handles this |

```bash
cd /home/engineer/workspace/NemoClaw
git pull origin main
bash scripts/install.sh
```

The installer backs up existing sandboxes first, then rebuilds, re-links, and runs onboarding which upgrades stale sandboxes.

### How to tell which kind of update you need
```bash
cd /home/engineer/workspace/NemoClaw
git pull origin main

# Check if blueprint/infra changed — if yes, run install.sh
git diff HEAD@{1} --name-only | grep -E 'blueprint\.yaml|Dockerfile|scripts/install'
```

If that command produces output, run `bash scripts/install.sh`. If empty, a rebuild is enough.

### Update to a Specific Version/Tag
```bash
cd /home/engineer/workspace/NemoClaw
git fetch --tags
git tag -l 'v*' | sort -V        # list available versions
git checkout v0.0.9               # pick one

# Rebuild (usually sufficient)
NEMOCLAW_INSTALLING=1 npm install --ignore-scripts \
  && npm run build:cli \
  && cd nemoclaw && npm install --ignore-scripts && npm run build && cd ..

# OR full install.sh if the version has infra changes
bash scripts/install.sh
```

---

## 6. Rebuilding After Code Changes

If you've made local edits or switched branches:

### Rebuild CLI only
```bash
npm run build:cli
```

### Rebuild plugin only
```bash
cd nemoclaw && npm run build && cd ..
```

### Rebuild everything
```bash
npm run build:cli && cd nemoclaw && npm run build && cd ..
```

### Plugin watch mode (auto-rebuild on changes)
```bash
cd nemoclaw && npm run dev
```

### Verify build is clean
```bash
make check     # runs all linters + hooks
npm test       # runs all test suites
```

---

## 7. Uninstalling NemoClaw

> **Goal:** Remove every host-side modification made during installation so the system is exactly as it was before NemoClaw was installed.

The built-in `uninstall.sh` handles most cleanup but has **known gaps**. The complete uninstall procedure below covers everything.

### What the installation touches (full inventory)

| Category | What Gets Created | Where |
|----------|-------------------|-------|
| **Global CLI** | `nemoclaw` symlink | `$(npm prefix -g)/bin/nemoclaw` |
| **npm module link** | `nemoclaw/` symlink | `$(npm prefix -g)/lib/node_modules/nemoclaw` |
| **npm prefix redirect** | `prefix=~/.npm-global` (if system prefix was read-only) | `~/.npmrc` |
| **~/.npm-global/** | bin/, lib/node_modules/ (if prefix was redirected) | `~/.npm-global/` |
| **Shell profile: PATH block** | `# NemoClaw PATH setup` ... `# end NemoClaw PATH setup` | `~/.bashrc`, `~/.zshrc`, `~/.profile`, fish, tcsh, csh |
| **Shell profile: npm-global PATH** | `# Added by NemoClaw installer` + `export PATH=~/.npm-global/bin:$PATH` | `~/.bashrc`, `~/.zshrc` |
| **Shell profile: nvm** | nvm initialization lines (only if Node.js was missing) | `~/.bashrc`, `~/.zshrc` |
| **~/.local/bin/nemoclaw** | Shim script (if npm global bin ≠ ~/.local/bin) | `~/.local/bin/nemoclaw` |
| **NemoClaw state** | credentials, sandbox registry, onboard session, shields audit | `~/.nemoclaw/` |
| **OpenShell config** | Gateway, provider definitions | `~/.config/openshell/` |
| **NemoClaw config** | Agent overrides | `~/.config/nemoclaw/` |
| **OpenShell gateway** | Docker cluster container + volume | `openshell-cluster-nemoclaw` |
| **OpenShell providers** | nvidia-nim, vllm-local, ollama-local, nvidia-ncp, nim-local | Inside gateway |
| **Sandboxes** | Docker containers per sandbox | Named per sandbox |
| **Docker images** | openshell/openclaw/nemoclaw images | Docker image store |
| **Ollama models** | nemotron-3-super:120b, nemotron-3-nano:30b | `~/.ollama/models/` |
| **Ollama auth proxy** | Background process + PID file | Process + `~/.nemoclaw/ollama-auth-proxy.pid` |
| **Swap file** | /swapfile + /etc/fstab entry (if RAM was low) | `/swapfile` |
| **Temp files** | Logs, service state | `/tmp/nemoclaw-*` |
| **~/.nvm/** | nvm + Node.js (only if Node.js was missing before install) | `~/.nvm/` |
| **Build artifacts** | node_modules/, dist/ | Inside the repo |

### Complete Uninstall Procedure

Run these steps in order. The built-in uninstaller handles steps 1-2; steps 3-8 cover the gaps it misses.

#### Step 1: Kill the ollama-auth-proxy (not handled by uninstall.sh)
```bash
# Kill ollama auth proxy if running
if [ -f ~/.nemoclaw/ollama-auth-proxy.pid ]; then
  kill "$(cat ~/.nemoclaw/ollama-auth-proxy.pid)" 2>/dev/null || true
fi
# Also hunt by process name in case PID file is stale
pkill -f 'ollama-auth-proxy' 2>/dev/null || true
```

#### Step 2: Run the built-in uninstaller
```bash
cd /home/engineer/workspace/NemoClaw
bash uninstall.sh --yes --delete-models
```

This handles:
- Stopping helper services (cloudflared, port-forwards, orphaned openshell processes)
- Deleting OpenShell sandboxes, providers, and gateway
- Removing global `nemoclaw` npm package/link
- Removing Docker containers, images, and volumes
- Removing Ollama models (`--delete-models`)
- Removing swap file (interactive only, may prompt for sudo)
- Cleaning `/tmp/nemoclaw-*` temp files
- Removing `~/.nemoclaw/`, `~/.config/openshell/`, `~/.config/nemoclaw/`
- Removing `# NemoClaw PATH setup` blocks from shell profiles
- Removing `~/.local/bin/nemoclaw` shim (if installer-managed)
- Removing openshell binary (unless `--keep-openshell`)

#### Step 3: Remove ~/.npm-global and reset npm prefix (not handled by uninstall.sh)
The installer may have redirected npm's global prefix to `~/.npm-global` if the system prefix wasn't writable. This change persists after uninstall.

```bash
# Check if npm prefix was redirected
if [ "$(npm config get prefix 2>/dev/null)" = "$HOME/.npm-global" ]; then
  # Reset to default (varies by system; /usr/local is typical)
  npm config delete prefix
  echo "[fixed] npm global prefix reset to default"
fi

# Remove the orphaned directory
rm -rf "$HOME/.npm-global"
echo "[fixed] Removed ~/.npm-global"
```

#### Step 4: Remove "Added by NemoClaw installer" lines from shell profiles (not handled by uninstall.sh)
The uninstaller only removes `# NemoClaw PATH setup` blocks but misses the separate `# Added by NemoClaw installer` lines added by `fix_npm_permissions()`.

```bash
for rc in "$HOME/.bashrc" "$HOME/.zshrc"; do
  [ -f "$rc" ] || continue
  if grep -q "# Added by NemoClaw installer" "$rc" 2>/dev/null; then
    sed -i.bak '/^# Added by NemoClaw installer$/,+1d' "$rc" && rm -f "${rc}.bak"
    echo "[fixed] Removed NemoClaw npm-global PATH entry from $rc"
  fi
done
```

#### Step 5: Remove nvm if it was installed by NemoClaw (conditional)
Only needed if Node.js was **not** present before NemoClaw installation and nvm was installed by the installer. Skip this step if you had Node.js beforehand (this system has Node 22.16.0 pre-installed, so this step is likely unnecessary).

```bash
# Only run this if NemoClaw installed nvm (check for nvm dir)
if [ -d "$HOME/.nvm" ]; then
  echo "WARNING: ~/.nvm exists. If NemoClaw installed it, remove with:"
  echo "  rm -rf ~/.nvm"
  echo "  Then remove nvm lines from ~/.bashrc and ~/.zshrc"
  echo "  (look for lines containing 'NVM_DIR' or 'nvm.sh')"
  echo ""
  echo "Skip this if you installed nvm yourself before NemoClaw."
fi
```

#### Step 6: Verify swap cleanup
The uninstaller skips swap removal in non-interactive mode. Verify it was cleaned:

```bash
if [ -f /swapfile ]; then
  echo "WARNING: /swapfile still exists. If NemoClaw created it, remove with:"
  echo "  sudo swapoff /swapfile"
  echo "  sudo rm -f /swapfile"
  echo "  sudo sed -i '\\|^/swapfile|d' /etc/fstab"
fi
```

#### Step 7: Clean build artifacts from the repo
```bash
cd /home/engineer/workspace/NemoClaw
rm -rf node_modules dist nemoclaw/node_modules nemoclaw/dist .version
```

#### Step 8: Final verification
```bash
echo "=== Verification ==="

# CLI gone?
which nemoclaw 2>/dev/null && echo "FAIL: nemoclaw still in PATH" || echo "OK: nemoclaw not in PATH"

# State dirs gone?
for d in ~/.nemoclaw ~/.config/openshell ~/.config/nemoclaw ~/.npm-global; do
  [ -d "$d" ] && echo "FAIL: $d still exists" || echo "OK: $d gone"
done

# npm prefix sane?
prefix="$(npm config get prefix 2>/dev/null)"
[ "$prefix" = "$HOME/.npm-global" ] && echo "FAIL: npm prefix still points to ~/.npm-global" || echo "OK: npm prefix = $prefix"

# Docker remnants?
docker ps -a --format '{{.Names}}' 2>/dev/null | grep -iE 'openshell|openclaw|nemoclaw' && echo "FAIL: Docker containers remain" || echo "OK: No NemoClaw Docker containers"
docker images --format '{{.Repository}}' 2>/dev/null | grep -iE 'openshell|openclaw|nemoclaw' && echo "FAIL: Docker images remain" || echo "OK: No NemoClaw Docker images"

# Shell profiles clean?
for rc in ~/.bashrc ~/.zshrc; do
  [ -f "$rc" ] || continue
  grep -q -i 'nemoclaw\|npm-global.*Added by' "$rc" && echo "FAIL: $rc still has NemoClaw entries" || echo "OK: $rc clean"
done

# Swap?
[ -f /swapfile ] && echo "WARN: /swapfile exists (may or may not be NemoClaw's)" || echo "OK: No /swapfile"

# Ollama models?
command -v ollama >/dev/null && ollama list 2>/dev/null | grep -q 'nemotron' && echo "WARN: Ollama nemotron models still present" || echo "OK: No NemoClaw Ollama models"

echo "=== Done ==="
```

### All-in-One Complete Uninstall Script (copy-paste)

```bash
#!/usr/bin/env bash
# Complete NemoClaw uninstall — covers all gaps in the built-in uninstaller.
set -euo pipefail

REPO="/home/engineer/workspace/NemoClaw"

echo ">>> Step 1: Kill ollama-auth-proxy"
if [ -f ~/.nemoclaw/ollama-auth-proxy.pid ]; then
  kill "$(cat ~/.nemoclaw/ollama-auth-proxy.pid)" 2>/dev/null || true
fi
pkill -f 'ollama-auth-proxy' 2>/dev/null || true

echo ">>> Step 2: Run built-in uninstaller"
bash "$REPO/uninstall.sh" --yes --delete-models

echo ">>> Step 3: Reset npm prefix and remove ~/.npm-global"
if [ "$(npm config get prefix 2>/dev/null)" = "$HOME/.npm-global" ]; then
  npm config delete prefix
fi
rm -rf "$HOME/.npm-global"

echo ">>> Step 4: Remove leftover shell profile entries"
for rc in "$HOME/.bashrc" "$HOME/.zshrc"; do
  [ -f "$rc" ] || continue
  if grep -q "# Added by NemoClaw installer" "$rc" 2>/dev/null; then
    sed -i.bak '/^# Added by NemoClaw installer$/,+1d' "$rc" && rm -f "${rc}.bak"
  fi
done

echo ">>> Step 5: Verify swap cleanup"
if [ -f /swapfile ] && [ -f "$HOME/.nemoclaw/managed_swap" ] 2>/dev/null; then
  sudo swapoff /swapfile 2>/dev/null || true
  sudo rm -f /swapfile
  sudo sed -i '\|^/swapfile|d' /etc/fstab 2>/dev/null || true
fi

echo ">>> Step 6: Clean repo build artifacts"
rm -rf "$REPO/node_modules" "$REPO/dist" "$REPO/nemoclaw/node_modules" "$REPO/nemoclaw/dist" "$REPO/.version"

echo ">>> Complete uninstall finished."
```

### Partial uninstall (CLI only, keep sandbox running)
```bash
npm unlink -g nemoclaw
```
This just removes the global `nemoclaw` command. The sandbox, gateway, and all state remain untouched. Re-link anytime with `cd /home/engineer/workspace/NemoClaw && npm link`.

### What is preserved (intentionally) after complete uninstall
- Docker engine
- Node.js and npm (unless NemoClaw installed them via nvm — see Step 5)
- Ollama daemon (models removed with `--delete-models`)
- The repo at `/home/engineer/workspace/NemoClaw` (code untouched; only build artifacts removed)

---

## 8. Reinstalling (Clean Slate)

Use the complete uninstall above, then install fresh:

```bash
cd /home/engineer/workspace/NemoClaw

# 1. Complete uninstall (all-in-one script above, or step by step)

# 2. Fresh install + onboard in one command
bash scripts/install.sh
```

---

## 9. Managing the Sandbox Lifecycle

These commands work once NemoClaw is installed and onboarded.

| Task | Command |
|------|---------|
| List sandboxes | `nemoclaw list` |
| Create/onboard a new sandbox | `nemoclaw onboard` |
| Connect to a sandbox | `nemoclaw <name> connect` |
| Destroy a sandbox | `nemoclaw <name> destroy` |
| Rebuild a sandbox | `nemoclaw <name> rebuild` |
| Backup all sandboxes | `nemoclaw backup-all` |
| Upgrade stale sandboxes | `nemoclaw upgrade-sandboxes --auto` |
| Show sandbox config | `nemoclaw <name> config get` |
| Show policy/shields status | `nemoclaw <name> shields status` |
| List policies | `nemoclaw <name> policy-list` |
| Add a policy | `nemoclaw <name> policy-add` |

### Sandbox Rebuild (detailed)

`nemoclaw <name> rebuild` destroys and recreates a sandbox while **preserving workspace data**. The process:

1. **Backup** — Copies sandbox state dirs (workspace files, agent config) to `~/.nemoclaw/rebuild-backups/<name>/` with a `rebuild-manifest.json` tracking what was saved. Also captures applied policy presets from the registry (since presets live in the gateway, not the sandbox filesystem, and are lost on destroy/recreate).
2. **Destroy** — Removes the sandbox container.
3. **Recreate** — Creates a fresh sandbox from the current blueprint (picks up new Docker image pins, updated policies, etc.).
4. **Restore** — Copies backed-up state dirs back into the new sandbox and re-applies policy presets.

**When to use rebuild:**
- After updating `blueprint.yaml` (new sandbox image, changed policies)
- To apply a new inference provider or rotated API token to an existing sandbox
- When a sandbox is in a broken state but you want to keep your workspace files
- After `nemoclaw upgrade-sandboxes --auto` flags a sandbox as stale

**Rebuild vs Destroy:**
- `rebuild` = destroy + recreate + restore state (preserves your work)
- `destroy` = permanent removal (workspace data is lost unless you ran `nemoclaw backup-all` first)

---

## 10. Managing Network Policies

Policy definitions live in the repo:

| File | Purpose |
|------|---------|
| `nemoclaw-blueprint/policies/openclaw-sandbox.yaml` | Standard policy |
| `nemoclaw-blueprint/policies/openclaw-sandbox-permissive.yaml` | Relaxed policy |
| `nemoclaw-blueprint/policies/tiers.yaml` | Policy tier definitions |
| `nemoclaw-blueprint/policies/presets/*.yaml` | Per-service presets (slack, github, npm, etc.) |
| `nemoclaw-blueprint/policies/presets/local-inference.yaml` | Local/private network inference routing |

To add a custom policy preset:
1. Create a YAML file in `nemoclaw-blueprint/policies/presets/`
2. Follow the structure of existing presets (e.g., `slack.yaml`)
3. Rebuild: `npm run build:cli`
4. Apply via `nemoclaw <name> policy-add`

---

## 11. Development & Debugging

### Type checking
```bash
npm run typecheck:cli          # CLI types
cd nemoclaw && tsc --noEmit    # Plugin types
```

### Linting & formatting
```bash
make check       # All linters (prek hooks)
make format      # Auto-format all code
npm run lint     # ESLint only
npm run lint:fix # ESLint auto-fix
```

### Testing
```bash
npm test                        # All 3 test projects (cli, plugin, e2e)
cd nemoclaw && npm test         # Plugin tests only
```

### Debug a running sandbox
```bash
bash scripts/debug.sh
```

### Validate config schemas
```bash
npm run validate:configs
```

### Build documentation
```bash
make docs          # Build HTML docs
make docs-live     # Live-reload dev server
make docs-clean    # Clean build artifacts
```

---

## 12. Quick Reference Card

```
REPO = /home/engineer/workspace/NemoClaw

┌─────────────────────┬────────────────────────────────────────────────────────┐
│ INSTALL             │ cd $REPO && bash scripts/install.sh                    │
│ (build+link+onboard)│   (does everything in one shot)                        │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ UPDATE (code only)  │ cd $REPO && git pull origin main                       │
│                     │ NEMOCLAW_INSTALLING=1 npm install --ignore-scripts     │
│                     │ npm run build:cli                                      │
│                     │ cd nemoclaw && npm install --ignore-scripts            │
│                     │   && npm run build && cd ..                            │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ UPDATE (infra)      │ cd $REPO && git pull origin main                       │
│                     │ bash scripts/install.sh                                │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ REBUILD (quick,     │ cd $REPO && npm run build:cli                          │
│  no onboard)        │ cd nemoclaw && npm run build && cd ..                  │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ UNINSTALL           │ (use complete uninstall script from Section 7)         │
│ (complete)          │                                                        │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ REINSTALL           │ (complete uninstall, then)                             │
│                     │ cd $REPO && bash scripts/install.sh                    │
├─────────────────────┼────────────────────────────────────────────────────────┤
│ RE-ONBOARD ONLY     │ nemoclaw onboard                                      │
│ CHECK VERSION       │ nemoclaw --version                                     │
│ CHECK LINK          │ ls -la "$(which nemoclaw)"                             │
│ RUN TESTS           │ npm test                                               │
│ LINT ALL            │ make check                                             │
└─────────────────────┴────────────────────────────────────────────────────────┘
```

---

## 13. Troubleshooting

### `nemoclaw: command not found` after `npm link`
Your npm global bin may not be in PATH. Check and fix:
```bash
npm config get prefix
# Add <prefix>/bin to your PATH in ~/.bashrc or ~/.zshrc
export PATH="$(npm config get prefix)/bin:$PATH"
```

### `npm link` fails with EACCES
npm's global prefix points to a system directory. Redirect it:
```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
export PATH="$HOME/.npm-global/bin:$PATH"
# Add the export to your shell profile
npm link
```

### Build fails: `tsc not found`
Dependencies weren't installed. Run:
```bash
NEMOCLAW_INSTALLING=1 npm install --ignore-scripts
```

### Plugin build fails
```bash
cd nemoclaw
rm -rf node_modules dist
npm install --ignore-scripts
npm run build
```

### Sandbox won't start after update
The blueprint or Docker image pin may have changed. Re-run onboarding:
```bash
bash /home/engineer/workspace/NemoClaw/scripts/install.sh
```

### State is corrupted
Reset state without touching the repo:
```bash
rm -rf ~/.nemoclaw ~/.config/openshell ~/.config/nemoclaw
nemoclaw onboard
```

### Git hooks failing on commit
Hooks are managed by prek. Reinstall them:
```bash
prek install
```

### Check what version is actually running
```bash
nemoclaw --version
git -C /home/engineer/workspace/NemoClaw describe --tags --match 'v*' 2>/dev/null || cat /home/engineer/workspace/NemoClaw/package.json | grep version
```

---

## Key Directories Reference

| Path | Purpose |
|------|---------|
| `/home/engineer/workspace/NemoClaw/` | Source repo (your single source of truth) |
| `~/.nemoclaw/` | Runtime state, credentials, sandbox registry |
| `~/.config/openshell/` | OpenShell gateway config |
| `~/.config/nemoclaw/` | Additional NemoClaw state |
| `/sandbox/.openclaw/` | OpenClaw immutable config (inside sandbox container, root-owned, read-only) |
| `/sandbox/.openclaw-data/` | OpenClaw writable data (inside sandbox container, sandbox-owned) |

---

*Generated 2026-04-24. Based on NemoClaw v0.1.0 source at `/home/engineer/workspace/NemoClaw`.*
