# Architecture

## Processes and ports on the VPS

| Service                          | Port        | User       | What it does                                                       |
|----------------------------------|-------------|------------|--------------------------------------------------------------------|
| `paperclip.service`              | `127.0.0.1:3100` | paperclip  | Paperclip API + UI + embedded Postgres (`:54329`)            |
| `hermes-gateway.service`         | (docker)    | root       | Hermes gateway — Discord-bot-facing consciousness (separate from Paperclip fleet) |
| `hermes-hudui-gateway.service`   | `127.0.0.1:3010` | root       | Web UI for gateway Hermes instance                            |
| `hermes-hudui-paperclip.service` | `127.0.0.1:3011` | paperclip  | Web UI for paperclip-fleet Hermes instances                  |
| `thinkfraction-poller.service`   | —           | root       | Polls Paperclip → fans out activity digests to Discord             |
| `hermes-auto-update.timer`       | —           | root       | Every 6h ±30min: check upstream Hermes releases, patch + rollback  |

All paperclip-fleet Hermes processes are **short-lived**: spawned per
Paperclip heartbeat by the adapter, exit when the turn ends. The HUD at
`:3011` provides a live view of running instances and session logs.

## Filesystem layout

```
/opt/hermes/
  src/hermes-agent/          git clone (upstream, updated by timer)
  venv/                      Python 3.12 venv (bin: hermes, hermes-hudui)
  docker-compose.yml         image: hermes:<semver> (gateway)
  data/.env                  gateway env (gitignored, 1Password-sourced)

/home/paperclip/
  .paperclip/                Paperclip CLI state (unused — server runs as root)
  .hermes/
    AGENTS.md                Global agent rulebook (all profiles inherit)
    SOUL.md                  Global identity (all profiles inherit)
    config.yaml              Model routing + fallback chain
    auth.json                Provider credential pool (gitignored)
    profiles/<slug>/
      SOUL.md                Per-agent identity
      AGENTS.md              Per-agent operational rules
      auth.json              (optional) Per-profile credential override
      sessions/              Persisted conversation state
      logs/                  stderr/errors
      state.db               SQLite session state
  node_modules/hermes-paperclip-adapter/
                             npm package — MONKEY-PATCHED, see below

/root/.paperclip/instances/default/
  config.json                Paperclip server config (deploymentMode, host, port)
  db/                        Embedded Postgres data dir
  data/                      Application data
  secrets/master.key         Paperclip encryption key (gitignored, chmod 600)
  logs/                      Server logs
```

## Request flow: issue assigned → agent responds

1. User (or another agent) posts a comment on a Paperclip issue assigned
   to, say, Heero.
2. Paperclip's **native comment-wake** enqueues a heartbeat run for
   Heero (because `wakeOnAssignment: true` and the issue is assigned to
   him).
3. Paperclip spawns `hermes-paperclip-adapter` (a Node child). The
   adapter receives a `ctx` object with `ctx.agent`, `ctx.config` (env
   bindings resolved to strings), `ctx.authToken` (a per-run JWT scoped
   to this agent).
4. **[Monkey-patch region]** The adapter:
   - Copies `ctx.config.env` into the child env (strings as-is,
     binding-dicts `.value`-extracted — the patch fixes a bug where
     bindings leaked in as `"[object Object]"`).
   - Injects `env.PAPERCLIP_API_KEY = ctx.authToken` so the child
     Hermes process can authenticate back to the Paperclip API.
5. Adapter spawns `hermes -p heero` (or whatever `extraArgs` say) with
   that env.
6. Hermes loads the profile: `/home/paperclip/.hermes/profiles/heero/`.
   It reads `SOUL.md` (identity), `AGENTS.md` (rules), the `paperclip`
   skill (provides `paperclip-skill issue:read`, `paperclip-skill issue:comment`,
   `paperclip-skill issue:update-status`).
7. Hermes calls its model (`anthropic/claude-opus-4-6` for Heero; see
   [MODELS.md](MODELS.md)). Model router tries MiniMax direct first,
   falls back to DeepSeek, finally Anthropic — configured globally in
   `hermes/config.yaml`.
8. Hermes executes tool calls. When it calls `paperclip-skill
   issue:comment`, the skill uses `PAPERCLIP_API_KEY` to post back to
   `http://127.0.0.1:3100/api`.
9. Hermes exits. Paperclip marks the run complete, logs duration/tokens,
   and the issue shows the new comment.
10. If that comment triggers another wake (e.g. Jet comments on a draft
    assigned to Faye), repeat from step 1.

## Native flow vs the shadow orchestrator we burned a week on (D-147)

Previously (retired): a custom Node listener watched Paperclip's event
stream, re-drove issues, posted on behalf of agents. **This was wrong.**
It duplicated work Paperclip does natively, introduced race conditions
with the real heartbeat scheduler, and every bug was ultimately
"shadow flow colliding with native semantics."

Current (D-147, 2026-04-22): **agents do their own work via native
comment-wake.** The only out-of-band process is the activity poller,
which is read-only (Paperclip → Discord digests). If you find yourself
building a shadow orchestrator on top of Paperclip, stop. Use:

- `wakeOnAssignment: true` for event-driven wake
- Comment content + status transitions as the state machine
- Paperclip skill tool calls for all inter-agent communication

That's it. Paperclip already does scheduling, auth scoping, retry, and
observability. Don't rebuild any of them.

## Why the adapter needs a monkey-patch

The upstream `hermes-paperclip-adapter@0.2.1` has two bugs that silently
break agent execution once you bind secrets or try to use the Paperclip
skill:

**A. Env binding corruption.** The adapter copied
`ctx.agent.adapterConfig.env` verbatim into the child process env. But
those entries are *bindings*, not strings:
```json
{"POSTIZ_API_KEY": {"type": "secret_ref", "secretId": "…"}}
```
Node's `child_process.spawn` coerces each env value via `String(v)`, so
every bound secret arrived at the child as the literal string
`"[object Object]"`. Every Postiz/Discord webhook call with bound
credentials silently posted a literal `null` or crashed on auth.

**B. Missing `PAPERCLIP_API_KEY`.** Paperclip issues a fresh per-run JWT
in `ctx.authToken`. First-party adapters (like
`@paperclipai/adapter-claude-local`) wire it into the child env as
`PAPERCLIP_API_KEY` if not already set. The hermes adapter did not. So
child Hermes processes had no way to call the Paperclip API —
`paperclip-skill issue:comment` hung or failed on every run.

The patch is tiny — see
[patches/hermes-paperclip-adapter.patch](../patches/hermes-paperclip-adapter.patch).
The full defect writeup is in the patch header. Candidate for upstream.

The `hermes-auto-update` script re-applies this patch automatically
after every upstream update, guarded by a marker-grep so it's
idempotent.
