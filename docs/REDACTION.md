# Redaction map

Every sensitive value in this repo has been replaced by a
`${PLACEHOLDER}` token. Nothing is hashed or one-way-transformed — the
placeholders are human-readable variable names so readers can see the
*structure* without the values.

## What was redacted

| Pattern                                  | Replaced with              | Notes                                             |
|------------------------------------------|----------------------------|---------------------------------------------------|
| Company UUID                             | `${COMPANY_ID}`            | One company ID for the ThinkFraction fleet        |
| Vicious agent UUID                       | `${AGENT_ID_VICIOUS}`      | Referenced in operational scripts                 |
| Jet agent UUID                           | `${AGENT_ID_JET}`          | Referenced in content pipeline                    |
| Faye agent UUID                          | `${AGENT_ID_FAYE}`         | Referenced in drafter flow                        |
| Other agent UUIDs                        | `<uuid-redacted>`          | Placeholder for all other per-agent IDs           |
| VPS public IP                            | `${VPS_IP}`                | DigitalOcean droplet IP                           |
| Tailscale hostname `srv…….ts.net`        | `${PAPERCLIP_HOST}`        | Tailscale magic DNS name                          |
| Short hostname `srv…`                    | `${PAPERCLIP_HOSTNAME}`    | Bare hostname                                     |
| Tailnet domain `tail…….ts.net`           | `${TAILNET}`               | Tailscale tailnet                                 |
| Personal ops dir `/root/thinkfraction`   | `${THINKFRACTION_HOME}`    | Holds Discord bot, env templates                  |
| `/root/patches/`                         | `${PATCHES_DIR}`           | Holds the adapter patch                           |
| `/root/.secrets/`                        | `${SECRETS_DIR}`           | Holds 1Password service-account token             |
| `/var/log/thinkfraction/`                | `${THINKFRACTION_LOG_DIR}` | Log dir for poller etc.                           |

## What was **not** redacted (and why)

- `/home/paperclip/*` — standard system user path, not a secret. Kept
  for clarity so readers can see actual file locations.
- `/opt/hermes/*` — Hermes's documented install location.
- `http://127.0.0.1:3100` — localhost bind, documented in Paperclip's
  config. Not reachable externally.
- `postiz.thinkfraction.xyz` — public subdomain. Well-known public
  marketing surface.
- `api.github.com`, `api.apollo.io`, `api.postiz.app`, `discord.com/api/v10` —
  public third-party APIs.
- `drew@thinkfraction.xyz` — public contact address.
- Systemd unit descriptions and port numbers — architectural, not
  sensitive.

## What is **never** in this repo

- `.env` files with real values (only `.env.example` with variable
  names).
- `auth.json` files from `~/.hermes/` or `~/.paperclip/` (provider
  credential pools, run tokens).
- Paperclip's `master.key` (server-side encryption key).
- 1Password service-account token.
- Any API keys (Apollo, Postiz, Discord bot, Anthropic, MiniMax,
  DeepSeek).
- Paperclip JWT signing secret (`PAPERCLIP_AGENT_JWT_SECRET`).
- `BETTER_AUTH_SECRET`.

See [.gitignore](../.gitignore) — it belts-and-suspenders any local
checkout, so even if you drop a real `.env` or `auth.json` into this
tree it won't commit.

## Verifying before you fork

Run this scan against any checkout:

```bash
grep -rE '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' . \
  | grep -v '\${' | grep -v '<uuid-redacted>'   # expect: empty
grep -rE '\b(187\.77\.[0-9]+\.[0-9]+|100\.[0-9]+\.[0-9]+\.[0-9]+)\b' .  # expect: empty
grep -rE 'srv[0-9]+\.tail[a-z0-9]+\.ts\.net' . # expect: empty
grep -rEi '(sk-[a-zA-Z0-9]{20,}|pcp_[a-zA-Z0-9]+|Bearer [A-Za-z0-9._-]{40,})' . # expect: empty
```

If any of those return hits that aren't placeholders, flag an issue.
