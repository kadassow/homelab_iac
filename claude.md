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
- **Services VM** (`vm2-services`, inventory group `cores`) — hosts the actual
  internet-accessible Docker services. Compose files live in
  `docker/compose/vm2_services/` (`arrstack`, `infrastructure`, `mediastack`,
  `photostack`, `smarthome`, `stashstack`).
- Both VMs run **Debian 13 (Trixie)**.
- Ansible control node: **must be Linux or WSL** — `ansible-vault` and Ansible's
  control-node functionality aren't supported on native Windows. Git pushes of
  already-encrypted files can happen from anywhere (Windows or WSL) since the
  file is opaque ciphertext once encrypted.
  
## `playbook.yml` — what each part does and why

### Play 1: `Standardize and Install Docker Engine` (hosts: all)

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

### Play 2: `Configure Core VM Specialized Storage` (hosts: cores)

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

## `group_vars/all.yml` (non-secret, committed in plain YAML)

- `PUID`, `PGID`, `TZ`
- `DATA_DIR`, `DOCKERCONFIGS_DIR` — the two real CIFS mount points
- `LOCALDOCKER_DIR` (default `/opt/docker-local`) — local disk for
  sqlite/logs; `CONFIGS_DIR` and `SQLITE_DIR` are subfolders under it, kept
  as separate variable names only so existing compose files (`$CONFIGS_DIR`,
  `$SQLITE_DIR`) don't need to change
- `cifs_server` + `cifs_mounts` list (server IP and share names — **edit
  `cifs_server` to your real NAS/OMV IP**, currently a placeholder)
- Secret values (`VPNPROVIDER`, `VPNUSERID`, `VPNPASSWORD`, `VPNREGION`,
  `JDOWNLOADEMAIL`, `JDOWNLOADPASS`, `STASH_API_KEY`, `CIFS_USERNAME`,
  `CIFS_PASSWORD`) are all just references to `vault_`-prefixed variables —
  the real values live only in the encrypted vault file.

## `group_vars/vault.yml` (encrypted, committed as ciphertext)

Holds: `vault_vpn_provider`, `vault_vpn_userid`, `vault_vpn_password`,
`vault_vpn_region`, `vault_jdownload_email`, `vault_jdownload_pass`,
`vault_stash_api_key`, `vault_cifs_username`, `vault_cifs_password`.

Workflow: `ansible-vault create group_vars/vault.yml` the first time (opens
your editor, encrypts on save — plaintext never touches disk unencrypted);
`ansible-vault edit group_vars/vault.yml` for any future changes. Run
playbooks with `--ask-vault-pass` or `--vault-password-file`. The `.example`
version (no real secrets) is fine to keep in the repo as a template.

## Known issue still to fix in the repo

`docker/compose/vm2_services/stashstack/docker-compose.yml` has a **live JWT
API key hardcoded in plaintext** in the `stash-vr` service's `STASH_API_KEY`
environment value. Since it's been committed to git, treat it as compromised.
Fix: change that line to `STASH_API_KEY: "${STASH_API_KEY}"` and generate a
fresh key in Stash after deployment — do not reuse the old one.

## Status: DONE so far

- ✅ Ansible bootstrap playbook (Docker install, IPv6 disabled) runs
  successfully on both VMs
- ✅ Secrets scaffolding (`all.yml` + encrypted `vault.yml`) committed
- ✅ CIFS mount + boot-race-condition fix added to playbook
- ✅ Local disk path separated out for sqlite/logs/configs

## Status: NOT DONE yet — next steps

1. **Fix the hardcoded Stash API key** in the compose file (see above) and
   rotate the key.
   ** This is Fixed **
3. **Get the compose files onto each VM** — decide between `ansible.builtin.
   template`/`copy` per stack vs. a `git clone`/`synchronize` of the whole
   repo onto each host.
4. **Render `.env` files per stack** so `docker compose` can actually resolve
   `$PUID`, `$DATA_DIR`, `$VPNPASSWORD`, etc. — compose does not know about
   Ansible's variable space on its own; it just reads a `.env` file sitting
   next to each `docker-compose.yml`. This still needs to be built.
5. **Confirm DMZ ↔ Services VM firewall rules** — what ports NPM/AdGuard/
   WireGuard expose publicly, and what's allowed DMZ → services VM
   internally. Not yet addressed; likely a manual firewall/router
   configuration step rather than something this playbook handles.
6. **Deploy**: once 2–4 are done, add a task using
   `community.docker.docker_compose_v2` (or `command: docker compose up -d`

## Questions and future tasks
1 - Can we point my arrStack and StashStack to an authentik server which then authorizes first and then proxies to the right service?
2 - I want to support direct access still for my home wifi users without authentik auth for certain apps.  Is this possible?
   as a fallback) per stack, targeting the correct host group
   (`gateways` → vm_dmz composes, `cores` → vm2_services composes).
