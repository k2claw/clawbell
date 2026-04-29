# ClawBell v0

A tiny public-safe website chat for letting visitors talk with a site owner's OpenClaw/agent.

Current dogfood target: replace Ken's simple personal site with a direct chat-first homepage.

## What exists

- Visitor chat UI at `/`
- Owner dashboard at `/admin.html`
- Configurable public policy in `config.local.json`
- Example config in `config.example.json`
- Mock/public-safe fallback replies
- Optional OpenClaw/Soren bridge with `ENABLE_SOREN_BRIDGE=1`
- Conversation log at `data/conversations.jsonl`
- Handoff log at `data/handoffs.jsonl`

## Run locally

```bash
node server.mjs
# open http://localhost:4181
```

With the current OpenClaw bridge on this machine:

```bash
ENABLE_SOREN_BRIDGE=1 node server.mjs
```

## API

- `GET /health`
- `GET /api/config`
- `POST /api/config` admin config update
- `POST /api/chat` visitor message
- `POST /api/handoff` visitor handoff
- `GET /api/conversations` recent conversation summaries

## Safety status

This is dogfood-safe, not production-safe.

Known gaps before replacing a real website:

- Admin auth is not implemented yet.
- Public chat rate limiting is not implemented yet.
- The current Soren bridge shells out to `openclaw agent` and should be replaced with a dedicated constrained public-agent runtime/config before broad public use.
- Logs are local JSONL files, not a database.
- Cloudflare quick tunnels are temporary and should not be used as production hosting.

## Likely OSS posture

Start open source. The product benefits from trust, inspectability, and easy self-hosting. If pull emerges, a hosted version can make setup, auth, storage, analytics, domains, and agent-provider wiring easier.

## Production dogfood checklist

Before using this on Ken's real domain:

- Create GitHub repo for ClawBell.
- Deploy to Render using `render.yaml`.
- Set `REQUIRE_ADMIN_AUTH=1` and a long random `ADMIN_TOKEN`.
- Keep `ENABLE_SOREN_BRIDGE=0` on hosted Render until a dedicated public agent bridge exists.
- Before enabling `ENABLE_SOREN_BRIDGE=1`, keep bridge throttles conservative: `SOREN_BRIDGE_MAX_CONCURRENT=1`, `SOREN_BRIDGE_RATE_LIMIT_MAX=4`, and `SOREN_BRIDGE_GLOBAL_RATE_LIMIT_MAX=30` per window. This prevents one visitor or another agent from burning through Soren/model time.
- Use fallback/mock mode or a separate webhook bridge for production.
- Add persistent disk/database before relying on logs long-term.
- Point Ken's personal domain at the Render service after smoke testing.
