# ClawBell Deploy + Auth Plan

## Goal

Make ClawBell real enough to replace Ken's live personal website, then package it as an open-source repo others can self-host.

## Recommended path

### Phase 1 — Ken deploy, private dogfood

Use a normal host with HTTPS and env vars. Good near-term options:

1. **Fly.io / Render / Railway**
   - Best fit for the current Node server.
   - Can run long-lived server process.
   - Easy env vars.
   - Can later add SQLite/Postgres.

2. **Vercel**
   - Good for static/Next-style app.
   - Current stdlib Node server would need serverless adaptation.
   - Not the fastest path from this prototype.

Recommendation: **Fly.io or Render first** for speed and fewer rewrites.

### Phase 2 — Basic production auth/safety

Minimum before putting on Ken's real domain:

- `ADMIN_TOKEN` required for:
  - `GET /admin.html`
  - `POST /api/config`
  - `GET /api/conversations`
- public chat rate limiting:
  - per-IP/session basic in-memory limit for v0
  - later durable rate limit if hosted
- hide raw logs from public routes
- no admin route linked from public chat
- persistent storage location configured by env
- clear public-safe agent prompt/config

### Phase 3 — Public OSS repo

Repo shape:

```text
clawbell/
  README.md
  DEPLOY.md
  LICENSE
  package.json
  server.mjs
  public/
    index.html
    app.js
    styles.css
    admin.html
    admin.js
    admin.css
  config.example.json
  .gitignore
```

Recommended license: MIT or Apache-2.0 for now. This is not defensible enough to hide, and adoption/trust matters more.

### Phase 4 — Hosted product only if pull emerges

Potential paid hosted value:

- no-server setup
- custom domains
- auth/admin dashboard
- hosted logs and summaries
- analytics
- rate limits/cost controls
- agent provider connectors
- team/workspace management
- premium priority queue / paid visitor routing

## Real-agent architecture

Do **not** expose full private Soren/OpenClaw context directly.

Better architecture:

- public ClawBell server receives visitor message
- server builds a narrow public-agent prompt from owner config
- server calls dedicated public agent/session/runtime
- dedicated public agent has no private tools by default
- approved handoff path can notify the owner or create a private follow-up task

Current dogfood bridge uses `openclaw agent --session-id clawbell-public-v0` and is acceptable only for local/temporary dogfood.
