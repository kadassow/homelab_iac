# Storage VM Setup — mergerfs + SnapRAID + Samba (Decision Notes)

Documents the decision to decommission the OMV NAS in favor of a dedicated,
Ansible-managed storage VM (`vm4-storage`) running mergerfs + SnapRAID +
Samba directly on Debian — no NAS-distro abstraction layer. Meant to sit
alongside `KOPIA_BACKUP_STRATEGY.md`, `EXPOSURE_ARCHITECTURE.md`, and
`PROXMOX_GPU_SETUP.md`.

## Why a dedicated VM, not `vm2-services` or `vm3-internal`

Same blast-radius reasoning already applied to Kopia and Authentik: the
storage layer is what every other service reads from, so it shouldn't share
a failure domain with the highest-risk workloads. `vm3-internal` exists
specifically to isolate arr-stack/Stash (qBittorrent pulling from arbitrary
peers, apps with real CVE history) — putting the storage pool on that same
VM would mean a compromise or crash there takes storage down with it. A
fourth, dedicated VM (`vm4-storage`, own inventory group `storage`) keeps
the pool isolated the same way `kopia-server` and Authentik already are.

`vm4-storage` does not need Docker, QuickSync, or CIFS-client setup — it's
the thing *serving* shares, not consuming them. It stays out of
`docker_hosts` entirely.

## Why plain Debian instead of OMV

OMV's mergerfs/snapraid support is itself third-party (the `omv-extras`
community repo, not OMV core), so "OMV" was never actually buying more
official support for this specific need. Running mergerfs/snapraid/Samba
directly means using each project's own upstream packaging and docs
instead of a NAS distro's plugin wrapper — and it fits this repo's
IaC-first pattern: an Ansible role per concern, versioned config, no GUI to
fight, and one fewer non-IaC-managed system in the homelab. This repo does
not currently define what other configuration OMV's GUI handled outside of
mergerfs/snapraid/shares — confirm nothing else about the OMV instance
needs replicating before decommissioning it.

## DAS passthrough — per-disk, not whole-controller

Each of the D4-320's 4 drives is passed through to `vm4-storage`
individually, identified by stable `/dev/disk/by-id/` path (not `/dev/sdX`,
which can reorder on reboot):

```bash
# On the Proxmox host
ls -la /dev/disk/by-id/ | grep -i usb

qm set <vmid> --scsi1 /dev/disk/by-id/usb-<drive1-serial>
qm set <vmid> --scsi2 /dev/disk/by-id/usb-<drive2-serial>
qm set <vmid> --scsi3 /dev/disk/by-id/usb-<drive3-serial>
qm set <vmid> --scsi4 /dev/disk/by-id/usb-<drive4-serial>
```

**Why per-disk over whole-controller passthrough**: whole-controller
passthrough hands the VM one upstream USB hub — if the enclosure resets or
the hub hiccups, all 4 drives can drop simultaneously, which defeats the
point of SnapRAID parity (protects against one drive failing, not the
whole enclosure dropping off the bus at once).

**CHECKPOINT — verify before building the pool**: USB-SATA bridge chips are
inconsistent about passing through real SMART data. Run `smartctl -a`
against all 4 drives (and the reused OMV drive, and the Kopia backup drive)
through their actual enclosures before finalizing the layout. Garbled SMART
output is a DAS-firmware limitation, not something Ansible/smartmontools
config can fix — better to know now than after data is migrated.

The same fifth 20TB USB drive (see "Kopia scope" below) attaches instead to
the `kopia-server` LXC, using LXC-appropriate syntax (`pct set`, not
`qm set`):

```bash
pct set <kopia-server-vmid> -mp0 /dev/disk/by-id/usb-<serial>,mp=/mnt/backup-drive
```

# Addendum to docs/STORAGE_VM_SETUP.md — Drive Preparation

Insert this as a new section, **before** "## Drive pooling plan".

---

## Preparing each drive — manual, not Ansible-automated

Partitioning and formatting are deliberately kept as a manual, one-time,
per-drive step rather than an Ansible task. Unlike the rest of this repo's
destructive-looking-but-actually-safe operations (e.g. `kopia repository
create`, whose idempotency check only ever short-circuits on a harmless
"already exists" stderr match), a `mkfs` task has no equivalent safe
failure mode — a wrong condition, a wrong host limit, or a re-run against
an already-populated drive has no undo. Mounting an *already-formatted*
filesystem is safe to automate (see the updated `mergerfs` role below);
creating that filesystem in the first place is not.

### Filesystem choice: ext4, not btrfs

btrfs's main selling point here — checksumming to catch silent
corruption — doesn't add a real protection ceiling in this setup: each DAS
bay is an independent single-device mergerfs branch (no btrfs RAID1/mirror
across drives), so btrfs can only *detect* corruption, not self-heal it.
Recovery still comes from a SnapRAID restore either way, and SnapRAID's own
block-hash scrub already provides corruption detection independent of the
underlying filesystem. So btrfs mainly buys earlier detection, not
additional recoverability — at the cost of real downsides for this
specific workload: btrfs's copy-on-write behavior is well known to cause
progressive fragmentation under torrent-style write patterns (qBittorrent
pre-allocates files and writes pieces out of order), and the usual fix
(`chattr +C` to disable CoW per-directory) also disables the checksumming
that was the reason to consider btrfs in the first place. ext4 has none of
that fragmentation behavior, is the more common SnapRAID pairing in
practice, and is simpler to reason about for a pool meant to be set up once
and left alone. **Decision: ext4 on every branch, parity disk, and the
Kopia backup drive — one consistent filesystem type across the whole
setup.**

### Steps — repeat per drive (3 new DAS drives, the reused OMV drive, the
parity drive, and the separate Kopia backup drive)

**Do this one drive at a time. Confirm the `by-id` serial against the
physical bay/enclosure label before touching anything — there is no
confirmation prompt once `mkfs` runs.**

```bash
# 0. One-time on vm4-storage: minimal Debian doesn't ship these. Not worth
#    adding to the mergerfs role — the role only mounts already-formatted
#    filesystems, it never calls parted/mkfs itself, so these tools are
#    never needed after this manual prep is done.
apt update && apt install -y parted gdisk smartmontools e2fsprogs

# 1. Identify the drive — confirm this serial matches the physical bay
#    you think it is before proceeding.
ls -la /dev/disk/by-id/ | grep -i usb

# 2. If reusing a drive that already has data/a filesystem (the old OMV
#    drive), make sure its data has already been copied off per the
#    migration steps in STORAGE_VM_SETUP.md BEFORE this step. Then wipe
#    old filesystem signatures:
wipefs -a /dev/disk/by-id/usb-<serial>

# 3. Partition — single GPT partition covering the whole disk:
parted /dev/disk/by-id/usb-<serial> --script mklabel gpt mkpart primary ext4 0% 100%

# 4. Format — label matches the branch it'll serve (disk1/disk2/disk3/
#    diskfallback/parity/kopiabackup), so `lsblk -f` is self-explanatory
#    later:
mkfs.ext4 -L disk1 /dev/disk/by-id/usb-<serial>-part1

# 5. Get the filesystem UUID — this is what goes in Ansible's fstab entry,
#    not the by-id path (by-id partition suffixes aren't fully consistent
#    across all USB bridge chips; UUID is the stable identifier):
blkid /dev/disk/by-id/usb-<serial>-part1
```

Record each drive's UUID against its intended role (disk1/disk2/disk3/
diskfallback/parity) — the updated `mergerfs` role below expects these in
`mergerfs_branch_uuids`.

### SMART monitoring is not available through this passthrough method

Drives passed through via `qm set --scsiN /dev/disk/by-id/usb-<serial>` are
direct raw block-device passthrough (no image file — writes go straight to
the physical disk), but the guest sees them through QEMU's emulated SCSI
controller, which identifies as `Vendor: QEMU, Product: QEMU HARDDISK` and
does not forward ATA/SMART passthrough commands — `smartctl` reports
"SMART support is: Unavailable" regardless of `-d sat`/`-d scsi` flags or
the real drive's own capability. This is a side effect of per-disk
passthrough (deliberately chosen over whole-controller passthrough to
avoid a shared-bus failure dropping all 4 drives at once), not a
TerraMaster/bridge-chip limitation — don't re-diagnose this later as a
hardware issue. SnapRAID's own scrub provides corruption detection
independent of SMART, so this is an accepted gap rather than a blocker.

**Once every drive is partitioned, formatted, and its UUID recorded, hand
off to Ansible** — mounting the finished filesystems and assembling the
mergerfs pool on top of them is safe to automate from here.
## Drive pooling plan

| Drive | Role | mergerfs branch? | SnapRAID member? |
|---|---|---|---|
| DAS Drive 1 (20TB) | Jellyfin — Movies & TV | Yes | Yes (data) |
| DAS Drive 2 (20TB) | Stash media | Yes | Yes (data) |
| DAS Drive 3 (20TB) | Immich photos + Calibre ebooks | Yes | Yes (data) |
| DAS Drive 4 (20TB) | SnapRAID parity | No | Yes (parity) |
| Reused OMV drive (20TB) | Universal fallback / safety net | Yes | Yes (data) |
| Fifth USB drive (20TB), separate from the above | Kopia repository target — irreplaceable-data backups | No — attaches to `kopia-server`, not this pool | No |

Parity math checks out: SnapRAID's parity drive must be at least as large
as the largest data drive; 20TB parity across four 20TB data members
(3 DAS + reused OMV drive) satisfies that.

### Enforcing per-category drive dedication in mergerfs

mergerfs has no literal "dedicate drive X to category Y" setting — it's a
union filesystem. Dedication comes from **which branch a category's folder
is pre-created on**, combined with an existing-path create policy:

- Pre-create `media/movies` and `media/tv` **only** under DAS Drive 1's
  branch.
- Pre-create `stash` **only** under DAS Drive 2's branch.
- Pre-create `photos` and `ebooks` **only** under DAS Drive 3's branch.
- Mount with `category.create=epmfs` (existing path, most free space) —
  new files for a category only land on branches where that category's
  folder already exists, which is what actually enforces the dedication.
  The default policy (`mfs`, drive-agnostic most-free-space) does **not**
  respect this and would scatter files across all branches.

Get the pre-creation step right once, before any data lands — it's fiddly
to fix after files are already scattered under a looser policy.

## Kopia scope — irreplaceable data only

- **Backed up via Kopia** (strong retention — 14 daily / 8 weekly / 12
  monthly, per `KOPIA_BACKUP_STRATEGY.md`'s existing "irreplaceable" tier):
  books (Calibre/CWA), photos (Immich), personal Stash data.
- **Not backed up via Kopia** — parity only: movies/TV. Re-downloadable via
  arr-stack; consistent with `KOPIA_BACKUP_STRATEGY.md`'s existing
  "replaceable" reasoning for media.
- **Why Kopia on top of SnapRAID parity at all, for the irreplaceable
  tier**: parity only protects against a single physical drive failing. It
  does not protect against accidental deletion, corruption/ransomware
  propagation (a bad sync can lock in a corrupted state as "correct"), or
  loss of the whole enclosure (fire, theft, surge — parity lives in the
  same physical box as the data it protects). Kopia's versioned snapshots,
  stored on a physically separate drive attached to a physically separate
  LXC (`kopia-server`), cover exactly the gap parity structurally can't.
- **`vm4-storage` runs its own Kopia client**, same pattern as
  `vm1-dmz`/`vm2-services` today — installs the client, connects to
  `kopia-server` with its own scoped credential
  (`vault_kopia_client_pw_vm4_storage`), with policies scoped only to the
  book/photo/Stash paths on the pool.
- **General `DOCKERCONFIGS_DIR`-equivalent backup is a separate open
  item**, not yet built for any host — noted here so it isn't conflated
  with this storage-VM work.

## Kopia clients — new dedicated inventory group

`playbook.yml`'s "Configure Kopia Backup Clients" play previously targeted
`docker_hosts`, which doesn't include `vm4-storage` (not a Docker host).
Rather than folding a non-Docker VM into `docker_hosts`, a new
`kopia_clients` group covers every host that needs a Kopia client
regardless of whether it runs Docker:

```ini
[kopia_clients:children]
docker_hosts
storage
```

This also means every host's Kopia backup paths are now expressed as a
**list** (path + per-path retention), not a single `kopia_backup_path`
string — needed because `vm4-storage` backs up three distinct paths with
one retention policy while movies/TV are deliberately excluded. See the
updated `host_vars/*.yml` files and `playbook.yml` patch.

## Migration from OMV — one-time manual step, not automated

Existing OMV data (single external USB drive) moves to its new role as the
mergerfs pool's fallback member via a one-time manual copy — not folded in
as a "4th equal member" with mismatched history, and not scripted, since
this only happens once:

1. Stand up `vm4-storage`, build the mergerfs pool (3 new DAS drives +
   the reused OMV drive, freshly wiped and re-added as the fallback
   branch) and SnapRAID parity first.
2. Copy old OMV data across (`rsync` or direct copy) into the correct
   pool categories before decommissioning the old OMV instance.
3. Decommission OMV once the copy is verified.

# Addendum to docs/STORAGE_VM_SETUP.md — Samba Access Model

Insert this as a new section, replacing the existing short "## Samba —
three role-based groups, not per-person ACLs" section (the model below
supersedes it with the actual reasoning worked through).

---

## Samba access model — per-host accounts, not per-stack or per-person

### The question that drove this

The instinct going in was to give each Docker stack (arrstack, stashstack,
mediastack) its own Samba account, on the theory that if `stash` changes
unexpectedly, the account that touched it tells you which stack did it —
real audit-trail value, same reasoning already applied to Kopia's
per-host scoped credentials.

### Why per-stack accounts don't actually deliver that

`vm2-services` and `vm3-internal` each mount the pool via **one CIFS
session, authenticated once, at the host level** — every container on that
host reaches the pool through a local bind-mount on top of that single
mount, not by individually authenticating to Samba itself. A Samba account
only does something if some connection actually presents its credential.
Since only one credential is ever presented per host, a `svc_stashstack`
account sitting in the vault alongside `svc_arrstack` would never actually
get used — there's no second CIFS connection for it to be the credential
for. Finer-grained accounts here aren't a security boundary, just unused
entries in the vault.

This also caps how precise the audit value can ever be, independent of
Samba entirely: every container across every stack runs under the same
`PUID`/`PGID` (1000:1000) today. Even with perfect per-container Samba
accounts, the filesystem itself wouldn't distinguish *which* container
wrote a file — there's no existing attribution signal underneath for a
finer Samba boundary to expose. (If per-container attribution is ever
wanted, the actual lever is giving each stack its own distinct PUID/GID —
a separate, smaller change, independent of anything here.)

### The boundary that *is* real: per-host

What the architecture actually enforces is one authenticated connection
per host. That's the real, usable boundary:

- A leaked `vm3-internal` Samba credential doesn't expose `vm2-services`'s
  access, and vice versa.
- If `stash` changes unexpectedly, `rw_stash` narrows the search to
  "something on `vm3-internal`" — not which specific container, but enough
  to know which host to go check logs on first, rather than guessing
  between both VMs or wondering about a direct LAN connection.

### Decision: two service accounts, one per host, each in the groups that
### host's stacks actually need

```yaml
samba_users:
  - { name: keith,           groups: [admin] }
  - { name: svc_vm2services, groups: [ro_media, rw_books, rw_dockerconfigs] }
  - { name: svc_vm3internal, groups: [rw_media, rw_stash, rw_dockerconfigs] }
```

### Stash's actual access pattern — corrected from an earlier draft

Stash doesn't organize/rename/import its own media the way Radarr/Sonarr
do — it scans and tags what's already there, writing only its own
thumbnails/metadata into its `DOCKERCONFIGS_DIR` slice. The write into the
Stash media library itself happens via **Whisparr's** import step (same
hardlink-on-same-branch pattern as Radarr/Sonarr), and Whisparr lives in
`arrstack`, not `stashstack`. So:

- `vm3-internal` (hosts both arrstack and stashstack) needs `rw_stash` —
  Whisparr's writes land here.
- Stash itself only ever needs read access to its own library — covered
  by the same `rw_stash` grant since it's host-level, not app-level, but
  worth knowing Stash's own container never actually writes there.

### `photos` and `emulation` — deliberately ungrouped for now

Both folders exist on disk3's branch layout as reserved space, but neither
has a deployed consumer (Immich, an emulation stack) yet. No group or ACL
entry exists for either — add one once a real service is standing up
against that path, rather than guessing at an access pattern for something
that doesn't exist yet.