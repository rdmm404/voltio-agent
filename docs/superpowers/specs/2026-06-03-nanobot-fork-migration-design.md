# Nanobot Fork Migration Design

## Summary

Switch the live Nanobot gateway from the current `uv tool install` package
(`nanobot-ai v0.1.5.post3`) to this fork at `/home/rdmm404/ai/voltio-agent` using
an editable uv tool install. Keep the existing systemd gateway command path
unchanged, preserve the WhatsApp progress-delta hotfix in source control, and
maintain the fork with an explicit upstream remote and short-lived sync branches.

The live gateway should continue to use the two existing profile configs under
`~/.nanobot/profiles`. This design does not move profile data, change gateway
ports, or replace the current systemd unit model.

## Runtime Migration

- Install the fork once as the uv tool source:

  ```bash
  cd /home/rdmm404/ai/voltio-agent
  uv tool install --force --editable --with langfuse .
  ```

- Keep the current gateway systemd unit unchanged. It already runs
  `%h/.local/bin/nanobot gateway --config %h/.nanobot/profiles/%i/config.json --workspace %h/.nanobot/profiles/%i/workspace`.

- Restart the active gateway after the editable install:

  ```bash
  systemctl --user restart nanobot-gateway@voltio.service
  ```

- For ordinary future code updates, do not reinstall the uv tool. Pull the fork
  and restart the gateway:

  ```bash
  cd /home/rdmm404/ai/voltio-agent
  git pull --ff-only origin main
  systemctl --user restart nanobot-gateway@voltio.service
  ```

- Rerun `uv tool install --force --editable --with langfuse .` only when
  dependencies, optional packages, console script metadata, Python version, or the
  uv tool environment itself changes.

## WhatsApp Bridge Handling

The bridge source in this repo is separate from the running Node bridge at
`~/.nanobot/bridge`. The bridge service starts Node directly from that runtime
directory, so restarting systemd does not copy or rebuild bridge source by
itself.

After the initial migration, and after future updates that touch `bridge/`, use
the normal Nanobot rebuild path with the active profile config:

```bash
systemctl --user stop nanobot-gateway@voltio.service
systemctl --user stop nanobot-whatsapp-bridge@voltio.service

rm -rf ~/.nanobot/bridge
nanobot channels login whatsapp --config ~/.nanobot/profiles/voltio/config.json

systemctl --user start nanobot-whatsapp-bridge@voltio.service
systemctl --user start nanobot-gateway@voltio.service
```

Before relying on the rebuilt bridge under systemd, verify that
`~/.nanobot/profiles/voltio/whatsapp-auth/bridge-token` and the
`bridge-token.env` file consumed by the systemd bridge service resolve to the
same token value.

## Fork Maintenance

- Add upstream once:

  ```bash
  git remote add upstream https://github.com/HKUDS/nanobot.git
  git fetch upstream
  ```

- Keep `main` as the production fork branch. Sync upstream changes through a
  short-lived branch:

  ```bash
  git fetch upstream
  git switch main
  git pull --ff-only origin main
  git switch -c sync/upstream-YYYYMMDD
  git merge upstream/main
  ```

- Resolve conflicts while preserving local production fixes. The WhatsApp
  progress-delta fix is already present in this source tree through
  `AgentRunSpec.stream_progress_deltas`, the `AgentLoop` wiring, and the runner
  gate that prevents provider progress deltas from streaming to non-streaming
  channels.

- Run focused verification before merging a sync branch:

  ```bash
  ruff check nanobot/
  pytest
  ```

- After merging to `main`, deploy to the live editable install by pulling the
  repo and restarting systemd. Reinstall the uv tool only for dependency or tool
  metadata changes.

## Test And Acceptance Criteria

- `~/.local/bin/nanobot` still exists and launches the editable fork.
- `nanobot --version` reports the fork version from `pyproject.toml`.
- Each deployed production gateway starts successfully with its matching profile
  config under `~/.nanobot/profiles/<profile>/config.json`.
- WhatsApp receives one final response on non-streaming replies rather than
  token/progress spam.
- If `bridge/` changed, the rebuilt `~/.nanobot/bridge/dist/index.js` starts
  through `nanobot-whatsapp-bridge@voltio.service`.
- `ruff check nanobot/` and relevant pytest coverage pass before upstream syncs
  are deployed.

## Assumptions

- `voltio` and `wattson` are both production profiles for different purposes.
- `voltio` is WhatsApp-only and owns the WhatsApp bridge service/rebuild steps.
- `wattson` is Telegram-only and does not need WhatsApp bridge handling.
- Existing profile data under `~/.nanobot/profiles` is the source of truth and
  will not be migrated.
- WebUI auth, heartbeat/cron changes, tool-surface changes, long-running goal
  behavior, and model routing changes are not treated as migration blockers.
