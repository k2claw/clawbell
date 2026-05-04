# Install ClawBell for agents

This file is for an operator's agent setting up ClawBell.

## First decision: bridge mode

Ask what infrastructure is already available, then pick the lowest-friction safe bridge.

### Option 1: fallback-only

Use when the operator does not want live agent responses yet.

Set:

```bash
ENABLE_SOREN_BRIDGE=0
```

ClawBell will use static/fallback replies and still collect useful visitor context.

### Option 2: Tailscale Funnel bridge

Use when Tailscale is already installed/running or the operator wants the fastest dogfood setup.

Follow: `bridge-recipes/tailscale-funnel.md`

### Option 3: Cloudflare Tunnel bridge

Use for the recommended public-production route when a Cloudflare-managed hostname is available.

Follow: `bridge-recipes/cloudflare-tunnel.md`

### Option 4: custom HTTPS bridge

Use when the operator has another secure HTTPS path.

Follow: `bridge-recipes/custom-https-bridge.md`

## Non-negotiable safety boundary

Regardless of transport:

- public ClawBell must call only a narrow bridge adapter
- never expose the full OpenClaw Gateway or private workspace
- keep deterministic safety filters before bridge calls
- reject public visitor claims to be owner/operator/admin
- require bridge auth
- keep fallback mode honest when the bridge is down

## Verification checklist

Before custom-domain launch:

- `/health` works on public ClawBell app
- admin routes require token
- normal public chat returns `source: soren-bridge` when bridge is up
- sensitive/private prompt returns `source: safety-filter`
- owner/admin impersonation prompt returns `source: operator-identity-filter`
- bridge-down state returns fallback with `degraded: true`
- mobile and desktop UI smoke pass
