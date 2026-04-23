# The adapter monkey-patch

**TL;DR:** `hermes-paperclip-adapter@0.2.1` ships with two bugs that
silently break bound secrets and the Paperclip skill. Our patch is
<100 lines and is a candidate for upstream contribution. The
`hermes-auto-update` script re-applies it automatically after every
Hermes update.

## Defects

### A. Env binding corruption

The adapter reads `ctx.agent.adapterConfig.env` and copies it into the
child process env via `Object.assign(env, userEnv)`. But
`adapterConfig.env` contains *Paperclip bindings*, not strings:

```json
{
  "POSTIZ_API_KEY": {"type": "secret_ref", "secretId": "abc-123"},
  "DISCORD_TF_CONTENT_WEBHOOK": {"type": "secret_ref", "secretId": "def-456"}
}
```

`child_process.spawn` coerces env values with `String(v)`. Bindings
become the literal string `"[object Object]"`. Every bound secret
arrives at the Hermes child as that string. Any code that tries to use
it (`fetch(POSTIZ_URL, { headers: { Authorization: POSTIZ_API_KEY } })`)
either gets a 401 or, worse, silently "succeeds" by posting the string
`"[object Object]"` as a webhook payload.

How we noticed: Jet posted a LinkedIn draft with `null` in the body
field. The Postiz call looked successful. The API key was
`"[object Object]"` so auth was accepted as "missing" which Postiz
handled by silently dropping the content.

**Fix:** prefer `ctx.config.env` (Paperclip resolves bindings to strings
here via `resolveAdapterConfigForRuntime`) over
`ctx.agent.adapterConfig.env`. Copy entries defensively per-key: accept
strings as-is, extract `.value` from any stray binding dict, skip
anything else. This mirrors the canonical pattern used by
`@paperclipai/adapter-claude-local`.

### B. Missing PAPERCLIP_API_KEY injection

Paperclip mints a fresh per-run JWT and passes it as `ctx.authToken` to
the adapter. First-party adapters set
`env.PAPERCLIP_API_KEY = ctx.authToken` if no explicit key is
configured, so the child process can authenticate back to the Paperclip
API. `hermes-paperclip-adapter@0.2.1` never did this.

Effect: every call from the spawned Hermes to
`paperclip-skill issue:comment`, `paperclip-skill issue:update-status`,
etc. hung or failed with 401. Agents did their work but could not
report it.

**Fix:** after finalising env, inject
`env.PAPERCLIP_API_KEY = ctx.authToken` when
`!env.PAPERCLIP_API_KEY` (so operator-configured keys still win).

## The patch itself

See [`patches/hermes-paperclip-adapter.patch`](../patches/hermes-paperclip-adapter.patch).

Target file: `node_modules/hermes-paperclip-adapter/dist/server/execute.js`.

Apply with:
```bash
sudo patch -p0 -d / < patches/hermes-paperclip-adapter.patch
sudo node --check /home/paperclip/node_modules/hermes-paperclip-adapter/dist/server/execute.js
sudo systemctl restart paperclip
```

## Automatic re-apply

The patch is monkey-patching `node_modules`. Every `npm install` or
Hermes auto-update can clobber it. `scripts/hermes-auto-update` checks
a marker string (`ctx.authToken && !env.PAPERCLIP_API_KEY`, which does
not exist in upstream) and re-applies the patch if the marker is
missing.

This is idempotent: marker present → skip. Marker missing → back up
the current file, apply, verify.

## Upstream contribution

The patch header has the full defect writeup. The code changes are
minimal and backwards-compatible. If upstream merges the fix, we'll
drop the patch and the re-apply step from `hermes-auto-update`.

Tracking: file an issue at
<https://github.com/NousResearch/hermes-paperclip-adapter> with this
diff as the proposed fix.
