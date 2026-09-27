<!-- blackbird:core — managed by blackbird/scripts/sync-agents. Edit blackbird/core/AGENTS.core.md, not this block. -->
# Olympia engineering rules

Every coding agent working on Olympia reads these rules: Claude Code, Codex and Cursor. The canonical copy
lives in `blackbird/core/AGENTS.core.md` and is synced into each repo; edit it there, never in a repo.
**Where another instruction file conflicts with this one (for example x-men's), this one wins.**

## The system in one screen

Olympia is a voice-first AI companion for older people. Always say "Olympia"; "Kin" is a retired name that
survives only in old code. New residents get an Android tablet (`wolverine`, Esper-managed, with a presence
sensor). Some existing residents stay permanently on a Raspberry Pi (`xavier`). The Pi is a supported
second device, not legacy to remove. Both run on one backend:

- `storm`: realtime hub and conversation lifecycle owner. `jean`: companion brain and the only manager of
  ElevenLabs agents. `beast`: post-conversation processing. `gambit`: proactive readiness gates.
- `phoenix`: main API, owner of schema migrations (`phoenix/supabase/migrations/`). `magik`: the WhatsApp
  family channel, entirely.
- `siryn`: radio. `domino`: versioned prompts, served via the `prompt_cache` table.
- Surfaces: `shadowcat` (app for residents, family and carers), `cyclops` (internal ops), `pyro` (care
  operators). `mirage` is a separate customer.
- Supabase is the system of record, and Render hosts everything. **Staging** is a full copy: its own Render
  environment, database, ElevenLabs workspace and WhatsApp number. It holds internal testers' households.
  Merges to main deploy to staging, and production is promoted at 03:00 London via `mansion`.

**Product domains are described in `iceman/domains/*.md`**, which are also searchable in the company brain
(colossus MCP). Where a page exists for the domain you're changing, read it before planning.

## Understand before you build

1. **Investigate first.** Read the real code and the real data before forming a view. For a bug, gather
   evidence before theory: what changed, who is affected, since when, and what production shows. Then prove
   one root cause before writing a fix.
2. **Model the domain, not the code's accidents.** Before adding anything, name:
   - the real-world thing being changed and its genuine states
   - the invariant that must hold
   - the single source of truth
   - which service already owns it

   Branches and flags that exist only because of past implementation choices are candidates for deletion,
   not patterns to extend.
3. **New concepts cost something.** The default budget for new tables, columns, settings, flags, services,
   workers, queues, dependencies, abstractions, caches, fallbacks and execution paths is **zero**.
   - If your plan adds a table, service/worker/queue, runtime setting or feature flag, **stop and show the
     plan to the engineer you're working with before you write code.**
   - Justify everything else on the list in the PR.

## Build

- **Extend the existing owner and path**; never stand up a parallel one. Follow the house patterns, and use
  the industry's established answer for solved problems (retries, auth, pagination, idempotency).
- **The least code that does the job.**
  - No hypothetical future requirements.
  - No fourth boolean parameter. If you're bolting on another if-else, the shape is wrong.
  - No LLM call where an if-statement or regex would do.
- **Configuration must earn its existence.** A value goes in `system_settings` or an env var only if
  operators really change it without a deploy, it really varies by environment/customer/device, or an
  experiment needs it. Otherwise it's a named constant in code. "Might be tuned someday" isn't enough.
- **One source of truth.** Remove duplicated state before building sync around it. Add a reconciler only
  when duplication is unavoidable (e.g. our DB vs ElevenLabs), and name the canonical side.
- **Prefer invariants to defensive branches.** Don't handle states that can't happen; make them impossible.
- **One path forward.** When you replace something, delete the old version in the same change: its code,
  config, flags and tests. No "keep v1 for safety".
  - A compatibility window for devices that update slowly must carry a removal date and an issue.
  - Don't do unrelated cleanup, but do remove dead code on the path you're changing once you've shown it's
    unused.
- **Schema only through migrations** in `phoenix/supabase/migrations/`, expand then contract, with the
  backfill. Never write SQL in the dashboard. Prompts live in `domino`, never hardcoded.
- **Secrets never go in git.** A committed key must be rotated even if it's deleted. Browsers and apps get
  the anon key only; phoenix derives `user_id` from the verified JWT, never from the client.
- **Intentional behaviours, don't "fix" them:** trigger windows use the user's timezone, and storm forces a
  user's previous device offline when a new one connects.

## Test and prove

- **Prove every change on staging before it touches production.** You have staging and production access.
  Use production to investigate, and to change things only once staging has proven the change.
- **Production decides what's live**, not the source code. Before you call something unused, check real
  usage: logs, rows written, callers across all repos. "grep found nothing" is not proof for routes, tables,
  settings keys, queue names or webhook URLs.
- **Tests:** add the smallest durable evidence that protects valuable behaviour. Extend an existing
  behavioural test before writing a new one. Don't test framework behaviour, trivial wrappers,
  implementation details, impossible states, or code you're deleting.
- **Green tests are not proof.** Bug fixes carry evidence of the root cause and repair the damage they
  caused (a backfill, or a stated reason none is needed). Watch it work on staging.

## Workspace and git

- **Work only in the checkout your session was given.** Claude Code, Codex and Cursor give each session its own
  worktree; in a terminal, `bb task start` does the same. Never:
  - switch branches or check out main
  - touch another checkout
  - `reset --hard`, force-push, delete branches, or merge main without being asked
- **Secrets come from Doppler, never `.env` files.** `doppler run -p <repo> -c stg -- make run` runs with staging
  secrets; use `-c prd` for production, only after staging proves the change.
- Work in a draft PR by default. Short conventional commits (`feat:`, `fix:`, `chore:`). Use each repo's
  `make` targets and its own package manager (`uv` for Python, `npm` for Node).

## Production safety

- Irreversible production actions need a clear yes from the engineer in this session. That covers dropping
  or truncating tables, deleting rows, deleting services, rotating live credentials, and bulk ElevenLabs
  agent changes. Everything else you can just do.
- **Never trigger a proactive conversation without an explicit yes.** It reaches a real resident and can't
  be undone.

## Before you say done

1. **Review your own diff** against `blackbird/standards/engineering-principles.md`. Better still, run the
   `review` skill in a fresh context.
2. **Simplify it** (the `simplify-pass` skill). Look for:
   - a branch that could disappear
   - two states that could become one
   - a setting that could be a constant
   - an old path left behind
3. **Write the PR** with the repo's template: why, domain, approach, complexity delta (what you added and
   removed), deletions, evidence, and production impact.

When you explain work to a human, use plain words. Define every codebase name the first time it matters,
and describe changes by what someone would experience.
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
