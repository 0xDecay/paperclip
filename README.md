# paperclip — ThinkFraction agent fleet (public reference)

Production config for a 14-agent AI workforce running on
[Paperclip](https://github.com/paperclipai/paperclip) orchestration and
[Hermes Agent](https://github.com/NousResearch/hermes-agent) as the
execution runtime, operated by [ThinkFraction](https://thinkfraction.xyz).

This is a **reference repository**. Every file here is the real, running
configuration — agent instruction files, systemd units, adapter patches,
model assignments — with secrets, UUIDs, and hostnames replaced by
`${PLACEHOLDER}` tokens. It exists so other operators can see how a
non-trivial multi-agent deployment is wired end-to-end.

---

## What's in here

```
company/                    Paperclip company package
  COMPANY.md                Company-level config
  paperclip.manifest.json   Agent roster, models, heartbeat, permissions
  agents/                   14 per-agent AGENTS.md files
    AGENTS.md               Fleet-wide rulebook
    <slug>/AGENTS.md        Per-agent persona + adapter config

profiles/                   Hermes per-agent profile directories
  <slug>/
    SOUL.md                 Hermes identity prompt
    AGENTS.md               (optional) Agent-specific operational rules

hermes/
  config.yaml               Model routing (MiniMax → DeepSeek → Anthropic fallback)

systemd/                    All unit files from /etc/systemd/system/
  paperclip.service         Paperclip orchestration daemon
  hermes-auto-update.*      Release-gated auto-update + timer
  hermes-gateway.service    Hermes gateway (Discord bot consciousness)
  hermes-hudui-*.service    HUD web UIs (ports 3010, 3011)
  thinkfraction-poller.*    Paperclip → Discord activity poller

scripts/
  hermes-auto-update        Full auto-update script with supervision gate,
                            patch re-apply, rollback
  op-exec                   1Password secret injection wrapper

patches/
  hermes-paperclip-adapter.patch  Monkey-patch for two adapter bugs
                                  (env binding coercion + missing authToken)

docs/                       Architecture, heartbeats, models, adapter patch
```

---

## The fleet

14 agents in a two-team structure reporting to a CEO agent:

| Agent       | Role              | Adapter       | Model                       |
|-------------|-------------------|---------------|-----------------------------|
| **Treize**  | CEO               | `hermes_local`| `MiniMax-M2.7`              |
| Misato      | PM                | `hermes_local`| `MiniMax-M2.7`              |
| Heero       | Lead Engineer     | `hermes_local`| `anthropic/claude-opus-4-6` |
| Ritsuko     | Backend Engineer  | `hermes_local`| `MiniMax-M2.7`              |
| Quatre      | Full-Stack        | `hermes_local`| `MiniMax-M2.7`              |
| Asuka       | Frontend          | `hermes_local`| `MiniMax-M2.7`              |
| Wufei       | QA                | `hermes_local`| `MiniMax-M2.7`              |
| Cid         | DevOps            | `hermes_local`| `MiniMax-M2.7`              |
| Ed          | Research Analyst  | `hermes_local`| `MiniMax-M2.7`              |
| Toji        | Financial Analyst | `hermes_local`| `MiniMax-M2.7`              |
| **Jet**     | Content Lead      | `hermes_local`| `anthropic/claude-sonnet-4-6` |
| Vicious     | Content Ideation  | `hermes_local`| `anthropic/claude-sonnet-4-6` |
| Faye        | X drafter         | `hermes_local`| `MiniMax-M2.7`              |
| **Spike**   | Outreach Lead     | `hermes_local`| `MiniMax-M2.7`              |
| tf-frasier  | Router            | `hermes_local`| `MiniMax-M2.7`              |

See [docs/MODELS.md](docs/MODELS.md) for model rationale and cost/quality
trade-offs per role.

---

## Architecture in one diagram

```
        ┌─────────────────────────────────────────────────────────────┐
        │  Paperclip server  (systemd: paperclip.service, :3100)      │
        │  - Embedded Postgres                                         │
        │  - Company registry, agent registry, issues, comments        │
        │  - Native comment-wake triggers assignee heartbeats          │
        └──────────────────────┬───────────────────────────────────────┘
                               │ spawns per heartbeat
                               ▼
        ┌─────────────────────────────────────────────────────────────┐
        │  hermes-paperclip-adapter (npm, MIT, NousResearch)          │
        │  + local monkey-patch (see patches/)                         │
        │  - Resolves adapter env bindings                             │
        │  - Injects PAPERCLIP_API_KEY from run-scoped JWT             │
        └──────────────────────┬───────────────────────────────────────┘
                               │ child_process.spawn("hermes", …)
                               ▼
        ┌─────────────────────────────────────────────────────────────┐
        │  Hermes Agent v0.10 (Python, /opt/hermes/venv)              │
        │  - Loads profile: /home/paperclip/.hermes/profiles/<slug>/  │
        │  - Model router: MiniMax → DeepSeek → Anthropic (fallback)  │
        │  - Executes per SOUL.md + AGENTS.md + Paperclip skill       │
        └──────────────────────┬───────────────────────────────────────┘
                               │ posts comments, updates status
                               ▼
                         back to Paperclip issue
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full flow,
including the native comment-wake pattern that replaced an earlier
custom "Publisher Listener" shadow orchestrator (D-147 lesson).

---

## Why this repo exists

**For operators:** copy-paste-grade reference for wiring Paperclip + Hermes
in production. The hard lessons (heartbeat cost, adapter bugs, native vs
shadow orchestration) are encoded in systemd units, the patch, and the
docs — not just words.

**For ourselves:** source of truth for the ThinkFraction fleet. Every
change to models, heartbeats, or agent instructions ships through this
repo first, VPS second.

---

## Upstream projects

- **Paperclip** — <https://github.com/paperclipai/paperclip> (MIT)
- **Hermes Agent** — <https://github.com/NousResearch/hermes-agent> (MIT)
- **hermes-paperclip-adapter** — <https://github.com/NousResearch/hermes-paperclip-adapter> (MIT)

Our monkey-patch ([patches/hermes-paperclip-adapter.patch](patches/hermes-paperclip-adapter.patch))
is a candidate for upstream contribution — see header comments for the
defect analysis.

---

## Reproducing a comparable deployment

High-level:

1. Provision a VPS (we use DigitalOcean), expose `:3100` privately via Tailscale.
2. Install Node 20, Python 3.12, Docker, Hermes venv (`/opt/hermes/venv`).
3. `npx paperclipai onboard` → creates `/root/.paperclip/instances/default/`.
4. `npx companies.sh add` the company package in [company/](company/) (strip placeholders first).
5. Patch adapter: `patch -p0 -d / < patches/hermes-paperclip-adapter.patch`.
6. Bind credentials into agent `adapterConfig.env` as `secret_ref` items
   (we source everything from 1Password via `scripts/op-exec`).
7. Enable systemd units: `systemctl enable --now paperclip hermes-gateway
   hermes-hudui-gateway hermes-hudui-paperclip hermes-auto-update.timer`.
8. Set heartbeats to `wakeOnAssignment: true, intervalSec: 0`
   (see [docs/HEARTBEATS.md](docs/HEARTBEATS.md) — this alone saves 40×
   on token spend vs interval polling).

---

## Security posture

- No secret values are in this repo. Every credential is either (a)
  sourced at runtime from 1Password via `op-exec`, (b) a `secret_ref`
  binding resolved by Paperclip, or (c) a systemd `EnvironmentFile=`
  pointing to a root-owned file outside the repo.
- All UUIDs (company, agent) redacted to named placeholders.
- Hostnames/IPs redacted to `${PAPERCLIP_HOST}`, `${VPS_IP}`, etc.
- See [docs/REDACTION.md](docs/REDACTION.md) for the full map.

---

## License

MIT. See [LICENSE](LICENSE).

Upstream code (Paperclip, Hermes, the adapter) is separately MIT-licensed
by their respective authors.
