# vm2-services Clean Rebuild Checklist

Consolidated from a full reinstall troubleshooting session. Follow top to
bottom in order — several steps here exist specifically because doing them
out of order (or skipping them) is what caused problems last time.

**Two paths are documented below:**

- **Path A — ISO Installer**: the traditional interactive Debian install.
  More manual steps, more places to hit the issues from this session, but
  more familiar/visible if you want to see and control every step.
- **Path B — Cloud-Init**: build one reusable template, then every future
  VM (this rebuild included) clones from it with network, hostname, and SSH
  keys injected automatically at first boot — no installer screens at all.
  Eliminates nearly every issue hit this session (empty DNS, wrong-user SSH
  key, `sudo` missing) by construction rather than by remembering to fix
  each one.

Both paths converge at **Section 9** (manual SSH test) and share
everything from there through Ansible connectivity. Path A additionally
clears the stale SSH host key early (**Section 3**), before any SSH
connection is attempted — since editing `sshd_config` and adding root's key
may themselves happen over SSH from your control node, not just via the
Proxmox console.

---

## 0. Proxmox VM Creation Settings

Decide these *before* attaching the install ISO — some (BIOS type
especially) are painful to change after the OS is installed.

- **BIOS: OVMF (UEFI)**, not the SeaBIOS default. SeaBIOS's 32-bit-only PCI
  address space can't map large BARs on modern discrete GPUs — not actually
  a constraint for this VM's Quick Sync VF (compute-only, no display
  passthrough, small BAR), but OVMF is the modern Proxmox-recommended
  default regardless, and switching later requires a full OS reinstall
  anyway. Requires adding an **EFI Disk** (tiny, a few hundred KB) on
  System tab creation — Proxmox prompts for this automatically once OVMF
  is selected.
- **Machine type: `q35`**, not the default `i440fx`. This is what actually
  gives correct PCIe topology for passthrough (including the SR-IOV VF
  later) — OVMF alone without `q35` doesn't fully deliver on PCIe
  passthrough correctness. Set on the same System tab.
- **Secure Boot**: leave unchecked ("Pre-Enroll keys" unticked). Not needed
  for a stock Debian install with no custom/unsigned kernel modules.
- **Disk size**: size generously up front. Jellyfin cache, Docker image
  layers, and container logs (even capped via `json-file` `max-size`) add
  up — resizing a Proxmox virtual disk later is easy, but easier still to
  just not need to.
- **SR-IOV VF passthrough itself** (the actual PCI Device add under
  Hardware) happens *after* the host-side driver/kernel-param setup in
  `PROXMOX_GPU_SETUP.md` is complete — not part of initial VM creation.
  Don't block this checklist on that being done first.

**Choose your path now:**
- Continue to **Section 1** below for Path A (ISO Installer)
- Skip to **Section "Path B — Cloud-Init"** (after Section 8) to build/use
  a cloud-init template instead

---

# Path A — ISO Installer

## 1. Debian 13 Installer — Disk Partitioning

- **Guided — use entire disk**
- When asked about partition scheme: **"All files in one partition"**
  (not the separate /home, /var, /tmp split — that's for multi-user bare
  metal, not a single-purpose Docker VM)

## 2. Debian 13 Installer — Network Configuration

**If you're on a headless/remote console (can't unplug the cable in time to
force the manual DHCP-cancel prompt), don't fight the installer's timing —
just let DHCP proceed during install, then set the static IP manually right
after first boot instead (steps below).** Trying to catch the "Configuring
the network" progress bar in the few seconds before DHCP succeeds is not
reliable enough to depend on remotely.

**During the installer:**
- Let DHCP auto-configure. Note whatever hostname/domain prompts appear;
  set hostname to `vm2-services` regardless of what IP DHCP hands out.
- Don't worry about matching the final static IP at this stage — it'll be
  wrong temporarily and that's fine, you're fixing it in the next step.

**After first boot, once you have a shell (`su -` per Section 5):**

Check which interface name is in use — Proxmox VMs are commonly `ens18`:
```bash
ip link
```

Edit the interfaces file:
```bash
nano /etc/network/interfaces
```
Replace the `dhcp` line for that interface with a static block:
```
auto ens18
iface ens18 inet static
    address 192.168.69.240/24
    gateway 192.168.69.1
    dns-nameservers 1.1.1.1
```
(Static IP matching `inventory.ini` exactly; DNS set to a public resolver
for now — see the reasoning below on why not AdGuard yet.)

Apply it:
```bash
systemctl restart networking
```

**Do this from the Proxmox console tab, not an active SSH session** — the
interface will drop momentarily as it switches from the DHCP address to the
static one, which would cut off an SSH session connected to the old IP
mid-command. Reconnect via SSH at the new static IP once `ip addr show`
confirms it's live.

Then re-run the verification commands in Section 6 to confirm it actually
took — including the `resolvconf` package gotcha noted there, since
`dns-nameservers` in this file silently does nothing without it installed.

**DNS nameserver: use a public resolver for now (`1.1.1.1`)**, not
AdGuard's IP — even though AdGuard is the long-term plan, pointing here
before AdGuard is actually deployed creates a circular dependency where
this VM can't resolve anything (including what it needs to bootstrap
itself). Switch to AdGuard deliberately later, once AdGuard is confirmed
running.

## 3. Clear Any Stale SSH Host Key — Do This Before Any SSH Connection

**Do this now, before Section 4 onward** — even if you plan to do the next
few steps via the Proxmox console, you may end up reaching over SSH from
your control node sooner than expected (e.g. using `scp` in Section 8), and
this needs to be clear before that first connection attempt.

If this VM is reusing a static IP from a previous install, the SSH host key
regenerated on reinstall, but your control node still has the *old* one
cached. On the **control node**:
```bash
ssh-keygen -R 192.168.69.240
```
If this is a genuinely new IP that's never been connected to before, this
command will just report nothing to remove — harmless to run either way.

## 4. Debian 13 Installer — Software Selection

Select only:
- ✅ SSH server
- ✅ standard system utilities

Leave everything else unchecked (desktop environment, web server, print
server, SQL database) — all of that either doesn't belong on a headless
Docker host or gets installed/managed by Ansible itself instead.

## 5. First Login — Get to a Root Shell

`sudo` is **not installed** by default with this software selection. Don't
try to install it or work around it — just use:

```bash
su -
```
(enter the root password set during install)

## 6. Verify Static IP and DNS Actually Took Effect

```bash
ip addr show          # confirm the static IP is present
ip route show          # confirm default gateway is correct
cat /etc/resolv.conf    # confirm a nameserver line is present, not empty
ping -c 2 1.1.1.1        # raw connectivity
nslookup google.com 1.1.1.1   # DNS resolution specifically
```

**If `/etc/resolv.conf` is empty despite setting `dns-nameservers` in
`/etc/network/interfaces`:** that directive is only applied if the
`resolvconf` package is installed — Debian's minimal install doesn't include
it by default. Either:
```bash
apt install resolvconf
systemctl restart networking
```
or just write `/etc/resolv.conf` directly (fine for now, gets superseded by
Ansible/AdGuard later):
```bash
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```
(First check `ls -la /etc/resolv.conf` — if it's a *symlink*, `systemd-resolved`
owns it instead, and direct edits will get overwritten. In that case set
`DNS=1.1.1.1` in `/etc/systemd/resolved.conf` instead.)

## 7. Configure sshd for Root Key Login

```bash
nano /etc/ssh/sshd_config
```
Confirm/set:
```
PermitRootLogin prohibit-password
PubkeyAuthentication yes
```

## 8. Add the Ansible Control Node's Public Key to ROOT's authorized_keys

**This is the step that went wrong last time** — the key was added to a
regular user's `authorized_keys`, not root's. `prohibit-password` requires
the key in root's *own* file; it is never inherited from any other account.

**Important — don't `scp` straight to `root@` with a password.** Debian's
`sshd` defaults to `PermitRootLogin prohibit-password` even before Section
7's edit takes effect, so root password auth was never going to work here —
this isn't caused by having already applied Section 7, it's the out-of-box
default. Trying to work around it by temporarily setting `PermitRootLogin
yes` would work, but briefly exposes root to network password auth, which
is exactly what this setting exists to prevent — avoid that if you can.

**Route the key through your regular user account instead** — password
auth for non-root users is unaffected by `PermitRootLogin`:

```bash
# On the control node — password auth works here since this isn't root
scp ~/.ssh/id_ed25519.pub youruser@192.168.69.240:/tmp/id_ed25519.pub
```

Then, on the VM (SSH in as `youruser` with the password, or via the Proxmox
console):
```bash
su -
mkdir -p /root/.ssh && chmod 700 /root/.ssh
cat /tmp/id_ed25519.pub >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
chown -R root:root /root/.ssh
rm /tmp/id_ed25519.pub   # /tmp is world-readable; don't leave a stray copy of your public key there
systemctl restart ssh
```

This avoids both problems from this session at once: no root-password
chicken-and-egg, and no manual copy-paste into `nano` risking the
wrapped/truncated-key corruption noted earlier — `scp` transfers the file
exactly, byte for byte.

**If you'd rather not create/use a separate regular user account at all**,
the alternative is: temporarily set `PermitRootLogin yes` in `sshd_config`,
`systemctl restart ssh`, run the original root-targeted `scp` with the
root password, then immediately set it back to `prohibit-password` and
restart `ssh` again. Works, but leaves a short window where root password
auth is reachable over the network — the regular-user relay above avoids
that window entirely.

---

# Path B — Cloud-Init

Replaces Sections 1–8 above entirely. Do this instead if you'd rather not
click through the interactive installer at all.

## B1. One-Time: Build a Reusable Debian 13 Cloud-Init Template

Only needs doing once — every VM after this just clones it (Section B2).

```bash
# On the Proxmox host shell
wget https://cloud.debian.org/images/cloud/trixie/latest/debian-13-generic-amd64.qcow2

qm create 9000 --name debian13-template --memory 2048 \
  --net0 virtio,bridge=vmbr0 --bios ovmf --machine q35 --efidisk0 local-lvm:0

qm importdisk 9000 debian-13-generic-amd64.qcow2 local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm resize 9000 scsi0 +20G   # cloud images default small; add headroom

qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot order=scsi0
qm set 9000 --serial0 socket --vga serial0   # cloud images use serial console, not VGA

qm template 9000
```

Note: BIOS/machine-type (Section 0 above) are already baked into the
template here via `--bios ovmf --machine q35` — every clone inherits them,
no need to set them again per VM.

## B2. Every New VM (Including This Rebuild)

```bash
qm clone 9000 240 --name vm2-services --full
qm set 240 --ipconfig0 ip=192.168.69.240/24,gw=192.168.69.1
qm set 240 --nameserver 1.1.1.1
qm set 240 --ciuser root
qm set 240 --sshkeys ~/.ssh/id_ed25519.pub
qm start 240
```

**If this IP was used by a previous install of this VM**, also clear the
stale host key on your control node before testing connectivity (same
reasoning as Path A's Section 3):
```bash
ssh-keygen -R 192.168.69.240
```

This one command block replaces Sections 1 through 8 entirely:
- Static IP + DNS are correct from first boot — no `resolv.conf`
  troubleshooting, no `resolvconf` package gap.
- `--ciuser root` injects the SSH key directly into **root's**
  `authorized_keys` — the exact wrong-user mistake from Section 8 isn't
  possible here since there's no separate user-creation step at all.
- Debian's generic cloud image ships with `PermitRootLogin
  prohibit-password` already set by default — matches Section 7's setting
  with zero manual `sshd_config` editing.
- No `sudo`-missing surprise (Section 5) — you're never dropped into a
  non-root user session in the first place.

## B3. One Tradeoff to Know

There's no interactive console fallback the way the ISO installer's local
user/password setup gave you. Password auth is off from first boot, so
double-check your key is right *before* cloning — run
`ssh-keygen -lf ~/.ssh/id_ed25519.pub` and eyeball it — since there's no
"just log in locally and fix it" safety net if the injected key is wrong.

---

## 9. Test Manually From the Control Node BEFORE Involving Ansible

```bash
ssh -v -i ~/.ssh/id_ed25519 root@192.168.69.240
```
- Accept the new host key fingerprint prompt (`yes`) if this is a fresh
  host key.
- You should land in a root shell with **no password prompt**. If your key
  has a passphrase, you'll be prompted for *that* — normal, and fine for
  this manual test.
- If you get `Permission denied (publickey,password)` here, **stop and fix
  it before touching Ansible at all** — compare
  `ssh-keygen -lf ~/.ssh/id_ed25519.pub` (control node) against the exact
  key contents of `/root/.ssh/authorized_keys` (server) character-for-
  character. A mismatch here is almost always a copy-paste truncation from
  the key-add step (Path A Section 8, or a wrong value passed to Path B's
  `--sshkeys`).

## 10. If Your Private Key Has a Passphrase — Load an Agent

Ansible's non-interactive connections can't prompt for a key passphrase.
Do this once per control-node session before running Ansible:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

## 11. Confirm Ansible Connectivity

```bash
ansible vm2-services -i inventory.ini -m ping --ask-vault-pass
```
(The vault prompt is expected and unrelated to SSH — `group_vars/all/vault.yml`
gets loaded for every host in the `all` group, even for a plain `ping`, so
Ansible needs the vault password regardless of whether the task itself uses
any secrets.)

Expect a clean `pong`. If this works, you're ready for the real run:
```bash
ansible-playbook playbook.yml -i inventory.ini --limit vm2-services --ask-vault-pass
```

---

## Appendix A: PuTTY Setup (Optional — Windows Terminal Access)

Only needed if you want to SSH in from a Windows terminal via PuTTY,
separate from your Linux/WSL Ansible control node. Uses the same
`id_ed25519` keypair, converted to PuTTY's `.ppk` format.

### One-time setup

1. In PuTTYgen, click **Load**, switch the file filter to "All Files," and
   select `id_ed25519` (the OpenSSH-format private key — PuTTY can't read
   this directly, hence the conversion).
2. Click **Save private key** and save it as `id_ed25519.ppk`.
3. In PuTTY: **Session** — enter the VM's IP, port 22.
4. **Connection → SSH → Auth → Credentials** — browse to the `.ppk` file.
5. **Connection → Data** — set "Auto-login username" to `root`.
6. Back on **Session**, name and **Save** the session for reuse.

### After every VM rebuild

Nothing needs to change in the `.ppk` file or PuTTY's Auth settings — the
same keypair works regardless of how many times the VM is reinstalled.
Two things unrelated to the key itself do need addressing again, though:

1. **Root's `authorized_keys` on the fresh VM must already contain this
   key.** This isn't PuTTY-specific — it's the same Section 8 (ISO path)
   or cloud-init `--sshkeys` (Path B) step from earlier in this guide. If
   you've already completed that for your control node using this same
   keypair, you're covered; no separate re-add needed for PuTTY.
2. **PuTTY caches host keys separately from your Linux control node** — in
   the Windows Registry, not `~/.ssh/known_hosts`. Since the VM's SSH host
   key regenerates on every reinstall, PuTTY will show a **"WARNING —
   POTENTIAL SECURITY BREACH"** dialog on your first reconnect attempt
   after a rebuild — this is expected, not a real security problem, same
   reasoning as the `ssh-keygen -R` step in Section 3/B2. Click **Accept**
   (or **Update**, on newer PuTTY versions) to proceed. If this dialog gets
   dismissed with "Cancel" by reflex, the connection aborts *before* your
   key is ever offered — which looks identical to "PuTTY won't accept my
   key," even though the key was never actually tried.

If you get an explicit **"Server refused our key"** message (different
from the host-key warning above), that confirms it's actually cause 1 —
go complete Section 8 or confirm the right public key file was passed to
cloud-init's `--sshkeys`, then retry.

---

Mostly Path A (ISO Installer) issues — Path B (Cloud-Init) avoids most of
these by construction rather than requiring you to remember a fix.

| Symptom | Root cause |
|---|---|
| `Temporary failure in name resolution` | Empty/wrong `/etc/resolv.conf`, or pointed at AdGuard before it's deployed |
| `dns-nameservers` in `/etc/network/interfaces` has no effect | `resolvconf` package not installed, or `systemd-resolved` owns the symlinked file instead |
| `sudo: command not found` | Not installed by default with minimal software selection — use `su -` |
| PuTTY: "server refused our key" for root, but regular user works | Key was added to the wrong user's `authorized_keys` — must be root's own file specifically |
| `Host key verification failed` | Static IP reused after reinstall; control node's `known_hosts` has the old host key — fix with `ssh-keygen -R <ip>` |
| `[ERROR]: Attempting to decrypt but no vault secrets found` | Normal — any Ansible run against a host in `group_vars/all` scope needs `--ask-vault-pass` / `--vault-password-file`, even for `ping` |
| Key offered but still `Permission denied (publickey,password)` | Public/private key mismatch — usually a wrapped/truncated copy-paste into `authorized_keys` |
| `scp ... root@host` gets `Permission denied` even with the right password | Debian's `sshd` default is `PermitRootLogin prohibit-password` out of the box — root password auth never worked here regardless of Section 7. Route the key through a regular user account instead (Section 8) |
| PuTTY "won't accept" a key that worked before a rebuild | Usually not the key at all — either root's `authorized_keys` isn't repopulated yet on the fresh VM (do Section 8 / cloud-init `--sshkeys` again), or PuTTY's own host-key cache (separate from Linux `known_hosts`) is showing a security-breach warning that got dismissed by reflex (see Appendix A) |
