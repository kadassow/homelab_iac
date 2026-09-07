# Kopia Backup Strategy — Setup & Decision Notes

Documents the decision to standardize on **Kopia** as the single backup tool
across the homelab, replacing the earlier rsync (media) + restic (pictures)
plan before either was actually deployed. Meant to live alongside
`PROXMOX_GPU_SETUP.md` and `BUILD_NOTES.md` at the repo root so the reasoning
survives between sessions.

## Why Kopia instead of rsync + restic

The original plan split backups by data type: rsync for replaceable media,
restic (with retention) for the irreplaceable pictures library. Neither job
was actually cron'd yet, and the rsync side in particular was shaping up to
be ad hoc — a per-share script with no shared policy engine, credential
model, or way to extend cleanly to every VM and LXC as the homelab grows.

Kopia can do everything both tools were going to do, from one client binary
and one policy engine:

- Content-defined chunking + dedup + compression, same as restic — fine for
  the irreplaceable pictures library, with proper versioned snapshots and
  retention.
- Fine for the replaceable media library too, just with a lighter retention
  policy — no need for a separate rsync codepath.
- One consistent way to add a new source (a new LXC, a new VM) — install the
  client, connect to the server, define a policy. No new script to write.
- A genuine multi-client **server mode** with per-user access control, which
  solves a security problem the rsync/restic plan never addressed (see
  below).

## Why a dedicated Kopia server, not direct-to-repository clients

Kopia supports two topologies:

1. **Direct repository access** — every client holds the full repository
   password and talks straight to the storage backend.
2. **Kopia server mode** — one machine holds the repository password and
   exposes an HTTPS API; every other machine authenticates with its own
   scoped username/password and can only see/manage its own snapshots.

Option 2 is the one that fits this homelab's trust model. `vm1-dmz` is the
most exposed machine on the network — it's the one running WireGuard and the
reverse proxy, internet-facing by design. It should never be a machine that,
if compromised, hands over the credentials needed to read or delete *every*
machine's backups. With Kopia server mode, `vm1-dmz` only ever holds a scoped
client credential tied to its own identity — worst case from a compromise
there is damage to its own backup set, not the whole repository.

## Topology

```
 vm1-dmz            vm2-services         other LXCs / OMV
 (scoped cred)      (scoped cred)        (scoped cred)
      \                   |                    /
       \                  |                   /
        \-----------> Kopia server <---------/
                  (holds repo password,
                   enforces per-client ACLs)
                           |
                           v
                    Backup storage
                 (20TB drive, repo data)
```

- **Kopia server** lives in its own dedicated LXC on the Proxmox host — not
  co-located with `vm2-services` (biggest attack surface among the
  internet-facing services) and not on the DMZ VM. Its only jobs are running
  the Kopia server process and owning the repository.
- The **20TB backup drive** is attached directly to this LXC as local block
  storage rather than mounted over CIFS. Kopia's content-addressable store
  doesn't have the SQLite-style locking problem CIFS causes elsewhere in
  this stack, but local disk removes a variable and keeps the pattern
  consistent with how `LOCALDOCKER_DIR` is already handled.
- Every source machine (`vm1-dmz`, `vm2-services`, other LXCs, and the OMV
  NAS itself) runs the Kopia **client** and connects outward to the server —
  nothing pushes credentials the other direction.

## Security model

- `kopia server user add <name>@<hostname>` creates one scoped identity per
  source machine. Each identity is, by default, restricted to snapshots
  under its own username/hostname — it cannot browse or delete another
  client's snapshots without being explicitly granted broader (multiuser)
  permissions, which this setup deliberately avoids granting.
- The repository password (the actual encryption key) is only ever entered
  on the Kopia server LXC when the repository is created. It is never
  distributed to `vm1-dmz`, `vm2-services`, or any LXC.
- Follow the existing secrets pattern for anything Ansible touches: per-client
  Kopia passwords become `vault_`-prefixed vars in `group_vars/vault.yml`,
  referenced from `group_vars/all.yml` the same way `VPNPASSWORD` and
  `STASH_API_KEY` already are. Nothing plaintext in compose files or repo
  history.
- TLS between clients and the server — Kopia can self-sign
  (`--tls-generate-cert`) for a LAN-only server; fine to start with, revisit
  if the server ever needs to be reachable outside the LAN (it shouldn't be).

## What gets backed up, and how

Three data classes, three retention profiles, one tool:

| Data class | Source | Volatility | Retention approach |
|---|---|---|---|
| Docker configs / SQLite DBs | `LOCALDOCKER_DIR` on `vm1-dmz`, `vm2-services`, other LXCs | Changes often, easy to lose real work (arr configs, Stash configs, Jellyfin watch state) | Moderate: e.g. keep last 7 daily, 4 weekly, 3 monthly snapshots |
| Media library (movies/TV/music) | `DATA_DIR` on OMV | Replaceable — re-downloadable via the arr stack | Light: keep latest + maybe 1–2 weekly snapshots, mainly for accidental-delete protection, not long history |
| Pictures library | OMV, dedicated share | Irreplaceable | Strong: e.g. keep last 14 daily, 8 weekly, 12 monthly, and consider a yearly tier |

`DATA_DIR` and `DOCKERCONFIGS_DIR` already live on the NAS (CIFS-backed), so
their Kopia client runs **on OMV itself** rather than on `vm2-services` —
`vm2-services` only needs to back up what's actually local to it
(`LOCALDOCKER_DIR`).

## Setup steps

### 1. Provision the Kopia server LXC

Minimal Debian 13 LXC, a couple GB RAM is plenty. Attach the 20TB drive
(or the interim storage while that drive is being stood up) as a mounted
local block device, not a CIFS share.

```bash
apt update && apt install -y curl gnupg
curl -s https://kopia.io/signing-key | gpg --dearmor -o /etc/apt/keyrings/kopia-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kopia-keyring.gpg] http://packages.kopia.io/apt/ stable main" \
  > /etc/apt/sources.list.d/kopia.list
apt update && apt install -y kopia
```

### 2. Initialize the repository

Run once, directly on the server LXC:

```bash
kopia repository create filesystem --path=/mnt/backup-drive/kopia-repo
```

This is the one moment the repository password is entered anywhere. Store
it in `ansible-vault` immediately (`vault_kopia_repo_password`), then treat
it like the CIFS/VPN secrets already in the vault — never in a compose file,
never logged.

### 3. Start the server and create per-client users

```bash
kopia server start --tls-generate-cert \
  --address=0.0.0.0:51515 \
  --server-username=admin --server-password=<admin-pw>

kopia server user add vm1-dmz@vm1-dmz     --user-password=<scoped-pw>
kopia server user add vm2-services@vm2-services --user-password=<scoped-pw>
kopia server user add omv@omv             --user-password=<scoped-pw>
# repeat per LXC
```

Exact flags drift between Kopia releases — check current docs before
running, this is the shape of it rather than a copy-paste guarantee.

### 4. Connect each client

On `vm1-dmz`, `vm2-services`, each LXC, and OMV:

```bash
apt install -y kopia   # same repo setup as step 1
kopia repository connect server \
  --url=https://<kopia-server-ip>:51515 \
  --username=<hostname>@<hostname> --password=<scoped-pw> \
  --server-cert-fingerprint=<fingerprint-from-server-start-output>
```

### 5. Define snapshot policies per data class

```bash
# on vm2-services
kopia policy set /opt/docker-local \
  --keep-daily 7 --keep-weekly 4 --keep-monthly 3

# on OMV, media share
kopia policy set /path/to/data/media \
  --keep-latest 1 --keep-weekly 2

# on OMV, pictures share
kopia policy set /path/to/data/pictures \
  --keep-daily 14 --keep-weekly 8 --keep-monthly 12
```

### 6. Schedule snapshots

Kopia can run its own background scheduler (`kopia server` handles this for
connected clients automatically once policies are set), or drive it via cron
per host for simplicity/visibility — pick whichever matches how the rest of
this repo is managed. Given the project's Ansible-first approach, this is a
natural candidate for a `kopia` role later (see Ansible integration below).

### 7. Verify — CHECKPOINT

```bash
kopia snapshot list --all      # from the server, confirms every client is checking in
kopia snapshot verify          # spot-check restorability, don't just trust "it ran"
```

Treat a successful `kopia snapshot verify` on at least one snapshot per data
class as the real checkpoint — a snapshot that completes without error but
was never test-restored isn't a verified backup.

## Ansible integration (future)

Not built yet, but the natural home for this once the manual setup above is
proven out:

- A `kopia_client` role (hosts: `all` + relevant LXCs) — installs the
  package, templates the connection command with a vault-backed password,
  defines policies via `kopia policy set` tasks.
- A `kopia_server` role (its own inventory group, e.g. `backup`) — installs
  Kopia, creates the repository (idempotently — skip if already
  initialized), manages server users from a vault-backed list.
- Fits the existing `group_vars/all.yml` / `group_vars/vault.yml` pattern
  without changes to that structure, just new `vault_kopia_*` entries.

## Offsite backup (future / open item)

The 20TB drive attached to the Kopia server LXC is still on the same home
network — not a true 3-2-1 offsite copy, same gap noted for the earlier
Windows-share interim target. Kopia has a relevant answer for this:
`kopia repository sync-to` can mirror a local repository to a remote backend
(e.g. Backblaze B2) without needing every client to know about the second
location — only the Kopia server needs to run the sync job. That makes B2
(already under consideration for the pictures library) a reasonable next
step: point a scheduled `sync-to b2` job at it from the server LXC once the
primary repository is stable and verified. Not implemented yet — noted here
so it isn't lost.

## Status

- ⬜ Not yet started — this doc reflects the plan and reasoning; no LXC,
  repository, or client connections exist yet.
- ⬜ rsync and restic jobs intentionally not being built — superseded by
  this plan before they were ever cron'd.

## Open questions / future considerations

- Where exactly does the Kopia server LXC live relative to the existing LXC
  set, and what's its inventory group name for eventual Ansible management?
- Cron-per-host vs. Kopia's own server-driven scheduling — pick one pattern
  and apply it consistently once the manual setup is validated.
- Retention numbers above are starting points, not final — revisit after
  seeing real dedup/compression ratios and actual storage consumption on the
  20TB drive.
- `sync-to` cadence for eventual B2 offsite — nightly is probably overkill
  for the pictures library; weekly may be sufficient given local snapshots
  already cover short-term recovery.
