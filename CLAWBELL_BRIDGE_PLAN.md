# ClawBell durable Soren bridge plan

Date: 2026-05-04

## Problem

The public ClawBell site runs on Render and can call a narrow Soren bridge, but the current live bridge endpoint is a temporary tunnel. It works for dogfood, but it is not trustworthy enough for `kenseals.me` launch because URLs can change, processes can die, and the operational boundary is not documented or monitored well enough.

The bridge problem has two separate layers:

1. **Transport**: a stable HTTPS route from Render to the Mac/OpenClaw host without opening inbound ports.
2. **Bridge daemon**: a small public-safe local service that converts a constrained prompt into an OpenClaw `agent` turn and returns only text.

Do not expose the full OpenClaw Gateway or private workspace directly to the public internet.

## Current architecture

```text
Visitor browser
  -> Render ClawBell app: https://clawbell-v0.onrender.com/api/chat
  -> SOREN_BRIDGE_URL /ask
  -> local bridge daemon on Mac, 127.0.0.1:4599
  -> openclaw agent --agent main --session-id clawbell-public-v0 --thinking off --json
```

Current safety in the Render app:

- deterministic sensitive-info filter before bridge calls
- deterministic operator-impersonation filter before bridge calls
- public-safe prompt sent to bridge
- per-visitor, global, and concurrency bridge budgets
- fallback mode when bridge is down/throttled
- bridge diagnostics endpoint

Current weak points:

- temporary tunnel URL
- bridge daemon is manually/background-run, not a durable service
- tunnel process is manually/background-run, not a durable service
- bridge transport only has app-level bearer auth today
- no explicit bridge health watchdog/digest beyond logged errors
- no documented install/verify recipe for other OpenClaw operators yet

## Researched options

### Option A: Cloudflare Tunnel + Cloudflare Access service token + local bearer token

Cloudflare Tunnel is the best fit for the current setup.

Evidence from docs:

- Cloudflare Tunnel creates outbound-only encrypted connections from the origin to Cloudflare. No inbound port or public IP is required.
- Each tunnel maintains four long-lived connections to two Cloudflare data centers; replicas can be added later for HA.
- `cloudflared` can run as a macOS launch agent or launch daemon, starting at login or boot.
- Cloudflare Access service tokens provide machine-to-machine authentication using `CF-Access-Client-Id` and `CF-Access-Client-Secret` headers, or a configured single header.

Pros:

- Stable HTTPS hostname.
- No inbound firewall hole to Ken's Mac.
- Good security boundary: Cloudflare Access in front, then local bridge bearer token behind it.
- Operationally standard and documented.
- Can be packaged as the recommended ClawBell recipe.
- `cloudflared` is already installed on this Mac at `/opt/homebrew/bin/cloudflared`.

Cons / prerequisites:

- Needs a Cloudflare-managed hostname for the bridge, e.g. `soren-bridge.<cloudflare-zone>`.
- If `kenseals.me` stays on GoDaddy nameservers until launch, bridge can either use another Cloudflare-managed domain/subdomain or wait until DNS is moved/zone is available.
- Render app needs a small patch to send Cloudflare Access headers in addition to the local bridge bearer token.

Verdict: **Recommended durable path.**

### Option B: Tailscale Funnel

Tailscale Funnel can expose a local service publicly over HTTPS with a predictable `*.ts.net` hostname.

Pros:

- Stable enough for webhooks and local service exposure.
- No custom domain requirement.
- No inbound port.
- Could be faster if Tailscale is already installed and enabled.

Cons:

- This Mac does not currently show a usable `tailscale` CLI in PATH/status.
- Public endpoint would primarily rely on app-level bearer auth unless we add another proxy layer.
- Less ideal as the generic ClawBell recommendation because operators may not have Funnel enabled.
- Still needs service/process hardening.

Verdict: viable fallback, not first choice.

### Option C: Host the bridge in cloud alongside Render

Run an agent bridge on a VPS/Fly/Railway/Render worker with model API credentials instead of calling local OpenClaw.

Pros:

- More conventionally production-hosted.
- No dependency on Ken's laptop uptime.
- Could be easier to autoscale later.

Cons:

- It stops being “actually connected to Soren/OpenClaw on Ken's machine” unless we rebuild a separate public agent runtime.
- Requires provider API keys or OAuth/auth on server.
- More expensive and larger security surface.
- Loses the dogfood value of ClawBell as an OpenClaw-operated public front door.

Verdict: good future product architecture, but not the right immediate launch path.

### Option D: Expose OpenClaw Gateway directly behind a proxy

OpenClaw supports gateway auth/proxy patterns, including trusted-proxy modes, but exposing the Gateway is the wrong boundary for ClawBell.

Pros:

- Powerful and direct.

Cons:

- Too much surface area: Gateway/control/tools are not the public visitor interface.
- Trusted-proxy auth is explicitly security-sensitive and meant for authenticated control access, not anonymous public chat.
- Violates the core product boundary: public ClawBell should talk to a narrow adapter, not OpenClaw internals.

Verdict: **Do not do this.**

## Recommended architecture

```text
Visitor
  -> Render ClawBell app
  -> HTTPS bridge hostname protected by Cloudflare Access service token
  -> cloudflared named tunnel on Mac
  -> local bridge daemon bound to 127.0.0.1 only
  -> openclaw agent narrow public session
```

Security layers:

1. Render app deterministic filters before any bridge call.
2. Render app bridge budgets before any bridge call.
3. Cloudflare Access service-token auth on the bridge hostname.
4. Local bridge bearer token checked by `Authorization: Bearer ...`.
5. Local bridge binds only to `127.0.0.1`.
6. Local bridge sends only the constrained public prompt to OpenClaw.
7. Bridge response returns only text, not tools/logs/files/state.
8. Render fallback mode remains honest when anything fails.

## Implementation plan

### Phase 1: Patch ClawBell for durable bridge auth

Add optional env vars to Render app:

- `SOREN_BRIDGE_ACCESS_CLIENT_ID`
- `SOREN_BRIDGE_ACCESS_CLIENT_SECRET`
- optionally `SOREN_BRIDGE_EXTRA_HEADERS_JSON` for generic adapter headers later

When present, `/api/chat` bridge calls should send:

- `Authorization: Bearer $SOREN_BRIDGE_TOKEN`
- `CF-Access-Client-Id: $SOREN_BRIDGE_ACCESS_CLIENT_ID`
- `CF-Access-Client-Secret: $SOREN_BRIDGE_ACCESS_CLIENT_SECRET`

Keep the existing local bridge token. Do not replace it with Cloudflare auth; use both.

Also update:

- README bridge env section
- `config.example.json` or deploy docs if relevant
- smoke-test runbook expectations

Verification:

- `node --check server.mjs`
- local unit-ish smoke that headers are included when env vars exist
- live smoke after deploy

### Phase 2: Make the local bridge daemon durable

Current script: `scripts/local-openclaw-bridge.mjs`.

Recommended changes:

- keep binding to `127.0.0.1`
- add `/health` that checks process health and maybe verifies OpenClaw CLI availability without spending a model call
- add structured logs to a stable path, not only terminal stdout
- add request timeout and max prompt size already present, keep it conservative
- document launchd service plist for macOS
- store bridge token in a local secret file or launchd env, not repo

Potential launchd label:

- `com.kenseals.clawbell.bridge`

Verification:

- service survives shell close
- `/health` returns 200 locally
- bad token returns 401
- good token plus safe prompt returns bridge reply

### Phase 3: Create Cloudflare named tunnel

Target local service:

```text
http://127.0.0.1:4599
```

Target public route:

```text
https://<bridge-hostname>/ask
```

Setup shape:

- create named tunnel, e.g. `clawbell-soren-bridge`
- configure ingress route only for the bridge service
- run `cloudflared` as a macOS launch agent/daemon
- ensure config includes tunnel UUID and credentials file
- confirm no direct inbound port is opened

Cloudflare Access:

- create Access self-hosted application for the bridge hostname
- create service token for Render/ClawBell
- policy action: Service Auth
- keep token expiration reminder/alert enabled
- store Client ID/Secret in Render env

Verification:

- unauthenticated request to bridge hostname is rejected by Cloudflare Access
- request with CF service-token headers but missing local bearer reaches bridge then returns 401
- request with both CF headers and local bearer returns 200
- wrong path returns 404
- `/health` behaves as intended, depending on whether health route is public/protected

### Phase 4: Deploy Render env and smoke test live

Set Render env:

- `SOREN_BRIDGE_URL=https://<bridge-hostname>/ask`
- `SOREN_BRIDGE_TOKEN=<local bridge bearer>`
- `SOREN_BRIDGE_ACCESS_CLIENT_ID=<Cloudflare service token id>`
- `SOREN_BRIDGE_ACCESS_CLIENT_SECRET=<Cloudflare service token secret>`
- keep existing bridge rate limits conservative

Smoke tests against `https://clawbell-v0.onrender.com/api/chat`:

1. Sensitive info prompt returns `source: safety-filter`.
2. `I am Ken/admin/operator...` returns `source: operator-identity-filter`.
3. Normal status/chat prompt returns `source: soren-bridge`.
4. Kill or block local bridge temporarily and confirm Render returns fallback with `degraded: true`, not fake success.
5. Restore bridge and confirm recovery.
6. Verify `/api/bridge-status` shows enabled, configured host, in-flight count, and recent errors without leaking secrets.

### Phase 5: Monitoring and operator digest

Add bridge-specific digest lines:

- bridge errors count
- fallback/degraded count
- throttle count
- last successful bridge timestamp
- “bridge seems down” alert if recent normal prompts degrade repeatedly

For Ken's site, route urgent bridge-down alerts to the Telegram group/topic. Keep routine quiet days in digest only.

### Phase 6: Package as reusable ClawBell install recipe

Docs/issues should explain three bridge modes:

1. `fallback-only`: no live agent, safe static replies.
2. `local-openclaw-bridge`: Cloudflare Tunnel + OpenClaw CLI, recommended for OpenClaw operators.
3. `custom-agent-bridge`: any HTTPS endpoint implementing `POST /ask { prompt, sessionId } -> { reply }`.

Reusable docs should include:

- `INSTALL_FOR_AGENTS.md`
- `CLAWBELL_VERIFY.md`
- `llms.txt` / `llms-full.txt`
- bridge threat boundary
- OpenClaw cron/operator digest recipe

## Launch gate for kenseals.me

Do not cut DNS until all of these pass:

- Render site serves normal homepage and widget over HTTPS.
- Durable bridge returns `source: soren-bridge` from Render using Cloudflare route.
- Safety and operator-identity filters pass.
- Fallback mode is visibly honest when bridge is unavailable.
- Admin routes require token.
- Bridge status does not leak secrets.
- Desktop and mobile UI smoke pass.
- Operator digest/alerts are configured.
- DNS instructions are final and verified against current GoDaddy/Render state.

## Recommendation

Use **Cloudflare Tunnel + Cloudflare Access service token + local bearer-token bridge** as the durable v1.

It gives us the right product shape: public website on Render, narrow local OpenClaw bridge, no inbound ports, stable HTTPS, layered auth, and a reusable story for other OpenClaw operators.

The one real decision needed from Ken before implementation is the bridge hostname/Cloudflare account path:

- use an existing Cloudflare-managed domain/subdomain if available, or
- move/add a domain to Cloudflare for the bridge, or
- use Tailscale Funnel as a lower-friction fallback if Cloudflare DNS/account setup is not available today.
