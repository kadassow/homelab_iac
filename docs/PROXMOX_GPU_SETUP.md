# Proxmox GPU Sharing (SR-IOV) — Setup, Verification & Lessons Learned

Documents how the Proxmox host's Intel N150 iGPU is shared between `vm2-services`
(a VM, for Jellyfin QuickSync transcoding) and existing LXCs (which need no
special configuration at all). This is **host-level Proxmox configuration** —
it lives outside the `homelab_iac` Ansible inventory/playbook since Proxmox
itself isn't a managed node, so it's documented here instead so the reasoning
and steps aren't lost.

This supersedes two earlier drafts of this doc that were written across
separate sessions and drifted out of sync with each other (conflicting GRUB
flags, different DKMS clone paths, different VF counts). **This version
reflects the guide actually followed to build the current working setup** —
the "N150 iGPU SR-IOV Slicing Guide" approach — with the abandoned first
draft's ideas folded in only where they added something real (the single-VF
rationale, the LXC GID note).

---

## Why SR-IOV at all

`vm2-services` runs Jellyfin, which benefits significantly from Quick Sync
hardware transcoding instead of burning CPU cores on software transcoding —
important on an N150's limited core count if multiple streams transcode at
once. Several existing LXCs on the same host also do their own
GPU-accelerated work, and both need to run **at the same time**.

**Plain PCI passthrough doesn't work for this.** Per Proxmox's own docs: once
a device is passed through to a VM, it is unavailable to the host and every
other guest for as long as that VM is running — not just while it's actively
rendering something. There's no dynamic hand-back; it's a hard, exclusive
claim via `vfio-pci` for the VM's entire runtime. Community reports also show
this doesn't work reliably as a manual "take turns" setup either — shutting
down the VM and reassigning the device often requires a full host reboot
before it resets cleanly enough to be claimed again.

**LXCs and VMs need different treatment:**
- **LXCs share the host kernel.** Multiple LXCs can bind-mount the same
  `/dev/dri/renderD128` node concurrently and the driver handles multiple
  simultaneous opens fine — the same way multiple apps on a desktop OS share
  one GPU. **No SR-IOV, no changes needed, for existing LXCs.**
- **VMs run their own isolated kernel**, so they need something that looks
  like a dedicated PCI device — a Virtual Function (VF).

Since only `vm2-services` is a VM, SR-IOV is only needed to produce VF(s) for
it. LXCs keep using the physical function's render node directly, unchanged.

## Known risk with this hardware

The N150 (Twin Lake/Alder Lake-N) has mixed community results with
`i915-sriov-dkms` — some setups work cleanly on Proxmox 9, others report the
`sriov_numvfs` control file never appearing at all after driver install. The
project's own README describes it as experimental. Treat the verification
checkpoints below as hard gates — if a checkpoint fails, stop and reassess
rather than building further on a broken assumption.

**Also worth knowing up front, from direct experience:** a full host lockup
during VM boot with a GPU slice attached is **not automatically a GPU
problem**. Two unrelated issues on this exact setup each independently
produced a full, unresponsive host freeze that looked identical to a driver
crash:

1. **VM memory overcommit** — a VM configured for more RAM than the host
   physically has (or with a balloon minimum pinned too high) will make the
   host thrash and become unresponsive the moment it boots, GPU or no GPU.
2. **A hung host-level CIFS mount** — if the Proxmox host itself has an
   `/etc/fstab` entry pointing at network storage (see the open item on
   `lxc_shares` below) and that target becomes unreachable, kernel threads
   can block indefinitely on it, again producing a full freeze with no
   relation to the GPU.

**Before assuming a GPU/driver bug from a host lockup, rule both of these
out first** — see the Troubleshooting section at the end.

---

## Prerequisites (BIOS)

1. Enable VT-d / IOMMU (may be under a "Virtualization" or "Advanced CPU"
   BIOS menu).
2. Disable Secure Boot — the DKMS-built kernel module isn't signed, and
   Secure Boot will silently refuse to load it, or load it in a broken
   half-state. **Re-check this after any BIOS update** — some updates
   silently re-enable it.

---

## 1. Install the SR-IOV driver on the Proxmox host

```bash
apt update && apt install -y pve-headers-$(uname -r) sysfsutils git dkms build-essential
git clone https://github.com/strongtz/i915-sriov-dkms.git /usr/src/i915-sriov-dkms-<version>
cd /usr/src/i915-sriov-dkms-<version>
dkms add .
dkms build -m i915-sriov-dkms -v <version>
dkms install -m i915-sriov-dkms -v <version>
```

Replace `<version>` with the version string from the repo's `dkms.conf`
(`PACKAGE_VERSION=`). Verify with:

```bash
dkms status
```

This should show `i915-sriov-dkms/<version>, <kernel>: installed`, with
`<kernel>` matching `uname -r` **exactly**. This match is important enough
to call out on its own: **any Proxmox kernel update requires rebuilding this
module.** DKMS does not do this automatically. A mismatched module attempting
to bind to real GPU hardware is a known cause of host-level crashes with this
driver — check `dkms status` vs `uname -r` after every `pve-kernel` upgrade,
before you next reboot into a new kernel with the VM's GPU slice attached.

## 2. Set kernel boot parameters

Edit `/etc/default/grub`, set `GRUB_CMDLINE_LINUX_DEFAULT` to:

```
quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=<N> module_blacklist=xe video=efifb:off
```

- `intel_iommu=on iommu=pt` — enables IOMMU, passthrough mode for
  performance.
- `i915.enable_guc=3` — enables GuC/HuC firmware submission, required for
  QuickSync encode.
- `i915.max_vfs=<N>` — how many VFs to expose. **Set this to the number of
  VM consumers you actually have, not a ceiling "just in case."** GPU memory
  is split statically across configured VF slots regardless of whether a VF
  is attached to a running guest — over-provisioning this wastes GPU RAM for
  no benefit. If `vm2-services` is your only VM consumer, use `1`.
- `module_blacklist=xe` — some kernels default to the newer `xe` driver
  instead of `i915` for this GPU generation; blacklist it to force `i915`.
- `video=efifb:off` — forces headless boot, avoiding a host kernel panic
  when a VM claims a graphics slice. Only relevant if this Proxmox host runs
  fully headless (no monitor doing real work through the iGPU) — confirm
  before adding.

Apply:

```bash
update-grub
reboot
```

After reboot, confirm the parameters actually took effect (not just that the
file looks right):

```bash
cat /proc/cmdline
```

## 3. Automate VF activation (required for consumer iGPUs)

Intel's consumer chips (unlike enterprise cards) don't auto-instantiate VFs
from the GRUB parameter alone — GRUB just reserves the capability. A
`systemd-tmpfiles` rule is what actually triggers slice creation at boot:

```bash
cp /usr/src/i915-sriov-dkms-<version>/i915-set-sriov-numvfs.conf /etc/tmpfiles.d/i915-set-sriov-numvfs.conf
nano /etc/tmpfiles.d/i915-set-sriov-numvfs.conf
```

Uncomment the line at the bottom and set the trailing number to match your
`i915.max_vfs` value:

```
w /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs - - - - <N>
```

```bash
systemctl daemon-reload
systemctl enable i915-set-sriov-numvfs.service
reboot
```

**To change the VF count later**: update the trailing number here *and*
`i915.max_vfs` in GRUB, `update-grub`, reboot. Both must agree — a mismatch
between these two is a real source of instability at VF-activation time.

## 4. Verify VFs exist — CHECKPOINT (host only, no VM yet)

```bash
lspci | grep -E "VGA|Display"
```

Expect the physical function **plus one entry per configured VF** at
different PCI addresses (e.g. `00:02.0` physical, `00:02.1` VF1,
`00:02.2` VF2 if `max_vfs=2`).

```bash
cat /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs
```

Should return your configured VF count. **If this file doesn't exist at
all**, that's the known N100/N150 failure mode — stop here and reassess
rather than proceeding to attach anything to a VM.

**Reboot a second time and re-check both commands.** Confirming this
survives a second reboot (not just the first, right after a fresh module
build) rules out a fluke from the just-built module still being warm in
memory.

## 5. Create a Proxmox resource pool for the VF(s)

Abstracts the volatile PCI address behind a stable name.

1. **Datacenter → Resource Mappings → PCI Devices → Add**
2. Name: e.g. `N150-QSV-Pool`
3. Add device(s): select the VF address(es) — e.g. `0000:00:02.1` (and
   `.2` if using a second VF). **Never select the `.0` (physical function)
   here** — that's the host/LXC's device, not a VF.
4. Create.

## 6. VM hardware requirements

Before attaching the PCI device, confirm the VM has:

- **Machine type: `q35`** — required for correct PCIe topology; `i440fx`
  can't present the PCIe address space passthrough needs.
- **BIOS: OVMF (UEFI)** with an EFI Disk added.
- **Display: `none`** — leaving this at default causes Proxmox's web UI to
  attempt a VNC capture of the passed-through slice, hanging the browser
  console. (Note: this specific symptom is a **UI hang**, distinguishable
  from a full host lockup — if your keyboard on a directly-attached monitor
  also stops responding, that's not this issue; see Troubleshooting.)
- **Memory configured well within actual physical host RAM** — see the
  Troubleshooting section. Check with `qm config <vmid> | grep -i
  -E "memory|balloon"` and compare against `free -h` on the host, accounting
  for every other guest that might run concurrently.

Then, with the VM powered off:

1. **Hardware tab → Add → PCI Device**
2. Select **Mapped Device**, choose your resource pool (e.g.
   `N150-QSV-Pool`)
3. Check **PCI-Express** (only selectable once the VM is on `q35` — if
   greyed out, fix the machine type first)
4. Leave **All Functions** unchecked, **ROM-Bar** checked
5. Add, then start the VM

## 7. Guest-side setup (handled by Ansible)

`playbook.yml`'s QuickSync play installs `intel-media-va-driver-non-free` +
`vainfo` inside `vm2-services`, and checks for `/dev/dri/renderD128`,
warning if it's not yet present. This is guest OS driver setup only — it
doesn't create the VF itself, which must already exist per steps 1–6 above.

## 8. Existing LXCs — no changes needed

LXCs keep bind-mounting the host's physical-function render node directly,
exactly as before SR-IOV existed. If an LXC's GPU access ever breaks
**after** enabling SR-IOV, check the render group GID first — the SR-IOV
module can shift GID tracking on some setups:

```bash
getent group render   # on the host
```

Compare against the LXC's config (`/etc/pve/lxc/<vmid>.conf`):

```
dev0: /dev/dri/renderD128,gid=<matching-gid>,uid=0,mode=0660
```

If Jellyfin (or similar) inside an LXC specifically throws
`Error setting child device handle: -17` after this, switch that
container's hardware acceleration setting from QSV to **VAAPI** pointed at
`/dev/dri/renderD128`, with the low-power H.264/HEVC encoder options
enabled — Alder Lake-N/Twin Lake needs the low-power execution path for
VAAPI specifically.

---

## Verification checklist — confirm it's actually working end to end

Don't stop at "the device exists" — confirm it's functionally usable.

1. **VF visible inside the VM**: `lspci | grep -i vga` — should show an
   Intel VGA/display controller (the VF), separate from what the host sees.
2. **Render node exists**: `ls -la /dev/dri/` inside the VM — expect
   `renderD128`. If missing, check `dmesg | grep i915` inside the guest for
   binding errors.
3. **Ansible agrees**: re-run
   `ansible-playbook playbook.yml -i inventory.ini --limit vm2-services`
   — the "iGPU hasn't been passed through" warning should no longer fire.
4. **Codecs actually enumerate**: `vainfo` inside the VM — must list
   `VAProfileH264Main`, `VAProfileHEVCMain`, etc. with working entrypoints.
   A device node existing with `vainfo` showing zero profiles means the
   node is present but the driver stack underneath isn't functional
   (commonly a GuC/HuC firmware issue).
5. **`group_add` GID still correct**: `getent group render` inside the VM,
   compare to `group_add` in `mediastack/docker-compose.yml`'s `jellyfin`
   service — a VM rebuild can shift this GID silently.
6. **Real transcode test**: Jellyfin Dashboard → Playback → Transcoding →
   Hardware acceleration = Intel QuickSync (QSV), enable needed codecs. Play
   something that forces a transcode (e.g. 4K source to a 1080p-limited
   device). Confirm via `docker exec -it jellyfin ps aux` that the running
   `ffmpeg` process shows `-hwaccel qsv` — this is the actual proof, not
   just that playback works (software fallback also "works," just slower
   and on CPU).

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Only the Proxmox **web UI/console** freezes; SSH and physical keyboard still work | VM's Display setting isn't `none` — web UI attempting VNC capture of the passed-through slice | Set Display to `none` on the VM's Hardware tab |
| **Entire host** unresponsive — no SSH, no ping, physical keyboard on attached monitor also dead | Real kernel-level issue. Check the two items below before suspecting the GPU driver itself | See next two rows |
| Host thrashes/freezes right as the VM boots; `free -h` shows memory collapsing toward zero, swap filling | **VM memory overcommit** — configured VM memory (check `qm config <vmid> \| grep memory`) exceeds actual host RAM, or a balloon minimum is pinned too high, once other running guests are counted | Reduce the VM's configured memory to something that actually fits alongside everything else; if using ballooning, don't set the minimum equal to the maximum — confirm `qemu-guest-agent` is running in-guest so ballooning can respond to real pressure |
| Host freezes some time after a network target (e.g. OMV) goes down or reboots; `dmesg`/journal shows `CIFS: VFS: \\<ip> has not responded in N seconds` before the freeze | A **host-level** CIFS mount (not a guest's) hung on an unreachable target — default `hard` mount behavior retries indefinitely and can block kernel threads | Add `soft` to the fstab mount options (do **not** use `timeo=`/`retrans=` — those are NFS options, not valid for `cifs.ko`, and will cause every mount to fail with `Unknown parameter`); use `echo_interval=<seconds>` if you want to tune keepalive/reconnect timing |
| `mount -a` fails with `Couldn't chdir to <path>: No such device` on every CIFS share, even ones confirmed to exist via `smbclient -L` | Almost always a **mount option typo** being passed straight to `cifs.ko`, which rejects it — check `dmesg \| tail` for `cifs: Unknown parameter '<name>'`. Common one: `x-systemd.mount-timeout=30` accidentally written as `s-systemd.mount-timeout-30` (wrong prefix, wrong separator) | Fix the exact option syntax in `/etc/fstab`, then `mount -a` again |
| `sriov_numvfs` file doesn't exist under `/sys/devices/pci.../0000:00:02.0/` | Known N100/N150 failure mode — SR-IOV not exposed for this exact kernel/firmware combo | Stop and reassess; check for a project GitHub issue matching your exact `uname -r` before sinking more time in |
| `vainfo` shows zero profiles despite the render node existing | GuC/HuC firmware missing/mismatched for this silicon, or DKMS module built against a different kernel than the one running | Confirm `dkms status` kernel matches `uname -r` exactly; rebuild against the current kernel if not |
| DKMS module fails to load after a Proxmox kernel update | Module wasn't rebuilt against the new kernel — DKMS doesn't do this automatically | `apt install pve-headers-$(uname -r)` then rebuild/reinstall the DKMS module explicitly for the new kernel version |
| Module loads but crashes/won't bind | Secure Boot re-enabled (e.g. by a BIOS update) | Re-verify `mokutil --sb-state` or BIOS setting; disable and reboot |
| LXC's Jellyfin/transcoding breaks specifically after enabling SR-IOV | `render` group GID shifted | `getent group render` on host, update the LXC's `.conf` `dev0:` line to match |

---

## Status

- ✅ SR-IOV driver installed and VFs confirmed present on the host
- ✅ One VF attached to `vm2-services`, host confirmed to remain stable
  through VM boot (after resolving the memory-overcommit and CIFS issues
  below — neither was GPU-related)
- ⬜ Full verification checklist (section above) — confirm and check off
  each item once run for real
- ⬜ Jellyfin QSV setting applied and a real hardware transcode confirmed
  via `ffmpeg -hwaccel qsv` in the running process list

## Open items / future considerations

- **`lxc_shares` circular dependency (flagged, not yet resolved)**: the
  Proxmox host itself mounts OMV CIFS shares at `/mnt/lxc_shares/*` via
  `/etc/fstab`, originally to make passing shares into unprivileged LXCs
  easier. This makes the **hypervisor's own stability depend on an external
  device (OMV)** that it should be able to survive without — a genuine
  single point of failure, and the direct cause of one of the two false
  "GPU is broken" leads chased during this setup. Options to revisit:
  - Move the CIFS mount into whichever specific LXC actually needs that
    share, rather than mounting it at the host level at all (requires
    `mount.cifs` available in that LXC, and typically
    `features: mount=cifs` set, or running that one container privileged).
  - If a host-level mount is kept for anything, ensure `soft` +
    `x-systemd.automount` + a sane `mount-timeout` are set (see
    Troubleshooting table) so an unreachable target degrades to an I/O
    error instead of an indefinite hang.
- **VF count right-sizing**: currently running `i915.max_vfs=2` (carried
  over from the guide followed) with only one VF actually attached to a
  running VM. Per the GPU-RAM-is-split-statically caveat noted above,
  consider dropping to `max_vfs=1` to reclaim that reservation, unless a
  second VM consumer is planned soon.
- **Re-verify after every Proxmox kernel update**: `dkms status` vs
  `uname -r`, and the full verification checklist, since this driver isn't
  officially supported and a kernel update is the most likely thing to
  quietly break the binding.