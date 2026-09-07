# Ansible Host Onboarding Checklist

A repeatable checklist for bringing any new VM or LXC into Ansible management
— written after onboarding the Kopia server LXC, which surfaced most of the
gotchas below. Meant to be worked top to bottom for every new host from here
on, not just re-derived from memory each time.

## 0. Before touching anything: sync the repo

- [ ] `git pull` on whatever machine you're about to run `ansible-playbook`
      from — **do this every time**, not just when onboarding a new host.
      The Kopia LXC ping failure turned out to be a stale local
      `inventory.ini` that was never pushed from the desktop repo. Treat
      `git pull` as a standing habit alongside `--check --diff`.
- [ ] Confirm you're editing the same `inventory.ini` the control node
      actually reads — if in doubt, `git log -1 -- inventory.ini` on the
      control node to see what it currently has.

## 1. Prerequisites on the new host itself

- [ ] Host is reachable on a static/known IP on the LAN.
- [ ] `python3` is present — Ansible needs it on the managed node:
      `which python3` (install with `apt install -y python3` if missing).
- [ ] `curl` is present if any task or manual troubleshooting will need it —
      **not guaranteed on minimal LXC templates**. Learned this the hard
      way on the Kopia LXC: `apt-get install -y curl` first if a "command
      not found" shows up mid-troubleshooting.
- [ ] SSH is running and reachable (`systemctl status ssh` if you have
      console access already).

## 2. SSH key access (reuse the existing keypair — don't generate a new one)

The private key lives on the **control node**, not on any managed host. You
are not copying keys between VMs — you're adding your one existing public
key to a new host's `authorized_keys`.

- [ ] On the control node: confirm the keypair referenced in
      `inventory.ini` (`ansible_ssh_private_key_file=~/.ssh/id_ed25519`)
      actually exists: `ls -la ~/.ssh/`.
- [ ] Get the public key: `cat ~/.ssh/id_ed25519.pub`.
- [ ] Get onto the new host through an out-of-band path first (SSH won't
      work until the key is added — chicken-and-egg):
  - Proxmox LXC: `pct enter <vmid>` from the Proxmox host shell, or the
    Console tab in the web UI.
  - Proxmox VM: console tab, or whatever initial access method was used to
    set it up.
- [ ] On the new host: `mkdir -p ~/.ssh && chmod 700 ~/.ssh`, then append
      the public key line to `~/.ssh/authorized_keys` and
      `chmod 600 ~/.ssh/authorized_keys`.
- [ ] Confirm `/etc/ssh/sshd_config` allows key-based root login:
      `PermitRootLogin yes` or `prohibit-password` (either is fine — only
      `no` blocks it), and `PubkeyAuthentication yes`. Restart with
      `systemctl restart ssh` if you changed anything.
- [ ] **Shortcut**: if the host still accepts a root password (common right
      after Proxmox creates it), skip the manual copy/paste entirely —
      `ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<host-ip>` from the control
      node does steps above in one command.
- [ ] Test from the control node: `ssh -i ~/.ssh/id_ed25519 root@<host-ip>`
      — should drop straight into a shell, no password prompt.

## 3. Inventory

- [ ] Decide the right group. New role entirely → new group (e.g.
      `[backup]`). Same role as existing hosts → add to that group instead.
- [ ] Add the host under the correct group in `inventory.ini`:
  ```ini
  [backup]
  kopia-server ansible_host=192.168.69.XXX
  ```
- [ ] If this host needs Docker, also add it (or its group) under
      `docker_hosts` rather than assuming `hosts: all` in the playbook will
      cover it correctly — see the playbook-scoping note below.
- [ ] **Commit and push `inventory.ini` immediately** — this is the exact
      step that got missed for the Kopia LXC. An uncommitted inventory
      change only exists on whichever machine you edited it on.
- [ ] `git pull` on the control node if it's a different machine than the
      one you edited on.

## 4. Verify Ansible can actually reach it

- [ ] `ansible <hostname> -i inventory.ini -m ping` — expect a green
      `pong`.
- [ ] If you get `Could not match supplied host pattern` — the host isn't
      in the inventory the control node is reading. Re-check step 3, not
      the SSH connection (SSH working is a separate thing from inventory
      being correct).
- [ ] If you get an actual connection error here instead — re-check step 2.

## 5. Playbook scoping — don't assume `hosts: all` does the right thing

`playbook.yml`'s plays aren't all meant for every host type. Before running
anything against a brand-new host, check what each play actually targets:

- [ ] IPv6-disable / apt-hygiene tasks — fine for `hosts: all`, every host
      benefits.
- [ ] Docker install — should target a `docker_hosts` group (an alias of
      `gateways` + `cores` via `[docker_hosts:children]`), **not** `all` —
      a Kopia server or other non-Docker host doesn't need Docker installed
      just because it's in the inventory.
- [ ] QuickSync / CIFS storage plays — already scoped to `cores`
      specifically; a new host in a different group won't pick these up,
      which is usually correct (confirm it's correct for this specific
      host though).
- [ ] Run `ansible-playbook playbook.yml -i inventory.ini --limit <new-host> --check --diff`
      before the real run, same as always — this also surfaces scoping
      mistakes (e.g. Docker tasks unexpectedly trying to run against a
      non-Docker host) before anything actually changes.

## 6. Editing the encrypted vault (`group_vars/vault.yml`)

Comes up any time a new host needs a new secret (a Kopia scoped password,
for example) added to the vault alongside the existing `vault_vpn_*`,
`vault_cifs_*`, etc. entries.

- [ ] **Set `nano` as the editor before running any vault command** —
      confirmed working over SSH; this is the one that matters.
      `ansible-vault edit` launches whatever `$EDITOR` points to, and
      **GUI editors that fork a separate window return control to
      `ansible-vault` immediately** — it re-encrypts the file before
      you've actually saved your changes in the window that just opened,
      so the edit is silently lost. `nano` blocks in the terminal until
      you exit it, which is what makes it work correctly here.
- [ ] One-off, for a single command: `EDITOR=nano ansible-vault edit group_vars/vault.yml`
- [ ] Persistent, so you stop having to remember this: add
      `export EDITOR=nano` to `~/.bashrc` (or `~/.zshrc`) on the control
      node, then `source ~/.bashrc` (or open a new shell).
- [ ] Note: `code --wait` also works correctly *if run locally on the same
      machine* as the vault file — the `--wait` flag is what makes VS Code
      block instead of forking. This only matters if you're not on SSH;
      for anything over SSH, use `nano`.

**Common vault commands, once `$EDITOR` is set:**

| Task | Command |
|---|---|
| Edit an existing vault file | `ansible-vault edit group_vars/vault.yml` |
| Create a brand-new vault file | `ansible-vault create group_vars/vault.yml` |
| View contents without editing | `ansible-vault view group_vars/vault.yml` |
| Encrypt an existing plaintext file | `ansible-vault encrypt <file>` |
| Change the vault password itself | `ansible-vault rekey group_vars/vault.yml` |
| Run a playbook that needs vault secrets | `ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass` (or `--vault-password-file <path>`) |

**Basic nano controls**, since it isn't the default for everyone:

- Move around with arrow keys, no mouse needed.
- Save: `Ctrl+O`, then `Enter` to confirm the filename.
- Exit: `Ctrl+X` (after saving, or it'll ask if you want to save first).

**Workflow for adding a new secret** (e.g. a Kopia scoped password for a
new host):

- [ ] `EDITOR=nano ansible-vault edit group_vars/vault.yml`
- [ ] Add the new `vault_`-prefixed line, e.g. `vault_kopia_scoped_pw_kopia_server: "..."`
- [ ] Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`) — this re-encrypts on save automatically.
- [ ] Add the corresponding plain-name reference in `group_vars/all.yml`,
      pointing at the `vault_` var, same pattern as `VPNPASSWORD` etc.
- [ ] Commit both files — `vault.yml` is safe to commit as-is since it's
      ciphertext; never commit an unencrypted version.
- [ ] `git push`, then `git pull` on the control node if it's a different
      machine (see the standing habit from step 0).

## 7. Known pitfalls seen so far (add to this list as new ones show up)

- **Minimal LXC templates may not include `curl`** — don't assume it's
  there when troubleshooting network issues on a fresh container.
- **Dead IPv6 routing** can silently break `apt`/`curl`/Ansible's
  `get_url` on any new host that hasn't had the IPv6-disable task applied
  yet — a host outside `playbook.yml`'s current `hosts: all` scope (or one
  that simply hasn't been run yet) is still exposed to this.
- **Unprivileged LXCs** can restrict writes to some `/proc/sys/net/*`
  paths depending on container security/nesting settings — if a sysctl
  task silently no-ops or errors on an LXC specifically (but the same task
  works fine on VMs), check the LXC's Proxmox-side feature flags
  (`pct set <vmid> --features nesting=1`) rather than assuming the sysctl
  syntax itself is wrong.
- **Stale local inventory** — an inventory edit only exists where it was
  made until it's committed and pulled everywhere else. This one cost a
  full troubleshooting detour before the real cause (an unpushed commit)
  was found.
- **GUI editors silently losing vault edits** — a forking editor (one that
  opens a separate window and immediately returns control to the terminal)
  makes `ansible-vault edit` re-encrypt before your changes are actually
  saved. Confirmed `nano` works correctly over SSH; `code --wait`
  specifically (not plain `code`) works when running locally.