# GORDON SESSION CONTEXT — Docker Media Stack (Condensed)

*Paste this entire file and say "Here's my context, get up to speed" to resume instantly.*

**Last updated:** 2026-08-14 (post dashboard-hardening session — Portainer card removed, security/PWA fixes staged, not yet deployed)

---

## 🚫 DO NOT TOUCH

- **Image version pinning** — services run `:latest`, deferred indefinitely, not wanted.
- **AIOMetadata's `/configure` Access gate** — confirmed gated (2026-08-14). Leave alone unless explicitly asked.
- **Competing cloudflared LaunchAgent/LaunchDaemon processes** — resolved, no longer tracked, don't touch.
- **Passwords** — never change any existing login/admin password.
- **Persistent data volumes** — never delete: `aiostreams_data`, `aiometadata_data`, `redis_data`, `openposterdb_data`, `portainer_data`, `dashboard_data` (Compose-prefixed as `combined-stack_<n>`).
- **Archived files/volumes** (`/Users/rj/zilean-stack/`, `/Users/rj/combined-stack/backups/`) without explicit approval.
- **Redis config** — no `--lazyfree-*` flags (fatal on this build), no RDB save enable. Confirmed healthy — leave as-is.
- **Cloudflare Access/Zero Trust config** — never via API/CLI; user manages manually in dashboard.
- **`claude_MASTER_SESSION_CONTEXT.md`** — flagged stale (describes old 7-service stack), not yet reconciled/archived.

---

## 🔴 CRITICAL RULES

1. Never reset/delete/overwrite databases, configs, or dashboard settings for AIOStreams, AIOMetadata, or OpenPosterDB.
2. Universal password `23700119rlJ` — exact string, unchanged, everywhere (Portainer admin is self-created, exempt).
3. Never delete persistent data volumes (see above).
4. Always list proposed changes and get explicit approval before touching stored data/config — this applies to resource limits too, not just data/secrets.
5. Before any risky config change, back up first (`.env`, `aiometadata.env`, `docker-compose.yml`, `~/.cloudflared/config.yml`) — timestamped copies to `/Users/rj/backups/`.
6. Client-facing URLs (Stremio/Nuvio manifests, APIs) must use public hostnames, never internal Docker service names.
7. Only restart/recreate the specific container(s) involved in a change — never restart unrelated services.
8. Never configure Cloudflare Access via API/CLI — state steps in plain language, user executes manually.
9. Don't trust stale numbers in this doc at face value — re-audit with `docker stats` / `redis-cli info` before acting on a documented number if it's driving a decision.
10. This file itself can be lost — treat it as disposable/regenerable, not a source of truth on its own; if in doubt, re-audit the live stack.
11. **Dashboard `index.html`/`manifest.json`/icons are hand-authored, not auto-generated** — any future edits should go through the same audit-then-edit workflow, not ad hoc live changes.

---

## 🟢 PRIORITIES (in order) — real audit numbers as of 2026-08-14 (memory), file state as of this session (dashboard)

| # | Service | Role | Memory (limit/reserved) | Notes |
|---|---|---|---|---|
| **1** | AIOStreams | Stream discovery addon | 1.3G / 950M | Top priority, max performance. Real usage ~649MiB (48.75%) — comfortable. |
| **2** | AIOMetadata | Metadata enrichment | 950M / 700M | Second priority. Real usage ~785.5MiB (82.69%) — high, watch, out of scope. |
| Support | Redis (aiometadata-redis) | Cache for #1 & #2 | 750M / 450M | Disposable, no durability. Healthy: 0 evictions, frag ratio 1.02. |
| Support | Portainer | Docker mgmt UI | 128M / 64M | Distroless, no healthcheck possible. Real usage ~23MiB. No longer linked from the dashboard UI (see below) — still reachable directly at its own URL, still Access-gated. |
| Support | OpenPosterDB | Poster/rating artwork | 300M / 150M | Real usage was 133MiB/150M (near OOM) before the bump. **Not Access-gated (by design)** — its link is public on the dashboard; lowest-friction target if `237119.xyz` gets found by a scanner/crawler. |
| Support | **Dashboard** | Static landing page linking to media services | 64M / 32M | nginx:alpine. Deployment method changed this session — see Key Tuning. Real usage trivial (~5-10MB). **Public, live, hardened this session (fixes staged, deploy pending).** |
| Support | Portainer Agent | Enables Portainer's volume Browse/upload UI | 128M / 64M | `portainer_agent` container; unlocked the ability to push files (dashboard assets, etc.) straight into a volume without console-paste or base64 tricks. |

**Hardware ceiling:** MacBook Air M1, 8GB RAM. Docker Desktop usable: 3.74GB (hard cap). Swap: 2GB confirmed. **Current stack total limits: ~3.62GB (~120MB headroom).** Re-audit before adding anything else — not re-checked this session.

---

## 📊 ARCHITECTURE (what's running)

**Location:** `/Users/rj/combined-stack/docker-compose.yml` (+ `.env`, `aiometadata.env`)
**Network:** `addon-network` bridge, `172.25.0.0/16`
**Remote access:** `237119.xyz` via Cloudflare Tunnel (`home-stack`), per-service subdomains
**Client:** Nuvio (Stremio-compatible) + Real-Debrid

| Service | Container | IP | Port | Subdomain |
|---|---|---|---|---|
| AIOStreams | aiostreams | 172.25.0.40 | 2370 | rstream.237119.xyz |
| AIOMetadata | aiometadata | 172.25.0.50 | 3232 | rmeta.237119.xyz |
| Redis | aiometadata-redis | 172.25.0.60 | 6379 (internal) | — |
| OpenPosterDB | openposterdb | 172.25.0.80 | 3000 | rposter.237119.xyz |
| Portainer | portainer | 172.25.0.90 | 9000 | radmin.237119.xyz (whole host Access-gated) |
| Dashboard | dashboard | *(not on addon-network)* | 8090 | 237119.xyz — LIVE, routed through the Cloudflare Tunnel |

**Portainer Agent:** container `portainer_agent`, port `9001`, host bind-mounts of `/var/run/docker.sock` and `/var/lib/docker/volumes`. Registered as Portainer environment "local-agent" at `host.docker.internal:9001`. Same Docker host as "local", not a separate machine.

All services running/healthy as of last check (2026-08-14). Portainer has no CMD healthcheck (distroless) — verify externally via `docker ps` status.

**Access (Zero Trust) status:** AIOStreams ✅, AIOMetadata ✅, Portainer ✅ (whole host), OpenPosterDB ⏳ not protected (by design). **Dashboard is NOT behind Access** — mitigated this session with a `noindex, nofollow` meta tag (keeps it out of search engines) but that's obscurity, not access control. Decision on whether to Access-gate the dashboard itself is still open (see TODOs).

---

## 🔐 KEY CREDENTIALS

- Universal password: `23700119rlJ`
- AIOStreams Auth: `rj2113:23700119rlJ` | Secret Key: `C8436F7023E2C4F3B13F365BE5167E7744E1F0B94FFD57C23F99CA49617576D4`
- TMDB: `e611f825981ed96c5b6564b3e08fa50d` | TVDB: `c326a3dc-e9db-4349-970d-a5b87f0489f0`
- OpenPosterDB JWT: `b4b1a9d7a6a96ab6e5e6f99b33021126eeecf96d96d35c7e74f0eea85f9de449` | admin login `rj2113`/`23700119rlJ` | API key for AIOMetadata: `6e137c519e87c0216002828242fd7dfa4bd63f3a5628655b0af322b884f9bc90`
- MDBList: `ue53byb3rdcwlqsxg5h8tmx8f` | Fanart.tv: `5a133ca31e1dd9ed7f90134e1eaf47e7`
- Portainer admin: self-created via setup token, not tracked here
- Dashboard: no credentials — public static page, no login. The old client-side "passphrase unlock" for a Portainer card has been **removed entirely** (see Recently Completed) — it was never real security (Access already gated the real Portainer URL), and it exposed a crackable SHA-256 hash in page source.

---

## 🛠️ KEY TUNING

- **AIOStreams:** `NODE_OPTIONS=--max-old-space-size=1000 --expose-gc --trace-warnings`. `BASE_URL=https://rstream.237119.xyz`. Healthcheck `/api/v1/health`. Never add undocumented env vars; never quote `NODE_OPTIONS`.
- **AIOMetadata:** `NODE_OPTIONS=--max-old-space-size=480 --expose-gc`. `HOST_NAME=rmeta.237119.xyz` (domain-only). Healthcheck `/health`. Depends on Redis.
- **Redis:** `--maxmemory 600mb --maxmemory-policy allkeys-lru`, RDB disabled. Never add `--lazyfree-*` flags.
- **OpenPosterDB:** `COOKIE_SECURE=false` required for admin login. Healthcheck `/api/auth/status`.
- **Portainer:** Docker socket R/W mount required. No healthcheck (distroless).
- **Dashboard — deployment method changed this session:** the old approach (base64-embedding `index.html` into the Portainer stack's `command:`, decoded on every container start) has been **retired** now that the Portainer Agent unlocks Volumes → Browse/upload. Files now go straight into the `dashboard_data` volume as real, persistent files:
  - `index.html` → web root
  - `manifest.json` → web root
  - `assets/icon-192.png`, `assets/icon-512.png`, `assets/icon-512-maskable.png`, `assets/apple-touch-icon.png` → `assets/` subfolder
  - **This session's updated versions of all these files exist locally (delivered in-chat) but have NOT yet been uploaded to the live volume — see Outstanding TODOs.**
  - To update in future: edit the file, re-upload via Portainer's volume Browse UI. No more base64/stack-editor dance needed.
- **Portainer Agent:** image `portainer/agent:latest`, port `9001:9001`, bind-mounts `/var/run/docker.sock` and `/var/lib/docker/volumes`. No app-level config.

---

## 💾 BACKUPS

- **Configs (manual, before any risky change):** `.env`, `docker-compose.yml`, `aiometadata.env`, `~/.cloudflared/config.yml` → timestamped copies in `/Users/rj/backups/`.
- **Volumes:** automated weekly (Sundays 3 AM, launchd, no sudo) for `aiostreams_data`, `aiometadata_data`, `openposterdb_data`, `portainer_data`. Retention: last 4 per volume. Logged to `backup_volumes.log`. **`dashboard_data` is NOT in this rotation.**
- ⚠️ **This note needs re-deciding:** `dashboard_data` used to be safely excluded from backups because its content was disposably regenerated from a base64 blob on every restart. That's no longer true — the volume now holds real, hand-authored files (`index.html`, `manifest.json`, icons) with no blob to regenerate from. If the volume were lost today, that work would need to be redone from scratch (or from whatever copies you kept from this chat). Worth adding `dashboard_data` to the weekly rotation, or at minimum keeping local copies of the deployed files outside Docker.
- **This context markdown file is not covered by any backup** — it lives outside Docker volumes and was lost once before. Consider a manual copy to `/Users/rj/backups/`.

---

## ⚡ QUICK COMMANDS

```bash
cd /Users/rj/combined-stack && docker compose up -d
cd /Users/rj/combined-stack && docker compose restart aiostreams   # isolate to one service
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker stats --no-stream --all

curl http://localhost:2370/api/v1/health       # AIOStreams
curl http://localhost:3232/health              # AIOMetadata
curl http://localhost:3000/api/auth/status     # OpenPosterDB
curl http://localhost:8090/                    # Dashboard (should return the HTML)
docker exec aiometadata-redis redis-cli ping   # Redis

# Manual config backup before a change
TS=$(date +%Y%m%d_%H%M%S)
cp /Users/rj/combined-stack/docker-compose.yml /Users/rj/backups/docker-compose_backup_${TS}.yml

# Manual volume backup run (also prunes per volume to last 4)
bash /Users/rj/backups/backup_volumes.sh

ps aux | grep cloudflared | grep -v grep
sudo launchctl list | grep cloudflare   # user runs manually, not Gordon
```

---

## 🚨 KEY GOTCHAS

1. AIOStreams healthcheck is `/api/v1/health`, not `/healthcheck`.
2. AIOMetadata `HOST_NAME` must be domain-only — protocol prefix causes redirect loop.
3. Redis `--lazyfree-*` flags crash this build; RDB intentionally disabled.
4. AIOStreams crashes on undocumented env vars or quoted `NODE_OPTIONS`.
5. OpenPosterDB needs `COOKIE_SECURE=false` or admin login silently fails.
6. Docker-internal service names only resolve container-to-container — client URLs need public hostname.
7. `cloudflared` config changes need a service restart, not live-reload.
8. Gordon cannot run `sudo` (no TTY) — tunnel/launchctl restarts needing sudo are manual, user-only.
9. Portainer setup token regenerates every restart if admin account doesn't exist.
10. Distroless images (Portainer) have no shell/curl/wget — check externally via `docker ps`.
11. Always back up before touching config/keys/resource limits.
12. AIOStreams' `max-old-space-size` must stay well below the container memory limit (~20-25% headroom).
13. Compose volumes are project-prefixed (`combined-stack_aiostreams_data`) — backup scripts must use the prefixed name.
14. Documented "real usage" numbers can go stale fast — always re-audit before acting on one.
15. Portainer's built-in Console (exec terminal) frequently fails to accept paste — especially from mobile browsers.
16. **(Superseded for the dashboard, may still apply elsewhere)** Base64-encoding a file and embedding it in a stack's `command:` was the old workaround for getting content into a container without console paste. The dashboard no longer needs this now that the Agent's volume Browse/upload works — but this trick is still worth remembering for any *other* service without Agent-enabled volume access.
17. Portainer's volume Browse/upload button requires the Portainer Agent — fixed by deploying `portainer_agent` and adding it as a second environment.
18. A Portainer "Stack" is tied to the environment it was deployed through — keep editing existing stacks (`dashboard`, `portainer-agent`) from "local" (wherever they were originally deployed).
19. **A client-side passphrase gate is not real access control if the hash and the "hidden" URL both ship in the public page source** — anyone can `atob()` or hash-crack it from devtools. Real protection has to come from something the client can't read, like Cloudflare Access. (This is why the Portainer card's unlock gate was removed rather than "fixed" — it was providing a false sense of security.)
20. This context file can be deleted/lost — rebuild from stored memory + a fresh `docker stats`/compose read if needed.

---

## ✅ RECENTLY COMPLETED

**Earlier sessions:** AIOStreams memory fix, weekly volume backups with retention, full resource audit and reallocation, confirmed AIOMetadata Access gate, deployed dashboard + Portainer Agent, routed dashboard through the tunnel (live at `237119.xyz`).

**This session (dashboard security/UX audit + fixes):**
1. Full code audit of `index.html` — found the dashboard was exposing all subdomain names in plain page source (ties to the still-open TODO about whether the dashboard needs Access), and that the Portainer card's "unlock" gate (base64-encoded URL + SHA-256 passphrase hash) was security theater — both were readable/crackable straight from the public HTML.
2. **Removed the Portainer card entirely** (from HTML, CSS, and JS) — one fewer access point exposed on a now-public page, as requested. Portainer itself is untouched and still reachable directly, still Access-gated.
3. Added `<meta name="robots" content="noindex, nofollow">` so crawlers/search engines stop indexing the subdomain list.
4. Converted the three remaining service cards from JS-driven `<div data-href>` elements to real `<a href>` tags — restores native middle-click / right-click "open in new tab" / copy-link, which the old pattern silently broke.
5. Fixed the "Dashboard last updated" footer, which was reading `document.lastModified` and therefore reset on every container restart regardless of whether content changed. Now a manually-maintained `DASHBOARD_UPDATED` date constant in the script — bump it by hand on real redeploys.
6. Made the live status dots' tooltips accurate: they now say "network reachable" instead of implying full health, since a `no-cors` fetch can't see through an Access-gate redirect or a real error response.
7. Fixed heading hierarchy — card titles were `<h3>`, siblings of the `<h3>` group labels they're nested under; now `<h4>`, correctly one level deeper.
8. Built full PWA support: authored `manifest.json` (name, icons, standalone display, theme colors) and generated a matching icon set (192×192, 512×512, 512×512 maskable with proper safe-zone padding, and a 180×180 Apple touch icon) from the existing favicon design.
9. **Deliverables produced and handed off in-chat, NOT yet deployed:** updated `index.html`, `manifest.json`, and 4 PNG icons.

---

## 📋 OUTSTANDING TODOs / NEXT STEPS

1. 🔴 **Deploy this session's files.** Upload the updated `index.html`, `manifest.json`, and `assets/*.png` into the `dashboard_data` volume via Portainer's Browse/upload UI (web root for the first two, an `assets/` subfolder for the icons). Nothing has gone live yet — the container is still serving the old version with the Portainer card and old JS.
2. 🟡 **Verify after deploying:** dashboard loads clean, all 3 service links + keyboard shortcuts (1/2/3) still work, status dots populate, "Add to Home Screen" on phone produces a proper icon (manifest + icons resolving correctly), no console errors.
3. 🟡 **Decide on `dashboard_data` backups** — now that it holds real hand-authored files instead of a regenerated blob, it's a real loss risk if not backed up. Add to the weekly rotation, or keep an off-Docker copy.
4. 📍 Still-open decision from before: should the dashboard itself sit behind Cloudflare Access? `noindex` was added this session as a partial mitigation (stops search-engine indexing) but doesn't stop a direct visitor or scanner from seeing the OpenPosterDB link, which has no Access gate of its own.
5. 📍 Headroom on Docker Desktop's memory cap was ~120MB as of the last audit — re-check with `docker stats` before deploying anything else; not re-verified this session.
6. 📍 Monitor OpenPosterDB and Portainer under their adjusted limits.
7. 📍 Monitor AIOMetadata memory (was 785.5MiB/950MiB, 82.69%).
8. 📍 Confirm the Sunday 3 AM backup job fired and pruned correctly on its first real run (was due Aug 16).
9. 📍 Reconcile or archive `claude_MASTER_SESSION_CONTEXT.md` (stale, describes old 7-service stack).
10. 📍 Keep a redundant copy of this context file in `/Users/rj/backups/` — especially now, since it's just been regenerated once already after being lost.

---

## 🎯 HOW TO USE

Paste this file and say: **"Here's my context, get up to speed."**
