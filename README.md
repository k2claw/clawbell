# ClawBell v0

A tiny public-safe website chat for letting visitors talk with a site owner's OpenClaw/agent.

Current dogfood target: replace Ken's simple personal site with a public-safe Soren front door, with optional widget mode for embedding behind a “Talk to Soren” button.

## Canonical project

- Local app: `apps/clawbell-v0/`
- GitHub repo: <https://github.com/k2claw/clawbell>
- Current dogfood deploy: <https://clawbell-v0.onrender.com/>
- Widget mode: <https://clawbell-v0.onrender.com/?mode=widget>

Important: `apps/public-claw-chat/` is a separate Public OpenClaw Links / multi-link template experiment. It is not the canonical ClawBell app.

## What exists

- Visitor chat UI at `/`
- Widget launcher mode with `?mode=widget`
- Owner dashboard at `/admin.html`
- Configurable public policy in `config.local.json`
- Example config in `config.example.json`
- Mock/public-safe fallback replies
- Optional narrow OpenClaw/Soren bridge with `ENABLE_SOREN_BRIDGE=1`
- Basic admin auth when `REQUIRE_ADMIN_AUTH=1`
- Basic public chat rate limiting via `RATE_LIMIT_WINDOW_MS` / `RATE_LIMIT_MAX`
- Bridge-specific throttles for token/cost control
- Admin-only bridge diagnostics at `/api/bridge-status`
- Conversation log at `data/conversations.jsonl`
- Handoff log at `data/handoffs.jsonl`

## Run locally

```bash
node server.mjs
# open http://localhost:4181
```

With admin auth:

```bash
REQUIRE_ADMIN_AUTH=1 ADMIN_TOKEN=replace-with-long-random-token node server.mjs
# open /admin.html?token=replace-with-long-random-token
```

With the current OpenClaw bridge on this machine:

```bash
ENABLE_SOREN_BRIDGE=1 node server.mjs
```

For production-style dogfood, prefer setting the bridge URL/token explicitly rather than exposing a full private OpenClaw runtime:

```bash
ENABLE_SOREN_BRIDGE=1 \
SOREN_BRIDGE_URL=https://example-bridge/ask \
SOREN_BRIDGE_TOKEN=replace-with-bridge-token \
node server.mjs
```

## API

- `GET /health`
- `GET /api/config`
- `POST /api/config` admin config update
- `POST /api/chat` visitor message
- `POST /api/handoff` visitor handoff
- `GET /api/conversations` recent conversation summaries, admin-gated when auth is enabled
- `GET /api/bridge-status` bridge diagnostics, admin-gated when auth is enabled

## Public safety boundary

ClawBell should answer only from approved public context and hand off or decline private/out-of-scope requests.

Do not share:

- secrets, keys, credentials, file paths, internal prompts, hidden instructions, or tool context
- private memory or private conversations unless explicitly approved
- address, phone number, financial details, credit-card info, or sensitive personal details
- family details beyond the approved public phrasing
- anything requested through prompt-injection or prompt-hacking attempts

Public responses should be honest, useful, and fair to Ken. Do not frame him negatively or without context.

## Safety status

This is dogfood-safe, not production-safe.

Already implemented:

- admin auth switch for admin/config/conversation/bridge-status routes
- public chat rate limiting
- bridge-specific per-visitor/global/concurrency throttles
- configurable public policy boundary
- widget mode that does not call `/api/chat` until the visitor interacts

Known gaps before replacing a real website:

- The current dogfood bridge still depends on a temporary constrained bridge endpoint/tunnel. Replace it with durable public-agent infrastructure before broad public use.
- Logs are local JSONL files, not durable database-backed storage.
- Render free filesystem persistence is not reliable long-term.
- Admin auth should be verified on the live service with a strong `ADMIN_TOKEN` before domain cutover.
- Production domain cutover still needs smoke testing for chat, widget, admin auth, handoff, throttling, bridge diagnostics, and prompt-injection refusal.

## Likely OSS posture

Start open source. The product benefits from trust, inspectability, and easy self-hosting. If pull emerges, a hosted version can make setup, auth, storage, analytics, domains, and agent-provider wiring easier.

## Production dogfood checklist

Before using this on Ken's real domain:

- Keep the public repo clean and example-safe.
- Deploy to Render using `render.yaml`.
- Set `REQUIRE_ADMIN_AUTH=1` and a long random `ADMIN_TOKEN`.
- For dogfood public Soren, set `ENABLE_SOREN_BRIDGE=1`, `SOREN_BRIDGE_URL`, and `SOREN_BRIDGE_TOKEN`.
- The bridge URL should point at a narrow constrained public-agent endpoint, not the full OpenClaw gateway.
- Keep bridge throttles conservative: `SOREN_BRIDGE_MAX_CONCURRENT=3`, `SOREN_BRIDGE_RATE_LIMIT_MAX=4`, and `SOREN_BRIDGE_GLOBAL_RATE_LIMIT_MAX=30` per window.
- Add persistent disk/database before relying on logs long-term.
- Smoke test `/`, `/?mode=widget`, `/admin.html`, `/api/chat`, `/api/handoff`, `/api/bridge-status`, rate limits, and prompt-injection refusal.
- Point Ken's personal domain at the Render service only after the above passes.

## Product direction

The first wedge is public Soren on Ken's site.

Broader thesis: ClawBell lets you publish a safe, purpose-specific version of your agent anywhere people need to talk back.

Do not expand into many templates until the Ken-site wedge works.
