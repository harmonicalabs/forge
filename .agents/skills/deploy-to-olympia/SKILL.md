---
name: deploy-to-olympia
description: Deploy an unmerged xavier branch to A1's Olympia Pi by setting XAVIER_GIT_BRANCH in forge's .env and restarting xavier.service. Use when someone asks to "test <branch> on Olympia", "deploy <branch> to the Pi", or similar. For waking the device with TTS after deploy, use /wake-olympia.
---

# /deploy-to-olympia — Run a specific xavier branch on the Pi

This is for forge-managed Pis (systemd wrapper). For balena-provisioned
devices (such as N14–N18), use the `deploy-to-balena` skill in the xavier repo
instead.

The xavier service wrapper on Olympia hard-resets the xavier checkout to
`origin/$XAVIER_GIT_BRANCH` on every startup. A plain `git checkout <branch>`
on the device is **wiped** on next restart. The only durable way to run a
non-main branch is to set `XAVIER_GIT_BRANCH` in `~/src/forge/.env` and
restart `xavier.service`. The launcher deliberately ignores the legacy
`GIT_BRANCH` variable from `raspberry-pi-client`-era installs.

Arguments: `$ARGUMENTS` — the branch name (e.g. `feat/new-ww`). Accepts an
optional `restore` token to revert the Pi to `main`.

## SSH target

- Host: `olympia` (Tailscale host, passwordless via ssh-agent)
- User: `eric`
- Wrapper env: `~/src/forge/.env` (may define `XAVIER_GIT_BRANCH`; unset means `main`)
- Launcher: `~/src/forge/launch.sh` (this repo's `launch.sh`) runs
  `git fetch origin $XAVIER_GIT_BRANCH && git reset --hard origin/$XAVIER_GIT_BRANCH`
  in `~/src/forge/xavier` on startup
- Service unit: `xavier.service` (system-level, runs as user `eric`)

## Procedure

### 1. Set the branch in the wrapper env

`sed` alone changes nothing when the line is missing, so add it if needed:

```bash
ssh olympia "cd ~/src/forge && if grep -q '^XAVIER_GIT_BRANCH=' .env; then sed -i 's|^XAVIER_GIT_BRANCH=.*|XAVIER_GIT_BRANCH=<branch>|' .env; else echo 'XAVIER_GIT_BRANCH=<branch>' >> .env; fi"
```

(Verify with `ssh olympia "grep XAVIER_GIT_BRANCH ~/src/forge/.env"` after.)

### 2. Restart the service

```bash
ssh olympia "sudo systemctl restart xavier.service"
```

### 3. Wait ~60s for startup

The Python process has to boot, load models, and reach the code path under
test. Watch logs if debugging:

```bash
ssh olympia "sudo journalctl -u xavier.service -f"
```

### 4. Verify you're running the expected commit

The working tree's branch name can lie after a `reset --hard`. Confirm with:

```bash
ssh olympia "cd ~/src/forge/xavier && git rev-parse HEAD && git log -1 --oneline"
```

Match the SHA against `origin/<branch>` from your local xavier clone.

### 5. When done — restore to main

Removing the line puts the launcher back on its default, `main`:

```bash
ssh olympia "sed -i '/^XAVIER_GIT_BRANCH=/d' ~/src/forge/.env && sudo systemctl restart xavier.service"
```

Leaving Olympia on a feature branch is a footgun — the next person to touch
it will be confused about what's running.

## Device layout reference

| Path | What |
|---|---|
| `~/src/forge/` | Wrapper: `.env`, `launch.sh`, service glue |
| `~/src/forge/xavier/` | The xavier checkout (reset hard on boot) |
| `~/src/forge/xavier/app/` | The Python client, its `.env` and `venv/` |
| `~/.kin_speaker_profiles.db` | Speaker enrollment DB (legacy file name, cerebro's default `SPEAKER_ID_DB_PATH`) |
| `/var/lib/xavier/device-config.json` | Device config cache (`KIN_CONFIG_CACHE_PATH`; the `KIN_` prefix is a legacy name) |
| `xavier.service` | systemd unit (this repo's `services/xavier.service`) |

## Related

- `/wake-olympia` (in the xavier repo,
  `${OLYMPIA_REPOS:-$HOME/Code/olympia}/xavier/.agents/skills/wake-olympia`) —
  speak the wake word + optional commands; also accepts a `branch:<name>`
  token for a one-shot combined deploy+wake.
