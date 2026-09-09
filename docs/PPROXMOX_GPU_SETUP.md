# Proxmox GPU Sharing (SR-IOV) — Setup & Decision Notes

Documents how the Proxmox host's iGPU is shared across `vm2-services` and
existing LXCs for hardware-accelerated video transcoding (Jellyfin, and any
LXC-based transcoding workloads like Frigate/Plex). This is **host-level
Proxmox configuration** — it lives outside the `homelab_iac` Ansible
inventory/playbook since Proxmox itself isn't a managed node, so it's
documented here instead so the reasoning and steps aren't lost.

## Hardware

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

## Why we needed to share the GPU at all

`vm2-services` runs Jellyfin, which benefits significantly from Quick Sync
hardware transcoding instead of burning CPU cores on software transcoding —
important on an N150's limited core count if multiple streams transcode at
once. Separately, several existing LXCs on the same host already do their own
GPU-accelerated work. Both needed to run **at the same time**, which ruled
out the simplest option.

## Why we rejected plain PCI passthrough

The initial assumption was that Proxmox could hand the GPU to a VM and
"release" it back to the host/other guests whenever the VM wasn't actively
using it. **This is not how PCI passthrough works.** Per Proxmox's own docs:
once a device is passed through to a VM, it is unavailable to the host and
every other guest for as long as that VM is running — not just while it's
actively rendering something. There's no dynamic hand-back; it's a hard,
exclusive claim via `vfio-pci` for the VM's entire runtime.

Community reports also show this doesn't even work reliably as a manual
"take turns" setup — shutting down the VM and trying to reassign the same
physical device to another guest often requires a full host reboot before
the device resets cleanly enough to be claimed again. It's not fit for a
scenario where a VM and multiple LXCs all need concurrent access.

## The two guest types behave differently, which shaped the final approach

- **LXCs share the host kernel.** They don't need any GPU splitting at all —
  multiple LXCs can bind-mount the same `/dev/dri/renderD128` node
  concurrently and the driver handles multiple simultaneous opens fine, the
  same way multiple apps on a desktop OS share one GPU. **No changes needed
  for existing LXCs.**
- **VMs run their own isolated kernel**, so they can't share a device node
  the way LXCs do — a VM needs something that looks like a dedicated PCI
  device handed to it.

Because only `vm2-services` is a VM, and every other GPU consumer on this
host is an LXC, **SR-IOV is only needed to produce one virtual function (VF)
for `vm2-services`.** LXCs continue using the physical function's render
node directly, unchanged.

## Known risk before starting

The N150 (Twin Lake/Alder Lake-N) has had **mixed community results** with
`i915-sriov-dkms` — some setups work cleanly on Proxmox 9, others (including
other N100/N150 users) report the `sriov_numvfs` control file not appearing
at all after driver install, meaning SR-IOV isn't currently exposed for
their exact kernel/firmware combo. Treat the verification step below as a
hard checkpoint — if VFs don't appear, stop and reassess rather than
continuing to build on top of a broken assumption.

---

## Setup steps

### Prerequisites (BIOS)

1. Enable VT-d / IOMMU (may be under a "Virtualization" or "Advanced CPU"
   BIOS menu).
2. Disable Secure Boot — the DKMS-built kernel module isn't signed, and
   Secure Boot will silently refuse to load it.

### 1. Install build tools and the SR-IOV driver on the Proxmox host

```bash
apt update
apt install -y dkms build-essential pve-headers-$(uname -r) sysfsutils git

cd /opt
git clone https://github.com/strongtz/i915-sriov-dkms.git
cd i915-sriov-dkms
dkms add .
dkms install -m i915-sriov-dkms -v $(cat VERSION) -k $(uname -r) --force
```

If `dkms install` fails looking for headers, confirm `pve-headers-$(uname -r)`
actually matches the running kernel — version drift here is the most common
failure.

### 2. Set kernel boot parameters

Edit `/etc/default/grub`, add to `GRUB_CMDLINE_LINUX_DEFAULT`:

```
intel_iommu=on i915.enable_guc=3 i915.max_vfs=7
```

Then:

```bash
update-grub
update-initramfs -u
reboot
```

(`max_vfs=7` is the ceiling most guides use for this GPU generation — we
only need one VF in practice, since `vm2-services` is the sole VM consumer.)

### 3. Verify VFs were created — CHECKPOINT

```bash
lspci | grep VGA
```

Expect the physical GPU **plus additional VF entries** at different PCI
function addresses (e.g. `00:02.0`, `00:02.1`, `00:02.2`...).

```bash
dmesg | grep i915
```

If VFs don't show up, check whether the control file exists at all:

```bash
cat /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs
```

If this file is missing, that's the known N100/N150 failure mode reported on
Proxmox 9 — stop here and reassess rather than proceeding further.

Optional persistence belt-and-suspenders alongside the GRUB param:

```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.conf
```

### 4. Assign one VF to `vm2-services`

Proxmox web UI → `vm2-services` → **Hardware** → **Add** → **PCI Device** →
select a VF address (e.g. `0000:00:02.1`) — **not** the physical GPU
(`00:02.0`). Leave "All Functions" unchecked. No ROM-bar / primary-GPU flags
needed since this isn't display passthrough, just a compute device.

### 5. Existing LXCs — no changes

Continue bind-mounting the host's `/dev/dri/renderD128` (the physical
function's render node) exactly as already configured. SR-IOV on the host
doesn't affect this path.

### 6. Guest-side (inside `vm2-services`)

Handled by `playbook.yml` (see main repo) — installs
`intel-media-va-driver-non-free` + `vainfo`, checks for
`/dev/dri/renderD128` inside the VM, and warns if the VF hasn't landed yet.

If `vainfo` inside the guest shows no profiles even after the VF is
attached, the fallback some guides report needing is installing
`i915-sriov-dkms` **inside the guest VM too** (not just the host) — try the
standard driver first; this is a secondary troubleshooting step, not
expected to be necessary in most cases.

### 7. Jellyfin app setting (manual, one-time, post-deploy)

Dashboard → Playback → Transcoding → set hardware acceleration to **Intel
QuickSync (QSV)**. This lives in Jellyfin's own config DB, not an env var —
no way to automate this via Ansible/compose.

---

## Status

- ⬜ Not yet started on the physical host — this doc reflects the plan and
  reasoning; steps above haven't been executed yet.

## Open questions / future considerations

- If SR-IOV turns out unsupported on this exact N150/kernel combo (see
  "Known risk" above), fallback options to revisit: software transcoding
  only (accept the CPU cost), or moving Jellyfin's GPU needs to LXC-only
  access patterns if the VM requirement can be relaxed.
