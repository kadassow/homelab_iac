# Homelab IaC — Build Notes

Running log of what's been built, why, and what's still outstanding. Meant to be
pasted into a new chat (or committed to the repo as `BUILD_NOTES.md`) so context
isn't lost between sessions.
## Intention
The goal of this project is to build internet facing infrastructure to expose home lab services to authorized family members.  
Leveraging Ansible to consistently build that infrastructure.  

## Architecture

- **DMZ VM** (`vm1-dmz`, inventory group `gateways`) — limited attack surface,
  runs WireGuard + reverse proxy (AdGuard/NPM). Compose files live in
  `docker/compose/vm_dmz/` (`proxystack`, `vpnstack`).
  This currently has 8gb ssd space allocated, up to 2gb ram, leveraging 1 cpu cores.
- **Services VM** (`vm2-services`, inventory group `cores`) — hosts the actual
  internet-accessible Docker services. Compose files live in
  `docker/compose/vm2_services/` (`arrstack`, `infrastructure`, `mediastack`,
  `photostack`, `smarthome`, `stashstack`).  
  This currently has 100gb ssd space allocated, up to 8gb ram, leveraging 4 cpu cores.
- Both VMs run **Debian 13 (Trixie)**.
- Ansible control node: **must be Linux or WSL** — `ansible-vault` and Ansible's
  control-node functionality aren't supported on native Windows. Git pushes of
  already-encrypted files can happen from anywhere (Windows or WSL) since the
  file is opaque ciphertext once encrypted.

**Exposure model revised — see `EXPOSURE_ARCHITECTURE.md` for full
reasoning.** Not every service routes through the DMZ VM + Authentik
anymore. Current split:

- **DMZ VM (`vm1-dmz`) → NPM → Authentik**: Jellyfin (planned), NutriTrace
  (confirmed working) — high-bandwidth or native-OIDC apps with a real
  reason to sit behind this specific path.
- **Cloudflare Zero Trust Tunnel** (already set up, outbound-only from
  `vm2-services`, not yet Ansible/Git-managed): arr stack (Radarr, Sonarr,
  Bazarr, Prowlarr, Jellyseerr, qBittorrent WebUI) — low-bandwidth
  management UIs, auth handled by Cloudflare Access rather than Authentik.
- **LAN-only, not exposed**: Stash (and `stash-vr`) — deliberately kept
  internal for now, pending a specific reason to expose it.
- **WireGuard + Tailscale** on `vm1-dmz` (`vpnstack`) — both running side
  by side, deliberately, to compare them for phone remote access rather
  than committing to one. Replaces a previously standalone,
  informally-built Tailscale LXC (subnet router role) — nothing from that
  LXC needs to be migrated over. Decision pending: run WireGuard on
  `vm1-dmz` as planned, or use the TP-Link Deco X75 Pro's built-in
  WireGuard server instead.

Reasoning: routing every service through one auth gate (the original
plan) means a single Authentik bug becomes a skeleton key for the whole
homelab. Splitting by exposure type/bandwidth profile reduces the number
of internet-facing paths in the first place, rather than trusting one gate
to guard all of them.


## Home Assistant VM (planned)

A new, dedicated VM for full HAOS (Home Assistant OS with Supervisor/
add-ons) — deliberately not HA Core in Docker, and not folded into
`vm3-internal` or any LXC.

- Proposed static IP: `192.168.69.242`
- Sonoff ZBDongle-P passed through via Proxmox USB passthrough
- LAN-only exposure — no internet-facing path planned
- Backup via HAOS's own snapshot mechanism, separate from Kopia (HAOS
  doesn't expose a filesystem Kopia can cleanly snapshot the way
  `LOCALDOCKER_DIR` does)
- **Out of Ansible's scope**: HAOS has no Python/package manager and a
  restricted SSH add-on, so Ansible can only handle Proxmox-level VM
  provisioning — same treatment as the Cloudflare Tunnel and Proxmox GPU
  config (documented, not playbook-managed)
- Full install plan: `docs/HOME_ASSISTANT_SETUP.md`


## Remote Access — Tailscale Notes

- Tailscale already installed on phone; used to reach Jellyfin (and
  potentially other services) while away from home, without routing
  through the Cloudflare Tunnel.
- Considered a Tailscale subnet-router LXC so the phone can reach all
  server apps remotely, not just Jellyfin — this is effectively what
  `vpnstack`'s Tailscale container will do once built via Ansible.
- **Known gotcha**: phone-side DNS issues (`jellyfin.sparkylab.win` not
  resolving over Tailscale, though the bare IP worked) were not fixed by
  logging out/into the Tailscale app — only a full uninstall/reinstall
  resolved it. Chrome also needed to be explicitly added to Tailscale's
  split-DNS allow list on the phone before it would resolve LAN IPs at
  all.
- `vpnstack` plan: WireGuard + Tailscale as Docker containers on
  `vm1-dmz`, deployed via a new Ansible play, with Kopia backing up the
  important config/state files for both.

# Cloudflare
I have a cloudflare account with zero trust application setup to talk to my cloudflare tunnel lxc. 
- This lxc may be better moved to vm and controlled by ansible with kopia support.

# VM installation
see VM_CLEAN_REBUILD_CHECKLIST.md for initial installation procedure. 

# `playbook.yml` — what each part does and why

## Play 1: `Standardize and Install Docker Engine` (hosts: all)

1. **Disable IPv6** via `ansible.posix.sysctl`, writing
   `/etc/sysctl.d/99-disable-ipv6.conf`. Root cause: on `vm1-dmz`, IPv6 routing
   is dead (`Network is unreachable` on every IPv6 address), which caused
   Ansible's `get_url` module to silently produce an empty/truncated file when
   downloading the Docker GPG key (unlike `curl`, which has automatic
   IPv4/IPv6 fallback). Disabling IPv6 outright since it's not used.
2. **Remove stale `/etc/apt/sources.list.d/docker.list`** before `apt update`
   runs. A malformed repo line from an earlier failed run persists on disk and
   breaks every subsequent `apt update` even after the playbook is fixed —
   this cleanup makes the playbook self-healing on re-runs.
3. **Docker GPG key**: originally downloaded via `get_url` — replaced with a
   `curl | gpg --dearmor` shell pipeline. Debian 13's new Sequoia-based apt
   verifier (`sqv`) is stricter about keyring format than the old GPG-based
   verifier, and this method is what Docker's own current docs recommend.
   Includes a stale-file cleanup + `stat`/`assert` check so a bad download
   fails loudly instead of surfacing as a cryptic `apt update` error later.
4. **Docker repo**: uses `{{ ansible_distribution_release }}` instead of a
   hardcoded codename (was hardcoded `bookworm`, which was also wrong —
   Docker officially supports Debian 13/Trixie now, no need to pin to 12).
   Repo URI, suite, and component are correctly split
   (`https://download.docker.com/linux/debian` / `trixie` / `stable`) — the
   original bug folded the suite into the URI, leaving no component, which is
   what caused `E:Malformed entry 1 (Component)`.
5. Installs `docker-ce`, `docker-ce-cli`, `containerd.io`,
   `docker-buildx-plugin`, `docker-compose-plugin`.

## Play 2: `Configure Core VM Specialized Storage` (hosts: cores)

1. Installs `cifs-utils`.
2. **CIFS credentials file** (`/etc/cifs-credentials`, mode `0600`, root-owned)
   templated from vault-backed vars, `no_log: true` so the password never
   appears in Ansible output.
3. **Local (non-CIFS) directories**: `LOCALDOCKER_DIR` and its `configs`/
   `sqlite` subfolders, created as plain local directories — deliberately
   *not* part of the CIFS mount loop. Reasoning: SQLite relies on POSIX file
   locking that CIFS/SMB doesn't reliably support, causing "database is
   locked" errors or corruption. All app configs/databases/logs (Sonarr,
   Radarr, Stash, etc.) live here instead, visible on the host for easy
   backup (rsync/borg/etc.), while only bulk media lives on the network share.
4. **CIFS mounts** (`DATA_DIR`, `DOCKERCONFIGS_DIR` only — 2 real shares) via
   `ansible.posix.mount`, idempotently writing proper `/etc/fstab` entries
   with `_netdev,x-systemd.automount,x-systemd.mount-timeout=30`.
5. **`docker.service.d/wait-for-mounts.conf`** systemd drop-in with
   `RequiresMountsFor` pointed at both CIFS mount paths. Fixes a real boot
   race condition: without this, Docker can start (and containers can
   bind-mount) *before* the CIFS share actually lands on top of that path.
   Docker doesn't reconnect containers to a mount that appears after the
   fact — the container just keeps seeing the empty local directory that was
   there first. VM startup order in Proxmox does **not** fix this since the
   race is *inside* `vm2-services`, not between the two VMs. Handlers
   (`daemon-reload` + `restart docker`) only fire when the drop-in changes.

## `group_vars/all/vars.yml` (non-secret, committed in plain YAML)

- `PUID`, `PGID`, `TZ`
- `DATA_DIR`, `DOCKERCONFIGS_DIR` — the two real CIFS mount points
- `LOCALDOCKER_DIR` (default `/localdocker`) — local disk for
  sqlite/logs; 
- `cifs_server` + `cifs_mounts` list (server IP and share names — **edit
  `cifs_server` to your real NAS/OMV IP**, currently a placeholder)
- Secret values (`VPNPROVIDER`, `VPNUSERID`, `VPNPASSWORD`, `VPNREGION`,
  `JDOWNLOADEMAIL`, `JDOWNLOADPASS`, `STASH_API_KEY`, `CIFS_USERNAME`,
  `CIFS_PASSWORD`) are all just references to `vault_`-prefixed variables —
  the real values live only in the encrypted vault file.

## `group_vars/all/vault.yml` (encrypted, committed as ciphertext)

Holds: 

### VPN Credentials
vault_vpn_provider:       # e.g. protonvpn, mullvad, nordvpn
vault_vpn_userid: 
vault_vpn_password: 
vault_vpn_region: 

### Jdownloader
vault_jdownload_email: 
vault_jdownload_pass:

### Stash secrets
vault_stash_api_key: # pro stash
vault_stash_api_key_personal: # personal stash
vault_stash_ip: # ip address for stashapp server

### CIFS credentials
vault_cifs_username: # user name to use for omv cifs
vault_cifs_password: # password to use for omv cifs 
vault_cifs_server: # IP address to omv

### Nutritrace secrets
vault_nutritrace_jwt_secret: 
vault_nutritrace_oidc_client_secret: 
vault_oidc_issuer: # base dns name, assuming vars will prefix with scheme and suffix with path as needed 

### Kopia Backup Server
vault_kopia_repo_password: 
vault_kopia_server_admin_user: 
vault_kopia_server_admin_password: 
vault_kopia_server_url:  # base url only
vault_kopia_client_pw_vm1_dmz: 
vault_kopia_client_pw_vm2_services: 
vault_kopia_server_cert_fingerprint: 


Workflow: `ansible-vault create group_vars/all/vault.yml` the first time (opens
your editor, encrypts on save — plaintext never touches disk unencrypted);
`ansible-vault edit group_vars/vault.yml` for any future changes. Run
playbooks with `--ask-vault-pass` or `--vault-password-file`. The `.example`
version (no real secrets) is fine to keep in the repo as a template.

# Known issue still to fix in the repo

`docker/compose/vm2_services/stashstack/docker-compose.yml` has a **live JWT
API key hardcoded in plaintext** in the `stash-vr` service's `STASH_API_KEY`
environment value. Since it's been committed to git, treat it as compromised.
Fix: change that line to `STASH_API_KEY: "${STASH_API_KEY}"` and generate a
fresh key in Stash after deployment — do not reuse the old one.
** This is corrected as a secret in the vault.yml now **

# Status: DONE so far

- ✅ Ansible bootstrap playbook (Docker install, IPv6 disabled) runs
  successfully on both VMs
- ✅ Secrets scaffolding (`all.yml` + encrypted `vault.yml`) committed
- ✅ CIFS mount + boot-race-condition fix added to playbook
- ✅ Local disk path separated out for sqlite/logs/configs
- ✅ 1. **Fix the hardcoded Stash API key** in the compose file (see above) and
   rotate the key.
   ** This is Fixed, api key in encrypted vault.yml **
- ✅ 4. *Render `.env` files per stack** so `docker compose` can actually resolve
   `$PUID`, `$DATA_DIR`, `$VPNPASSWORD`, etc. — compose does not know about
   Ansible's variable space on its own; it just reads a `.env` file sitting
   next to each `docker-compose.yml`. This still needs to be built.
   ** This is working for nutritrace in the media stack, assuming similar build for other stacks **

- ✅ 6. **Deploy**: once 2–4 are done, add a task using
   `community.docker.docker_compose_v2` (or `command: docker compose up -d`

- ✅ Authentik deployed (dedicated LXC via Proxmox helper script) and
  NutriTrace OIDC login confirmed working end-to-end — see
  `AUTHENTIK_OIDC_SETUP.md`
- ✅ arr-stack and stash-stack deployed via Ansible to `vm3-internal`
   (`docker_internal` group) — compose copy, `.env` render, and
   `docker_compose_v2` deploy plays all in `playbook.yml`.
   ** Second GPU VF (`.2`) passed through specifically for Stash
   transcoding, separate from vm2-services's VF — verify this on
   vm3-internal's first boot per PROXMOX_GPU_SETUP.md's checkpoint. **
- ✅ Jellyfin live TV working — Threadfin deployed, Samsung TV Plus XMLTV
   guide wired in, channel set scoped to history/ancient-mysteries/
   HGTV-style plus a custom scheduled channel built from personal library
   (X-Files episodes). Two non-obvious fixes worth keeping documented:
   EPG Source had to be switched from PMS to XEPG for channel mapping to
   work, and live playback needed the buffer set to FFmpeg specifically
   inside the Samsung playlist's own per-playlist edit dialog in
   Threadfin (not the global Settings page).

# Status: NOT DONE yet — next steps


3. **Get the compose files onto each VM** — decide between `ansible.builtin.
   template`/`copy` per stack vs. a `git clone`/`synchronize` of the whole
   repo onto each host.

5. **Confirm DMZ ↔ Services VM firewall rules** — what ports NPM/AdGuard/
   WireGuard expose publicly, and what's allowed DMZ → services VM
   internally. Not yet addressed; likely a manual firewall/router
   configuration step rather than something this playbook handles.

- confirm jellyfin decodes using intel qsv gpu features.
- verify docker containers store things in the correct locations
- rebuild omv server to host new terramaster d4-320 das with 4 20tb hard drives.  I will use mergerfs and snapraid plugins in omv. I want to setup drive 1 for jellyfin, fallback to drive 3, drive 2 is stashapp data, fallback to drive 3.  drive 3 is ebook, immich, anything else.  drive 4 is parity.  I would also like to incorporate my separate external usb 20tb hdd in this and use for backups and potentially more fallback data for all pools.
We need to do this all correctly to make setup of other services much easier as my current omv build seems incorrect or inappropriate where I had just got it to work brute force style.

- setup calibre web automated
- setup kopia server and clients to centralize backup strategy.
- setup authentik totp, sso, password complexity and domain auth
- setup shelfarr service to feed into cwa
- install portainer on docker hosts
- setup ai llm to analyze nutritrace pictures for meal and ingredient detection.
- setup uptime kuma or similar dashboard for monitoring kopia backups and service uptime.  Should be separate lxc for monitoring, maybe in the infrastructure compose stack
- setup homarr dashboard
- create restore playbook to make rebuilding and restoring vm services.
- setup bitwarden to feed vault password to ansible
- setup docker container schedules.  example - jellyfin could probably be shut down between 1am to 7am or trailarr isn't needed for a long time.  Will this save resources on the host or is it more troublesome than it's worth?
- Build dedicated Home Assistant VM (full HAOS) — see new "Home Assistant
  VM" section above; `docs/HOME_ASSISTANT_SETUP.md` has the drafted plan.
- Decide WireGuard-on-vm1-dmz vs. TP-Link Deco X75 Pro's built-in
  WireGuard server for phone remote access; build `vpnstack` (WireGuard +
  Tailscale) via Ansible either way, with Kopia backing up state.
- Decommission the standalone, non-Ansible-managed Tailscale LXC once
  `vpnstack`'s Tailscale container is confirmed working.
- Drop `i915.max_vfs` from `2` to `1` on the Proxmox host once confirmed
  only one VF is actually needed — currently running `max_vfs=2` carried
  over from initial SR-IOV setup. (Note: if vm3-internal's second VF for
  Stash transcoding is confirmed needed long-term, keep `max_vfs=2`
  instead — don't drop this until that's settled.)
- Fix `.env` rendering pipeline risk: NutriTrace OIDC fixes were
  hand-applied directly to `.env` on vm2-services at one point and could
  silently revert on the next Ansible run that regenerates it from the
  Jinja2 template. Needs either a documented manual re-apply step or a
  template fix so hand-edits can't silently disappear.
- Monitoring: tool choice settled — Uptime Kuma specifically (not the
  fuller Prometheus/Grafana stack), as a push-monitor dead-man's-switch
  for Kopia backup verification. Placement still undecided: new dedicated
  LXC (via Proxmox community helper script, Ansible-onboarded after) vs.
  reusing vm3-internal.
  
# Questions and future tasks
1. ~~Can we point my arrStack and StashStack to an authentik server which
   then authorizes first and then proxies to the right service?~~
   **Answered**: yes, via Authentik's Forward Auth (Proxy Provider) —
   not yet implemented, next integration target. See
   `AUTHENTIK_OIDC_SETUP.md`.
2. I want to support direct access still for my home wifi users without
   authentik auth for certain apps. Is this possible?
   **Leaning toward**: routing LAN traffic to `vm2-services` directly via
   internal DNS (AdGuard), bypassing NPM/Authentik entirely for LAN
   clients, rather than an IP-based Authentik policy. Not yet built.


AUTHENTIK_OIDC_SETUP.md — Authentik LXC deployment + NutriTrace OIDC integration, including the full troubleshooting chain (redirect URI, issuer scheme, provider-ID mismatch, account linking).

# Hardware in homelab
## Mini pc hardware 
- https://www.amazon.com/dp/B0DP2SGVVY
- **Host**: Beelink Mini S13 Mini PC
- **CPU/iGPU**: Intel Twin Lake N150 (up to 3.6GHz, successor to N100),
  integrated UHD Graphics (Quick Sync Video capable — H.264/HEVC/AV1
  encode+decode)
- **RAM**: 16GB DDR4
- **Storage**: 500GB M.2 SSD
- **Networking**: WiFi 6, BT 5.2
- **Hypervisor**: Proxmox VE
- **Guests on this host**: `vm2-services` (Debian 13 VM, runs the media/app
  Docker stacks — see main `claude.md`), plus several existing LXCs that also
  need GPU access for their own transcoding workloads

- zigbee usb dongle - sonoff ZBDongle-P (zigbee connection 3.0)  SONOFF ZBDongle-P Smart Zigbee Bridge USB Dongle Plus Stick Universal Gateway US

- 20tb external usb hdd
- terramaster D4-320 DAS enclosure, 4 20tb hdd

## pve host cpu perfomance
I set cpu governor to powersave 
use script to change -  
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/scaling-governor.sh)"

## Router
TP-LInk x75 pro mesh wifi 

Router/gateway IP: 192.168.68.1
Homelab servers live in the 192.168.69.0–255 range — VM network configs
(qm set --ipconfig0, Ansible inventory) need a /23 CIDR, not /24, to span
both subnets correctly.
 
## Android phone apps
I want to support native android apps - 
Jellyfin
bitwarden
immich

## TV apps
I want jellyfin to be supported on tv apps - samsung, fire tv, sony.
