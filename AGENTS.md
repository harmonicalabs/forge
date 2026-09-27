<!-- blackbird:core — managed by blackbird/scripts/sync-agents -->
> The shared Olympia engineering rules apply here. On a laptop they load from `~/Code/harmonica/AGENTS.md`; their source is `core/AGENTS.core.md` in harmonicalabs/blackbird. Below: notes for this repo only.
<!-- /blackbird:core -->

# forge: the legacy systemd installer/launcher for non-balena Pis

Forge is a bash-only wrapper that installs and runs the `xavier` Pi client (cloned from the `xavier` repo) as a systemd service (`xavier.service`, `otelcol`, `device-monitor`) on older, non-balena-provisioned Raspberry Pis. It clones/updates the client, installs its Python dependencies, manages device authentication (`DEVICE_ID`/`DEVICE_PRIVATE_KEY` against storm's `/auth/device/*`), and drives start/stop/restart/logs through `make`. No HTTP server, no tests, no CI — it's an operator-run installer, not a service. This is the permanent maintenance path for pre-balena Pis (newer Pis are provisioned by balena instead, in the `xavier` repo); only one forge device (Harry's) is currently on record, and it has been offline since 16 Sep.

## Commands
```bash
make install            # runs ./install.sh — one-time Pi setup
make start / stop / restart / status    # systemctl on the xavier unit
make logs / logs-follow / logs-all      # journalctl for xavier (+ otelcol)
make diagnostics        # bash diagnostics/run-device-diagnostics.sh
make uninstall           # ./uninstall.sh
make migrate-to-xavier   # one-shot migration off the old raspberry-pi-client repo (no longer relevant — no installs since 18 Apr)
```
No `make test`/`make check` — there is no test suite for this repo.

## Things that will bite you
- `README.md` is stale: it describes a `respeaker/` directory and `RESPEAKER_TUNING.md` that no longer exist in this repo, and calls itself "raspberry-pi-client-wrapper" — trust the actual file tree and Makefile, not the README's architecture diagram.
- Every service start/restart runs `pip install -r <xavier>/requirements.txt` with unpinned `>=` versions, plus a best-effort `openwakeword` pip install and a DaVoice wheel pulled with `--force-reinstall` from a third-party GitHub raw URL — a Pi can silently pick up different library versions on any restart. None of this DaVoice/openwakeword tooling is actually used by any live wake-word pipeline.
- `launch.sh` embeds its own copy of the device-auth handshake and mode-resolution (main/demo/config) logic, duplicating `xavier/app/launcher.py` — if you change mode resolution, check both places.
- `KIN_*` env vars (`KIN_STATE_DIR`, `KIN_CONFIG_CACHE_PATH`, `KIN_RUNTIME`, `KIN_MODE`) are the retired "Kin" product name baked into env var names — they still work at runtime, just say "Olympia" when talking about the product.
- `migrate-to-xavier.sh` and its Makefile target are a one-shot migration from the retired `raspberry-pi-client` repo; nothing has needed it since 5 May and no forge provisioning has happened since 18 Apr, so treat it as inert history, not an active path.
