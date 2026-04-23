# Heartbeats: the #1 cost mistake in Paperclip deployments

**TL;DR**: never use interval-based heartbeats on a multi-agent fleet.
Set `wakeOnAssignment: true` and `intervalSec: 0`. You save 40× on
tokens without losing responsiveness.

## The trap

Paperclip's heartbeat config has four knobs:

```json
{
  "heartbeat": {
    "enabled": true,
    "wakeOnAssignment": true,
    "intervalSec": 0,
    "cooldownSec": 30
  }
}
```

- `enabled: true` — heartbeat system is on.
- `wakeOnAssignment: true` — **event-driven**. Agent wakes the moment a
  task is assigned to it or a comment lands on one of its issues.
- `intervalSec: 0` — **no polling**. Disable time-based wake.
- `cooldownSec: 30` — debounce rapid assignments.

The cheap trap: someone sets `intervalSec: 30` "just to make sure
nothing gets stuck." Now every agent runs a full Claude/Hermes session
every 30 seconds — even when there's no work. A full session is
~3,000-5,000 tokens minimum (context load + system prompt).

## Math

Interval polling on an idle fleet:

```
31 agents × 2 heartbeats/min × 60 min × 24 hours × 10,000 tokens (Opus)
= 892.8M tokens/day
```

That's real money. (It's also what GitHub issue #906 on the Paperclip
repo is about — a 31-agent fleet was burning $1,500/day on heartbeat
overhead before wake-on-assignment existed.)

Event-driven on the same fleet doing the same real work:

```
31 agents × 10 actual tasks/day × 3,000 tokens (wake + one turn)
= 930K tokens/day
```

**960× reduction.** And you lose nothing — `wakeOnAssignment` is as
responsive as a 1-second poll, because it's event-driven.

## Our config

See [company/paperclip.manifest.json](../company/paperclip.manifest.json).
Every agent is either:

- `heartbeat.enabled: false` — for ICs that only respond when Misato
  (PM) assigns them work. They wake via native comment-wake on
  assignment, nothing else. Zero idle cost.
- `heartbeat.enabled: true, cooldownSec: 600, intervalSec: 86400` —
  for leads (Jet, Spike) who benefit from a daily "idle check" in case
  they want to self-initiate. The daily interval is effectively a
  dead-man's-switch, not a polling mechanism.

No agent polls more than once a day. Most don't poll at all.

## When a daily interval IS justified

Three conditions, all must hold:
- The agent has a recurring self-initiated responsibility (e.g. "check
  competitor signals every morning").
- You've benchmarked the actual cost — `cooldownSec × intervalSec`
  gates interact non-obviously.
- You're on a model that makes the cost trivial (MiniMax, DeepSeek,
  Haiku — not Opus).

If all three don't hold, use `enabled: false` and let assignment drive.

## If you're migrating from a polling setup

1. Dump current config: `npx paperclipai agent list --company-id "$CID" --json`.
2. For each agent, `PATCH /api/agents/<id>`:
   ```json
   {
     "runtimeConfig": {
       "heartbeat": {
         "enabled": true,
         "wakeOnAssignment": true,
         "intervalSec": 0,
         "cooldownSec": 30
       }
     }
   }
   ```
3. Watch token spend for 24h. If an agent stops responding, it means
   your workflow was secretly relying on polling to trigger it — fix the
   workflow (add an explicit assignment or comment), don't re-enable
   polling.

## Known upstream issue

- [paperclipai/paperclip#906](https://github.com/paperclipai/paperclip/issues/906) —
  Role-tiered heartbeat skill, not yet landed. Until it does,
  `wakeOnAssignment + intervalSec: 0` is the workaround.
