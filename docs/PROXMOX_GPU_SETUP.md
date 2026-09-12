# Proxmox GPU Sharing (SR-IOV) — Setup & Decision Notes

Documents how the Proxmox host's iGPU is shared across `vm2-services` and
existing LXCs for hardware-accelerated video transcoding (Jellyfin, and any
LXC-based transcoding workloads like Frigate/Plex). This is **host-level
Proxmox configuration** — it lives outside the `homelab_iac` Ansible
inventory/playbook since Proxmox itself isn't a managed node, so it's
documented here instead so the reasoning and steps aren't lost.



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

Slices dynamically adjust gpu time, however share gpu ram statically.  if you have 4gb ram and 4 slices, each slice will only support 1gb ram each.  Don't make the max and hope for the best performance.  Scale the slices to your current needs.
---

## Setup steps

### Prerequisites (BIOS)

1. Enable VT-d / IOMMU (may be under a "Virtualization" or "Advanced CPU"
   BIOS menu).
2. Disable Secure Boot — the DKMS-built kernel module isn't signed, and
   Secure Boot will silently refuse to load it.

### 1. Install build tools and the SR-IOV driver on the Proxmox host
To find out exactly which bootloader your Proxmox installation is using, run this command in your host 
```bash
[ -d /sys/firmware/efi ] && [ -d /pve-boot-esm ] || [ -f /etc/kernel/cmdline ] && echo "systemd-boot" || echo "GRUB"
** I was seeing GRUB as a result of this.

```bash
apt update
apt install -y dkms build-essential pve-headers-$(uname -r) sysfsutils git

``` this package assume install into /usr/src
cd /usr/src
git clone https://github.com/strongtz/i915-sriov-dkms.git

``` Enter the folder
cd i915-sriov-dkms

``` Read the version string inside the repository configuration to register it with DKMS
DKMS_VER=$(grep "PACKAGE_VERSION=" dkms.conf | cut -d'"' -f2)
``` Move the code into the proper system DKMS directory format
cd ..
mv i915-sriov-dkms i915-${DKMS_VER}

``` Tell DKMS to track, build, and load the new driver
dkms add -m i915 -v ${DKMS_VER}
dkms build -m i915 -v ${DKMS_VER}
dkms install -m i915 -v ${DKMS_VER}
```

If `dkms install` fails looking for headers, confirm `pve-headers-$(uname -r)`
actually matches the running kernel — version drift here is the most common
failure.

### 2. Set kernel boot parameters

Edit `/etc/default/grub`, add to `GRUB_CMDLINE_LINUX_DEFAULT`:

```
intel_iommu=on i915.enable_guc=3 i915.max_vfs=7
```

** I set this to max_vfs=1 for now as I have only 1 vm that needs transcode.  This can be increased in the future.

(`max_vfs=7` is the ceiling most guides use for this GPU generation — we
only need one VF in practice, since `vm2-services` is the sole VM consumer.)

Then:

```bash
update-grub
update-initramfs -u
reboot
```

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

### 3b - n150 consumer igpu problem
The repository contains an automation file called i915-set-sriov-numvfs.conf. This file creates a system background daemon (systemd template) that looks at your GRUB configuration, calculates how many slices you want, and automatically handles the echo command for you.To set up this single-point-of-control automation, run these commands in your Proxmox shell:
Why GRUB isn't enough for Intel Consumer iGPUsOn enterprise server GPUs (like an NVIDIA Tesla or Intel Flex card), the driver natively reads the GRUB command (i915.max_vfs=1) and automatically instantiates the virtual slices during boot.However, because the Intel N150 is a consumer desktop/mobile chip, Intel intentionally omitted the automation code that triggers the slices at boot.grub / i915-sriov.conf: These files simply reserve the hardware resources in the system kernel driver loop. They tell the driver: "Get ready, we might want to split this card into 1 slice later."sysfs.conf: This is the file that actually presses the start button. It executes the literal hardware command (echo 1 > .../sriov_numvfs) the moment Proxmox finishes loading. Without this file, the driver stays in standby mode, and your slices are never created.

``` 1. Copy the repository's automation script into your system services folder
cp /usr/src/i915-sriov-dkms-2026.08.12.1/i915-set-sriov-numvfs.conf /etc/tmpfiles.d/i915-set-sriov-numvfs.conf

nano /etc/tmpfiles.d/i915-set-sriov-numvfs.conf
Scroll to the bottom of the file. You will see a line that looks like this: 
  #w /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs - - - - 1
Uncomment it by deleting the # symbol at the very front of the line.Ensure the trailing number matches the slice count you want (it defaults to 1)

``` 2. Reload the system services manager to recognize it
systemctl daemon-reload

``` 3. Enable the service so it runs automatically every time Proxmox boots
systemctl enable i915-set-sriov-numvfs.service

From this point forward, if you ever want to increase your pool from 1 slice to 3 slices in the future:
Open your /etc/tmpfiles.d/i915-set-sriov-numvfs.conf 
 swap the trailing 1 to a 3.
Open your /etc/default/grub file 
  swap max_vfs=1 to max_vfs=3.
  Run  update-grub
  run reboot

### 4a. Resource mapping
Resource mapping abstract away the physical resources into a pool.  as long as the pool has resources to assign you can associate the pool to many VMs.  If your pool has 3 slices, you can have 3 vm using the pool.
Step 1: Create the PCI Resource Pool in ProxmoxOpen your browser and log into your Proxmox Web UI.
Click on Datacenter at the very top of the left-hand menu tree.
Select Resource Mappings (located under the Options section).
Click Add at the top of the PCI Devices section.
Fill out the configuration window exactly like this:
  Name: N150-QSV-Pool (No spaces allowed here).
  Description: Intel N150 iGPU SR-IOV Transcode Slices
  Look for the Devices table inside that same window and click Add.
    In the Device dropdown menu, look for and select your virtual function slice: 0000:00:02.1 (Do not select .0 variant, this is the main gpu)
    (It will likely be labeled Intel Corporation Alder Lake-N [Intel Graphics]).
    Click Create at the bottom to save the pool.


### 4b. Assign one VF to `vm2-services`

Step 2: Add the Mapped Device to your VM
Now we will link that pool to your target transcoding Virtual Machine.
In the Proxmox left menu, click on your target VM (ensure the VM is currently turned off).
Go to the Hardware tab.
Click the Add dropdown button at the top and select PCI Device.
In the window that pops up, change the selection bubble at the top from Raw Device to Mapped Device.
Click the Mapped Device dropdown menu and select the pool we just made: N150-QSV-Pool.
Crucial Settings to Check:
  Check the box for PCI-Express.
  Leave All Functions unchecked.
  Leave ROM-Bar checked.
  Click Add.

#### PCI-Express disabled
Why it is grayed out:
Your VM is likely configured with the older legacy i440fx hardware machine type, which simulates an old 1990s desktop motherboard that only has standard PCI slots (no PCI-Express). To enable the PCIe checkbox, your VM must be running the modern q35 chipset type. 
How to fix it and enable the checkbox:
Keep the VM turned off.
In the Proxmox Web UI, click on your VM, then go to the Hardware tab.
Look for the row named Machine (it probably says pc-i440fx-...).
Select it, click Edit, and change it to q35 (e.g., q35-run-latest or q35-8.x).
 Click OK.
 Note: If you change this, your VM should ideally be using OVMF (UEFI) for its BIOS instead of SeaBIOS. 
 Once the machine type is upgraded to q35, go back into Add -> PCI Device -> Mapped Device, choose your pool, and the PCI-Express checkbox will be fully enabled and clickable! Check it and click Add

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




* *******************  These may be different approach **********************
# Intel N150 iGPU SR-IOV Slicing Guide for Proxmox VE 8/9

This guide outlines the complete configuration required to share an **Intel Processor N150 (Alder Lake-N / Twin Lake)** integrated GPU across **multiple Virtual Machines (VMs)** simultaneously using **SR-IOV**, while preserving native hardware transcoding access for **Linux Containers (LXCs)** like Jellyfin.

---

## 1. Proxmox Host Configuration

### Step 1: Core Prerequisites & GRUB Setup
The Intel N150 requires explicit boot loader flags to enable IOMMU virtualization and expose virtual functions. It also requires forcing a **headless boot** (`video=efifb:off`) to prevent host kernel panics when a VM intercepts a graphics slice.

1. Open the host GRUB configuration file:
   ```bash
   nano /etc/default/grub
   ```
2. Modify the line `GRUB_CMDLINE_LINUX_DEFAULT` to match this string precisely:
   ```text
   GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=2 module_blacklist=xe video=efifb:off"
   ```
3. Update the boot records:
   ```bash
   update-grub
   ```

### Step 2: Build & Install the SR-IOV Kernel Module via DKMS
Because native Intel consumer iGPU SR-IOV drivers are absent from upstream Linux kernels, compile the community backport module:

1. Install development dependencies:
   ```bash
   apt update && apt install -y pve-headers sysfsutils git dkms build-essential
   ```
2. Clone the official repository directly into the system source tree:
   ```bash
   git clone https://github.com/strongtz/i915-sriov-dkms.git /usr/src/i915-sriov-dkms-2026.08.12.1
   ```
3. Register, build, and install the module entry inside the DKMS tree:
   ```bash
   cd /usr/src/i915-sriov-dkms-2026.08.12.1
   dkms add .
   dkms build -m i915-sriov-dkms -v 2026.08.12.1
   dkms install -m i915-sriov-dkms -v 2026.08.12.1
   ```
4. Verify deployment (`dkms status` should output `installed`).

### Step 3: Automate Virtual Function Activation via systemd-tmpfiles
Intel consumer chips require an explicit write to sysfs to physically spawn slices right after booting finishes.

1. Copy the repository's configuration profile to the active system folder:
   ```bash
   cp /usr/src/i915-sriov-dkms-2026.08.12.1/i915-set-sriov-numvfs.conf /etc/tmpfiles.d/i915-set-sriov-numvfs.conf
   ```
2. Open the file:
   ```bash
   nano /etc/tmpfiles.d/i915-set-sriov-numvfs.conf
   ```
3. Uncomment or add the creation string pointing to your intended target value (`2` slices):
   ```text
   w /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs - - - - 2
   ```
4. **Reboot the Proxmox Host** (`reboot`). Note that your physical monitor will go blank mid-boot; this confirms the headless setup is functional.

### Step 4: Verify Your Slices
Once rebooted, inspect the local PCIe hardware table layout:
```bash
lspci | grep -E "VGA|Display"
```
You must see three records:
* `00:02.0` — Physical Function (PF) / Master Host Card
* `00:02.1` — Virtual Function (VF) Slice 1
* `00:02.2` — Virtual Function (VF) Slice 2

---

## 2. Proxmox Web UI & Resource Mapping

To avoid hardcoding volatile hardware physical addresses directly inside VMs, use a virtual abstraction pool.

1. Go to **Datacenter** -> **Resource Mappings** -> **PCI Devices** and click **Add**.
2. **Name:** `N150-VM-Slices`
3. Under the **Devices** list, add both **`0000:00:02.1`** and **`0000:00:02.2`**. Do **NOT** add `.0`.
4. Click **Create**.

---

## 3. Virtual Machine (VM) Configuration

For any target VM intended to use a slice (e.g., Tdarr or a media server VM):

### Hardware Settings Alignment
* **Machine:** Must be set to **`q35`** (Standard `i440fx` does not support PCIe mapping address space requirements).
* **BIOS:** Must be set to **`OVMF (UEFI)`** (SeaBIOS lacks large 64-bit BAR addressing room and causes Code 10/43 driver failures).
* **EFI Disk:** You must add an **EFI Disk** to store the UEFI parameters permanently.
* **vIOMMU:** Leave as **`Default (None)`**.
* **Display:** Change to **`none`** (Headless). Leaving it as default forces the Proxmox Web UI to attempt a VNC canvas capture of the physical slice, resulting in browser interface lockups and endless console spin wheels.

### Adding the Device
1. Click **Add** -> **PCI Device**.
2. Select **Mapped Device** and pick **`N150-VM-Slices`**.
3. **Check the box** for **PCI-Express**.
4. Save and start. Manage the VM externally via network layers (SSH/RDP).

---

## 4. Linux Container (LXC) Configuration

To safely preserve native, shared iGPU hardware transcoding for unprivileged application containers (like Jellyfin) alongside active VM slices:

### Step 1: Mapping the Master Hardware via GUI
1. Select your **LXC Container** -> **Resources** tab.
2. Click **Add** -> **Device Passthrough**.
3. Set the **Path** to: `/dev/dri/renderD128`
4. Leave the Mode as `0666` and save.

### Step 2: Harmonizing Security Group IDs
Because the host SR-IOV module driver shifts GID tracking, you must explicitly match the file permissions inside the container configuration file.

1. On the **Proxmox Host shell**, inspect your local rendering group number:
   ```bash
   getent group render | cut -d: -f3
   ```
   *(For modern Proxmox nodes, this value is usually **`993`**).*
2. Open the raw configuration layout file for your target container (e.g., ID 100):
   ```bash
   nano /etc/pve/lxc/100.conf
   ```
3. Locate the automatically generated passthrough reference at the bottom of the file and append your exact GID number:
   ```text
   dev0: /dev/dri/renderD128,gid=993,uid=0,mode=0660
   ```
   *(Ensure any legacy line pointing to `/dev/dri/card0` is completely **deleted** out of the file to prevent LXC boot errors).*
4. Save and exit.

### Step 3: Application Level Alignment (Jellyfin UI)
Because an unprivileged container cannot pass native system display context parameters across a secure kernel border, picking standard Intel QuickSync (QSV) inside Jellyfin will return an `Error setting child device handle: -17` crash loop.

1. Open your **Jellyfin Dashboard** -> **Playback**.
2. Set **Hardware Acceleration** to **`VAAPI`**.
3. Set **VAAPI Device** path field explicitly to: `/dev/dri/renderD128`
4. **Check the boxes** for **Enable Intel Low-Power H.264 hardware encoder** and **HEVC**. Alder Lake-N / Twin Lake architectures require low-power execution loops to run properly over VAAPI.
5. Click **Save**.