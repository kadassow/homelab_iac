# Kopia Restore Procedure

Companion to `KOPIA_BACKUP_STRATEGY.md`, which covers why Kopia was chosen
and how the server/client topology is set up. This doc covers the other
half: **how to actually restore** when a host gets reinstalled — written
after doing this for real on `vm2-services`, including two mistakes made
along the way worth not repeating.

## When you'd use this

- A VM or LXC was reinstalled from scratch (OS wipe, hardware failure,
  testing the rebuild process itself) and needs its Kopia-backed data back.
- You need to roll back to an earlier point in time after a bad config
  change or accidental deletion, without a full OS reinstall.

This doc assumes the host is already reconnected to the Kopia server as a
client (see `KOPIA_BACKUP_STRATEGY.md` / `playbook.yml`'s "Configure Kopia
Backup Clients" play) — if not, do that first.

---

## The ordering gotcha, if you're restoring after a full reinstall

`playbook.yml`'s plays run in this order:

1. Standardize hosts
2. Install Docker
3. QuickSync setup
4. Configure Core VM Storage — creates `LOCALDOCKER_DIR` **empty**
5. **Deploy mediastack** — creates empty subdirs, renders `.env`, runs
   `docker compose up -d` — containers boot with brand-new empty configs
6. Install Kopia Backup Server (backup group only)
7. **Configure Kopia Backup Clients** — this host reconnects to Kopia here

Play 5 runs before play 7, so a full `ansible-playbook playbook.yml` run
against a freshly reinstalled host will boot Jellyfin/NutriTrace with empty
SQLite DBs *before* Kopia is even reconnected. That's fine — don't try to
reorder the playbook for this. Just let it run once, then follow the
restore steps below, which stop the stacks before overwriting anything.
Dropping restored files under a container that already has that file open
is the actual risk to avoid, not the empty-boot itself.

---

## Restore steps

### 1. Confirm the client is connected and can see its snapshots

```bash
kopia repository status
```
Confirms which client identity you're connected as (should be
`<hostname>@<hostname>`, e.g. `vm2-services@vm2-services`) — run this from
**the same host** whose data you're restoring, not the Kopia server LXC.

### 2. List available snapshots for the path you need

```bash
kopia snapshot list /localdocker/configs
```

Example output:
```
2026-09-09 02:30:01 CDT kf4c9d762c7da93ab48244deeb203abf 2.2 MB drwxr-xr-x files:47 dirs:34 (latest-3,hourly-3)
2026-09-09 08:32:05 CDT k40efb843daedc7d5134433f2e2d88325 2.2 MB drwxr-xr-x files:48 dirs:34 (latest-2,hourly-2,daily-2)
2026-09-10 02:30:32 CDT ka164d9de59f82118a87a3c7969733aeb 2.8 MB drwxr-xr-x files:49 dirs:34 (latest-1,hourly-1,daily-1,weekly-1,monthly-1,annual-1)
```
Most recent is generally the one tagged `latest-1` — usually what you want
unless you have a specific reason to roll back further (e.g. the most
recent snapshot captured a bad state you're trying to avoid restoring).

**⚠️ Copy the full ID, not a shortened version.** Unlike some other Kopia
subcommands, `kopia snapshot restore` needs the complete ID string
(`k40efb843daedc7d5134433f2e2d88325`, not `k40efb84`) — a truncated prefix
fails with `no snapshots contain data for that id`, even though the prefix
looks unambiguous. Copy directly from a fresh `list` output rather than
retyping or reusing an ID from memory/notes.

### 3. Stop the stack(s) before restoring

Don't restore into a directory a running container has open — this risks
corruption or a restore that gets silently overwritten by the still-running
container.

```bash
cd /opt/stacks/mediastack && docker compose down
```
Repeat for `arrstack`/`stashstack` if applicable (still hand-run outside
Ansible as of this writing — same `docker compose down` in wherever you
manually placed them).

### 4. Restore, overwriting the empty scaffold directories Ansible created

```bash
kopia snapshot restore <full-snapshot-id> /localdocker/configs --overwrite-files --overwrite-directories
```

Both `--overwrite-files` and `--overwrite-directories` are required —
without them, Kopia refuses to write into a non-empty target, and the
Ansible-deployed stack already populated `/localdocker/configs` with empty
subdirectories before you got here.

**If you get `Restoring to local filesystem ... unable to initialize output`
immediately after running this:** check for a stray trailing space after a
line-continuation backslash if you split the command across lines — a
misplaced space silently breaks the continuation, so the command runs
*without* the overwrite flags on the next line, and Kopia refuses to write.
Safest fix: keep the whole command on one line.

### 5. Spot-check ownership after restore

```bash
ls -la /localdocker/configs
```
Kopia generally preserves the UID/GID it snapshotted with, but confirm it
matches `PUID`/`PGID` (1000/1000 per `group_vars/all/vars.yml`) before
restarting containers — a mismatch here shows up as a confusing
"permission denied" on next boot, easy to misdiagnose as something else.

### 6. Bring the stack(s) back up

```bash
cd /opt/stacks/mediastack && docker compose up -d
```

Confirm Jellyfin/NutriTrace/etc. come up with the restored data (watch
history, library state, NutriTrace entries) rather than a fresh first-run
state — that's the real confirmation the restore worked, not just that the
command exited without error.

---

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `unable to initialize output` right after running restore | Line-continuation backslash had trailing whitespace, silently dropping the overwrite flags from the command | Put the whole command on one line |
| `no snapshots contain data for that id` | Used a truncated/shortened snapshot ID | Copy the **full** ID string directly from `kopia snapshot list` output |
| Restore succeeds but containers show permission errors on boot | UID/GID mismatch between snapshot and `PUID`/`PGID` | `chown -R 1000:1000 /localdocker/configs` (adjust if your PUID/PGID differ), then restart the stack |
| `kopia snapshot list` shows nothing | Wrong host/client identity, or wrong path | Run `kopia repository status` to confirm which client you're connected as; confirm the path matches `LOCALDOCKER_DIR` exactly (`/localdocker/configs`) |

---

## Testing this against a cloud-init rebuild

Since Path B (cloud-init) in `VM2_SERVICES_CLEAN_REBUILD_CHECKLIST.md`
reaches a working, Ansible-ready VM faster than the ISO installer path, it's
a good way to repeatedly validate this entire restore procedure end to end
without the SSH/networking friction from earlier rebuild attempts. Suggested
loop for testing:

1. Clone from the cloud-init template (`VM2_SERVICES_CLEAN_REBUILD_CHECKLIST.md`
   Section B2).
2. Run the full `ansible-playbook playbook.yml` (accepting that mediastack
   comes up empty first, per the ordering gotcha above).
3. Follow Sections 1–6 of this doc to restore.
4. Confirm actual data (not just file counts) came back correctly —
   Jellyfin watch history, NutriTrace entries, etc.
5. If anything breaks, that's a real gap in either the playbook's Kopia
   client play or this restore doc — worth fixing here rather than treating
   it as a one-off surprise on a real future rebuild.
