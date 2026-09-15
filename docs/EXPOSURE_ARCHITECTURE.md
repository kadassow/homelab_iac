# Exposure Architecture — Decision Notes

Documents a revision to the original plan of routing every internet-facing
service through the DMZ VM (`vm1-dmz`) + Authentik. Meant to sit alongside
`AUTHENTIK_OIDC_SETUP.md`, `KOPIA_BACKUP_STRATEGY.md`, and
`PROXMOX_GPU_SETUP.md` — same reasoning: capture why the architecture
changed so it isn't re-litigated from scratch in a future session.

## The original plan, and why it changed

The initial design routed everything through one path: internet →
WireGuard/NPM on `vm1-dmz` → Authentik → whichever internal service. The
appeal was a single, consistent auth gate and a single place to manage
access control for the whole homelab.

The problem: that design bets the security of *everything* on Authentik
never having a bad bug. Authentik is itself a service with its own attack
surface and CVE history (see `AUTHENTIK_OIDC_SETUP.md`'s note on
CVE-2026-40166) — running on this network like any other app. Funneling
every service's internet exposure through it turns one Authentik
compromise into a skeleton key for the whole homelab, rather than a
contained incident.

**Revised principle: reduce the number of paths in from the internet in
the first place, rather than routing every path through one gate.**
Defense in depth by *exposure type*, not just by auth layer.

## Current exposure model

| Service | Exposure path | Auth | Notes |
|---|---|---|---|
| Jellyfin | DMZ VM (`vm1-dmz`) → NPM → internet | Authentik (planned/TBD) | High-bandwidth video streaming — the actual reason the DMZ/WireGuard path exists, per Cloudflare's tunnel ToS restrictions on this kind of traffic |
| NutriTrace | DMZ VM (`vm1-dmz`) → NPM → internet | Authentik (confirmed working end-to-end) | Native OIDC support, low bandwidth — good fit for this path |
| arr stack (Radarr, Sonarr, Bazarr, Prowlarr, Jellyseerr, qBittorrent WebUI, etc.) | **Cloudflare Zero Trust Tunnel**, direct from `vm2-services` | Cloudflare Access (not Authentik) | Low-bandwidth admin/management UIs — nobody streams video through Radarr. Tunnel is outbound-only from `vm2-services`, so no inbound port-forward and no DMZ/NPM involvement for these routes at all |
| Stash | **Not exposed — LAN-only for now** | N/A (internal network trust only) | Deliberately kept off both paths pending a specific reason to expose it. `stash-vr` is therefore also LAN-only for now — confirm this is fine for how it's actually used (no remote VR viewing currently planned) |
| WireGuard | Still available on `vm1-dmz` | N/A — full LAN-level trust once connected | Kept for whenever full-tunnel LAN access is actually needed (not per-app proxying) |

## Why arr stack → Cloudflare Tunnel specifically

- **Bandwidth profile fit**: arr apps are management/admin interfaces, not
  media-serving. They're exactly the kind of traffic Cloudflare's tunnel
  ToS is fine with, unlike Jellyfin's actual video streaming — which is
  the whole reason the DMZ/WireGuard path exists as a separate thing in
  this homelab (see `claude.md`'s original architecture rationale).
- **Attack surface reduction**: `cloudflared` makes an outbound-only
  connection to Cloudflare's edge — no listening port on the WAN side for
  these apps at all, on any host. Removes them from NPM, from `vm1-dmz`
  entirely, and from anything scanning for open ports on the home WAN IP.
- **Edge-level auth for free**: Cloudflare Access sits in front of the
  tunnel and can gate every arr route behind identity *before* the request
  ever reaches `vm2-services` — an attacker can't even reach the arr apps'
  own (often weak or nonexistent) native auth to probe it.

## Cloudflare Zero Trust Tunnel — current state

- **Already set up and running** — the tunnel itself exists prior to this
  doc; this section is about routing more services through it and
  deciding how auth works on top of it, not initial setup.
- **Not Ansible/Git-managed today.** Cloudflare Tunnel and Access policy
  configuration lives in Cloudflare's dashboard (or their API/Terraform
  provider, neither of which is wired into this repo yet) — same category
  as Authentik's LXC and the Proxmox host-level GPU config: real
  infrastructure this homelab depends on, but outside `homelab_iac`'s
  current Ansible-managed scope. Treat it as out-of-band, same caveat
  `PROXMOX_GPU_SETUP.md` and `AUTHENTIK_OIDC_SETUP.md` already carry for
  their respective pieces.
- **Possible future IaC path, not yet explored**: Cloudflare does publish
  a Terraform provider (and a plain REST API) that can manage tunnels,
  DNS records, and Access policies as code. Bringing that under version
  control would close this gap the same way Ansible closed it for
  everything else here — worth a dedicated investigation later, but not
  blocking the arr-stack cutover itself. Not evaluated yet: whether
  Terraform gets bolted on alongside Ansible, or whether there's a
  lighter-weight option (a plain script hitting Cloudflare's API,
  version-controlled even without full Terraform) that fits this repo's
  existing tooling better.

### Auth options for arr behind the tunnel — not fully decided yet

1. **Cloudflare Access as the only gate** — simplest, no new integration
   work. Access policy (who's allowed in) lives entirely in Cloudflare's
   config, separate from Authentik.
2. **Cloudflare Access federated to Authentik as the IdP** — Access
   enforces at the edge, but the actual "who is this person" check still
   goes through Authentik, keeping one source of truth for identity across
   both exposure paths. More setup than option 1.
3. **Both, layered** — probably redundant for arr specifically; not
   recommended unless a specific reason shows up.

**Leaning toward option 1 or 2** — not yet committed. Revisit once the
tunnel is actually routing arr traffic and it's clear whether keeping
Authentik as the single identity source matters enough to justify the
federation setup.

## What this supersedes

- `AUTHENTIK_OIDC_SETUP.md`'s "Open questions" item on Forward Auth (Proxy
  Provider) for arr stack and Stash is **superseded** — neither service is
  routing through Authentik under this revised architecture, so Forward
  Auth integration work for them is no longer needed. (Marked in that doc
  directly — see the update there.)

## Open questions

- Jellyfin's exposure path — still DMZ/NPM/Authentik as originally
  planned. Worth explicitly confirming this stays the plan (it's the
  single most internet-exposed app by actual usage), rather than assuming
  it by default just because it was never reconsidered.
- Whether Stash gets a real exposure decision later (Cloudflare Tunnel
  doesn't fit it — it's consumed directly like Jellyfin, not managed like
  arr — so it would need the DMZ/NPM/Authentik path if it's ever exposed).
- Cloudflare Access ↔ Authentik federation (option 2 above) — worth
  prototyping once arr is confirmed working behind the tunnel with basic
  Access policy first.
- Terraform-for-Cloudflare (or a lighter API-scripted alternative) as a
  future IaC target — not started.
