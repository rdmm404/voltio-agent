# Nanobot Fork Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Switch both production Nanobot profiles to this fork through an editable uv tool install, preserve the WhatsApp progress-delta fix in source control, and keep the WhatsApp-only `voltio` bridge aligned with repo updates.

**Architecture:** The gateway command remains `~/.local/bin/nanobot`; uv rewires that tool entrypoint to import this repo in editable mode. Runtime profile data remains under `~/.nanobot/profiles`, with `voltio` as WhatsApp-only and `wattson` as Telegram-only. The WhatsApp bridge remains a separate systemd-managed Node service and is rebuilt only for `voltio`.

**Tech Stack:** Python 3.11+/3.13 uv tool environment, Nanobot `nanobot-ai`, systemd user services, Node.js/npm for the WhatsApp bridge, git remotes for fork maintenance.

---

## File Structure

- Read-only reference: `docs/superpowers/specs/2026-06-03-nanobot-fork-migration-design.md`
- No repo source files are modified by this migration plan.
- External runtime paths touched during execution:
  - `~/.local/bin/nanobot`
  - `~/.local/share/uv/tools/nanobot-ai`
  - `~/.nanobot/profiles/voltio`
  - `~/.nanobot/profiles/wattson`
  - `~/.nanobot/bridge`
  - `~/.config/systemd/user/nanobot-gateway@.service`
  - `~/.config/systemd/user/nanobot-whatsapp-bridge@.service`

---

### Task 1: Preflight Inventory

**Files:**
- Read: `pyproject.toml`
- Read: `nanobot/agent/runner.py`
- Read: `nanobot/agent/loop.py`
- Read external: `~/.config/systemd/user/nanobot-gateway@.service`
- Read external: `~/.config/systemd/user/nanobot-whatsapp-bridge@.service`

- [ ] **Step 1: Confirm repo state**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
git status --short --branch
git log --oneline -n 3
```

Expected:

```text
## main...origin/main [ahead 1]
e9f7cd01 docs: design nanobot fork migration
```

If there are unrelated uncommitted changes, stop and ask before continuing.

- [ ] **Step 2: Confirm current uv tool install**

Run:

```bash
uv tool list
ls -l ~/.local/bin/nanobot
sed -n '1,120p' ~/.local/share/uv/tools/nanobot-ai/uv-receipt.toml
```

Expected:

```text
nanobot-ai v0.1.5.post3
- nanobot
```

and `~/.local/bin/nanobot` should point into `~/.local/share/uv/tools/nanobot-ai/bin/nanobot`.

- [ ] **Step 3: Confirm both production profiles exist**

Run:

```bash
test -f ~/.nanobot/profiles/voltio/config.json
test -f ~/.nanobot/profiles/wattson/config.json
find ~/.nanobot/profiles -maxdepth 2 -type d -print
```

Expected: both `voltio` and `wattson` profile directories are present. `voltio` should include `whatsapp-auth`; `wattson` should not require WhatsApp bridge handling.

- [ ] **Step 4: Confirm systemd service shape**

Run:

```bash
sed -n '1,80p' ~/.config/systemd/user/nanobot-gateway@.service
sed -n '1,80p' ~/.config/systemd/user/nanobot-whatsapp-bridge@.service
systemctl --user list-units 'nanobot-*' --all
```

Expected gateway `ExecStart`:

```text
%h/.local/bin/nanobot gateway --config %h/.nanobot/profiles/%i/config.json --workspace %h/.nanobot/profiles/%i/workspace
```

Expected bridge service is only relevant to `voltio` and starts Node from `~/.nanobot/bridge`.

- [ ] **Step 5: Confirm source contains the WhatsApp progress-delta fix**

Run:

```bash
rg -n "stream_progress_deltas|wants_progress_streaming|supports_progress_deltas" nanobot/agent/runner.py nanobot/agent/loop.py
```

Expected:

```text
nanobot/agent/runner.py:... stream_progress_deltas: bool = True
nanobot/agent/loop.py:... stream_progress_deltas=on_stream is not None
nanobot/agent/runner.py:... spec.stream_progress_deltas
```

---

### Task 2: Add Upstream Remote

**Files:**
- Modify git config only: `.git/config`

- [ ] **Step 1: Check remotes**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
git remote -v
```

Expected before this task: only `origin` may be present.

- [ ] **Step 2: Add upstream if missing**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
if ! git remote get-url upstream >/dev/null 2>&1; then
  git remote add upstream https://github.com/HKUDS/nanobot.git
fi
git fetch upstream
git remote -v
```

Expected: `upstream` appears with `https://github.com/HKUDS/nanobot.git` for fetch and push URL display.

- [ ] **Step 3: Commit**

No commit is required. Git remotes are local clone configuration and should not be committed.

---

### Task 3: Verify Before Runtime Switch

**Files:**
- No files modified.

- [ ] **Step 1: Run lint**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
ruff check nanobot/
```

Expected: command exits 0.

- [ ] **Step 2: Run focused regression tests for progress behavior**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
pytest tests/agent/test_loop_progress.py -v
```

Expected: command exits 0, including the non-streaming channel regression that prevents WhatsApp token/progress spam.

- [ ] **Step 3: Run broader tests if time allows**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
pytest -q
```

Expected: command exits 0. If this is too slow or fails outside the migration surface, capture the failure and decide whether it blocks the runtime switch.

---

### Task 4: Install Editable uv Tool

**Files:**
- External uv tool env modified: `~/.local/share/uv/tools/nanobot-ai`
- External command shim preserved: `~/.local/bin/nanobot`

- [ ] **Step 1: Stop production gateways before replacing the tool env**

Run:

```bash
systemctl --user stop nanobot-gateway@voltio.service
systemctl --user stop nanobot-gateway@wattson.service
```

Expected: both commands exit 0. If `wattson` is not currently managed by that exact unit name, list units and identify the concrete `wattson` gateway service name before continuing.

- [ ] **Step 2: Install this repo as the editable uv tool**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
uv tool install --force --editable --with langfuse .
```

Expected: uv completes successfully and leaves `~/.local/bin/nanobot` installed.

- [ ] **Step 3: Verify the command points at the editable fork**

Run:

```bash
ls -l ~/.local/bin/nanobot
nanobot --version
python - <<'PY'
import nanobot
from pathlib import Path
print(Path(nanobot.__file__).resolve())
PY
```

Expected:

```text
0.2.1
/home/rdmm404/ai/voltio-agent/nanobot/__init__.py
```

If `python` imports a different Nanobot than the tool env, run this instead:

```bash
~/.local/share/uv/tools/nanobot-ai/bin/python - <<'PY'
import nanobot
from pathlib import Path
print(Path(nanobot.__file__).resolve())
PY
```

Expected path remains `/home/rdmm404/ai/voltio-agent/nanobot/__init__.py`.

---

### Task 5: Verify WhatsApp Bridge Token Compatibility

**Files:**
- Read external: `~/.nanobot/profiles/voltio/whatsapp-auth/bridge-token`
- Read external: `~/.nanobot/profiles/voltio/whatsapp-auth/bridge-token.env`

- [ ] **Step 1: Inspect token files without printing secret values**

Run:

```bash
test -f ~/.nanobot/profiles/voltio/whatsapp-auth/bridge-token
test -f ~/.nanobot/profiles/voltio/whatsapp-auth/bridge-token.env
python - <<'PY'
from pathlib import Path

base = Path.home() / ".nanobot" / "profiles" / "voltio" / "whatsapp-auth"
plain = (base / "bridge-token").read_text(encoding="utf-8").strip()
env_text = (base / "bridge-token.env").read_text(encoding="utf-8").strip()
prefix = "BRIDGE_TOKEN="
env = ""
for line in env_text.splitlines():
    if line.startswith(prefix):
        env = line[len(prefix):].strip().strip('"').strip("'")
print("plain_token_present", bool(plain))
print("env_token_present", bool(env))
print("tokens_match", plain == env)
PY
```

Expected:

```text
plain_token_present True
env_token_present True
tokens_match True
```

- [ ] **Step 2: Fix token env only if needed**

If `tokens_match False`, stop and ask before editing. The safe fix is to regenerate `bridge-token.env` from the plain token, but do not do that without explicit approval because it touches a secret-bearing runtime file.

---

### Task 6: Rebuild the WhatsApp Bridge for voltio

**Files:**
- External runtime directory replaced: `~/.nanobot/bridge`

- [ ] **Step 1: Stop voltio bridge and gateway**

Run:

```bash
systemctl --user stop nanobot-gateway@voltio.service
systemctl --user stop nanobot-whatsapp-bridge@voltio.service
```

Expected: both commands exit 0.

- [ ] **Step 2: Rebuild bridge through normal Nanobot login path**

Run:

```bash
rm -rf ~/.nanobot/bridge
nanobot channels login whatsapp --config ~/.nanobot/profiles/voltio/config.json
```

Expected:

- `~/.nanobot/bridge` is recreated from `/home/rdmm404/ai/voltio-agent/bridge`.
- `npm install` and `npm run build` complete.
- The command starts the bridge for login. If QR login is already valid, interrupt with `Ctrl+C` after the bridge is ready.

- [ ] **Step 3: Verify bridge build artifacts**

Run:

```bash
test -f ~/.nanobot/bridge/dist/index.js
test -f ~/.nanobot/bridge/.nanobot-bridge-source-hash
ls -l ~/.nanobot/bridge/dist/index.js ~/.nanobot/bridge/.nanobot-bridge-source-hash
```

Expected: both files exist.

---

### Task 7: Restart Production Services

**Files:**
- No repo files modified.

- [ ] **Step 1: Start voltio bridge and gateway**

Run:

```bash
systemctl --user start nanobot-whatsapp-bridge@voltio.service
systemctl --user start nanobot-gateway@voltio.service
```

Expected: both commands exit 0.

- [ ] **Step 2: Start wattson gateway**

Run:

```bash
systemctl --user start nanobot-gateway@wattson.service
```

Expected: command exits 0. There is no WhatsApp bridge restart for `wattson`.

- [ ] **Step 3: Check service health**

Run:

```bash
systemctl --user status nanobot-whatsapp-bridge@voltio.service --no-pager
systemctl --user status nanobot-gateway@voltio.service --no-pager
systemctl --user status nanobot-gateway@wattson.service --no-pager
```

Expected: all deployed services report `active (running)`.

- [ ] **Step 4: Check recent logs**

Run:

```bash
journalctl --user -u nanobot-whatsapp-bridge@voltio.service -n 80 --no-pager
journalctl --user -u nanobot-gateway@voltio.service -n 120 --no-pager
journalctl --user -u nanobot-gateway@wattson.service -n 120 --no-pager
```

Expected:

- `voltio` gateway connects to WhatsApp bridge.
- `wattson` gateway starts without WhatsApp bridge errors.
- No repeated import errors, config load errors, or provider setup errors.

---

### Task 8: Smoke Test Runtime Behavior

**Files:**
- No repo files modified.

- [ ] **Step 1: Smoke test voltio WhatsApp**

Send a normal WhatsApp message to `voltio`.

Expected:

- One final assistant response is delivered.
- No stream of token/progress deltas appears in WhatsApp.
- Logs show a normal turn completion.

- [ ] **Step 2: Smoke test wattson Telegram**

Send a normal Telegram message to `wattson`.

Expected:

- One assistant response is delivered.
- Gateway logs show the `wattson` profile config and workspace.
- There are no WhatsApp bridge logs associated with `wattson`.

- [ ] **Step 3: Verify command still uses fork after service restart**

Run:

```bash
~/.local/share/uv/tools/nanobot-ai/bin/python - <<'PY'
import nanobot
from pathlib import Path
print(Path(nanobot.__file__).resolve())
PY
```

Expected:

```text
/home/rdmm404/ai/voltio-agent/nanobot/__init__.py
```

---

### Task 9: Document Rollback Commands

**Files:**
- No files modified unless the operator explicitly chooses to add a local runbook as a separate follow-up change.

- [ ] **Step 1: Roll back Python package if editable install fails**

Run only if rollback is needed:

```bash
systemctl --user stop nanobot-gateway@voltio.service
systemctl --user stop nanobot-gateway@wattson.service
uv tool install --force --with langfuse nanobot-ai==0.1.5.post3
systemctl --user start nanobot-gateway@voltio.service
systemctl --user start nanobot-gateway@wattson.service
```

Expected: `nanobot --version` and `uv tool list` return the previous installed package version.

- [ ] **Step 2: Roll back bridge from backup if bridge rebuild fails**

Before Task 6, make a manual backup if preserving the current bridge is required:

```bash
cp -a ~/.nanobot/bridge ~/.nanobot/bridge.backup-before-fork-migration
```

Run only if rollback is needed:

```bash
systemctl --user stop nanobot-whatsapp-bridge@voltio.service
rm -rf ~/.nanobot/bridge
cp -a ~/.nanobot/bridge.backup-before-fork-migration ~/.nanobot/bridge
systemctl --user start nanobot-whatsapp-bridge@voltio.service
```

Expected: bridge service returns to the previous built code.

---

### Task 10: Future Update Workflow

**Files:**
- No repo files modified by the workflow itself, except normal upstream merge commits.

- [ ] **Step 1: Sync upstream through a short-lived branch**

Run:

```bash
cd /home/rdmm404/ai/voltio-agent
git fetch upstream
git switch main
git pull --ff-only origin main
git switch -c sync/upstream-$(date +%Y%m%d)
git merge upstream/main
```

Expected: merge completes or produces conflicts to resolve on the sync branch.

- [ ] **Step 2: Verify local fix survived**

Run:

```bash
rg -n "stream_progress_deltas|wants_progress_streaming|supports_progress_deltas" nanobot/agent/runner.py nanobot/agent/loop.py
ruff check nanobot/
pytest tests/agent/test_loop_progress.py -v
```

Expected: progress-delta gate is still present and focused tests pass.

- [ ] **Step 3: Deploy ordinary source-only update**

After the sync branch is merged to `main`:

```bash
cd /home/rdmm404/ai/voltio-agent
git switch main
git pull --ff-only origin main
systemctl --user restart nanobot-gateway@voltio.service
systemctl --user restart nanobot-gateway@wattson.service
```

Expected: both production gateways run the pulled source because the uv tool install is editable.

- [ ] **Step 4: Reinstall only when package metadata changed**

Run this only if `pyproject.toml`, optional dependencies, console scripts, or Python/tool environment assumptions changed:

```bash
cd /home/rdmm404/ai/voltio-agent
uv tool install --force --editable --with langfuse .
systemctl --user restart nanobot-gateway@voltio.service
systemctl --user restart nanobot-gateway@wattson.service
```

Expected: tool env rebuilds and both gateways start.

- [ ] **Step 5: Rebuild bridge only when `bridge/` changed**

Run Task 6 only if upstream or local changes touched files under `bridge/`. Do not rebuild the WhatsApp bridge for `wattson`.

---

## Self-Review

- Spec coverage: runtime migration, WhatsApp bridge handling, upstream maintenance, test criteria, and both production profiles are each represented by tasks.
- Placeholder scan: no unresolved placeholder markers remain.
- Type consistency: service names, profile names, and paths match the approved design.
