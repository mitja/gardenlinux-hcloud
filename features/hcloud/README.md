# Feature: hcloud

Platform feature for **Hetzner Cloud** — the genuinely novel piece of docs/73 §2 (no upstream
Hetzner flavor exists; `kvm` provisions via Ignition/QEMU fw_cfg which hcloud cannot deliver,
`openstack` pins `datasource_list: [ConfigDrive, OpenStack, Ec2]`). Modeled on the upstream
`openstack`/`aws` platform features: `cloud-init` + platform-specific datasource wiring on top of
the `cloud` element.

Flavors (see root README): `hcloud-gardener_prod-amd64` (classic, BIOS+UEFI via `_legacy`) and
`hcloud-gardener_prod_usi-amd64` (EFI-only, in-place-capable).

## Decisions

| What | Choice | Why |
|---|---|---|
| datasource | `datasource_list: [ Hetzner, NoCloud, None ]` | `Hetzner` is upstream in cloud-init (DMI `sys_vendor=Hetzner` + 169.254.169.254 metadata, reads `user_data` → Gardener's OSC provision script runs, gardener-node-agent takes over — same shape as GL's aws/azure/openstack). `NoCloud` keeps local QEMU smoke tests possible (cidata seed). `None` terminates discovery so boot never hangs. Not forcing `datasource:` in ds-identify.cfg (openstack does) keeps that fallback alive; `policy: enabled` still guarantees cloud-init runs. |
| network config | **disabled** in cloud-init (aws pattern) | GL's systemd-networkd default (`99-default.network`: DHCP on `en*`/`eth*`) covers hcloud public IPv4 *and* hot-attached private NICs by name-match — no MAC-pinned render to go stale (the 2026-07-06 shoot-nbg netplan incident class is structurally gone). Caveat: no static public IPv6 from metadata; worker nodes don't need it. **[V-GL1]** verify public IPv4 DHCP + private-NIC hot-attach on a real hcloud VM. |
| console | `console=ttyS0 console=tty0` (tty0 last = /dev/console) | hcloud's only reachable console is noVNC on the VGA device; kvm feature's serial-primary ordering would hide boot output there. |
| DNS trim | static Hetzner recursors ≤3, all dynamic DNS off | node-adapt.sh (2) carry-over, adapted from Ubuntu-netplan to a plain networkd drop-in (GL runs systemd-networkd + systemd-resolved; netplan does not exist). docs/12 C10/C18. |
| inotify | sysctl.d file | node-adapt.sh (3) carry-over, unchanged. |
| usirepo.conf | written by `exec.config` **only when `_usi` is in `$BUILDER_FEATURES`** | Upstream has no usirepo feature/file at all — the file is an operator-side contract read by os-gardenlinux's `inplace-update.sh` (`--repo` override for `gardenlinux-update`, default ghcr.io/gardenlinux/gardenlinux). The `$BUILDER_FEATURES` guard is the upstream idiom for variant-conditional config (see openstack's exec.config). |
| root access | `disable_root: false` + sshd `PermitRootLogin prohibit-password` (key-only); **no** `admin` default_user | Found live in the GL1 boot test (2026-07-07): hcloud **vendor-data** pins `system_info.default_user.name: root` — vendor-data outranks all /etc/cloud config, so the openstack-style `admin` user is never created on hcloud and the platform-injected key lands on root. With GL's ssh-feature hardening (`PermitRootLogin no`) + default `disable_root: true`, the delivered key was doubly dead. Passwords stay impossible (`lock_passwd`, prohibit-password). Gardener keeps `ssh.service` disabled by preset; this governs deliberate debug access only. USI caveat: `/root` = tmpfs → the key evaporates on reboot (cc_ssh is once-per-instance). |

## node-adapt.sh items intentionally dropped (docs/73 §2 table)

| node-adapt.sh | Why dropped on GL |
|---|---|
| (1) containerd 1.7 pin + apt hold | GL 2150.6.0 ships containerd **2.2.5** (config v3 — supported by gardener ≥ v1.115, fine on our v1.142.5; past the 2.2.1 `ctr` C9 regression). GL *is* the unpin. Re-verify node join in GL2 **[V-GL2]**. |
| (1b) apparmor runc kill-denial disable | Ubuntu-specific: apparmor≥4's `/etc/apparmor.d/runc` vs containerd <1.7.24 profile. GL: runc 1.4.3, containerd 2.2.5, and no runc apparmor profile expected — **verify none ships [V-GL2]** (gardener feature does enable apparmor: `security=apparmor` cmdline). |
| (2b) netplan MAC self-heal | netplan doesn't exist on GL; with cloud-init network config disabled there is no rendered MAC pin to go stale. Hot-attach behavior under plain networkd still needs live validation **[V-GL2]**. |
