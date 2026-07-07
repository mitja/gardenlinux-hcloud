# Feature: robot

Hetzner **Robot** (dedicated bare metal) element — composes with the upstream `baremetal`
platform: flavor **`baremetal-robot-gardener_prod-amd64`**. Deliberately different from
`hcloud` (docs/32, docs/73):

| Concern | hcloud (cloud) | robot (dedicated) |
|---|---|---|
| kernel | `linux-image-cloud-$arch` (VM drivers only) | full `linux-image-$arch` via `metal` (Intel igb/e1000e, AHCI, mdadm…) |
| provisioning | cloud-init + DataSourceHetzner (metadata/user_data) | **none** — Robot has no metadata service; bootstrap = docs/32 pull-onboarding agent (`autosetup.go`), test access = rescue-injection |
| boot | classic `_legacy` dual / usi EFI-only | `metal` includes `_legacy` (BIOS+UEFI dual) — auction boxes are commonly legacy-BIOS |
| deploy | snapshot via hcloud-upload-image / rescue-dd | **rescue-dd only** (`installimage` has no custom-image path); recovery = Robot API rescue+reset |
| network | DHCP (hcloud always answers) | DHCP on the primary NIC (Robot answers with the static primary IP — verified 2026-07-07 on an AX-class auction box); vSwitch = 802.1q VLAN sub-interface, **MTU 1400**, static IP — configured by the onboarding agent, not baked |
| carried over | DNS trim (same Hetzner recursors), inotify sysctls, sshd PermitRootLogin+Include fixes | same |

No cloud-init in this image at all (metal excludes `cloud`; we add no datasource) — anything
instance-specific is the onboarding agent's job. vSwitch/VLAN is intentionally NOT baked:
VLAN id/IP are per-deployment (docs/32 RobotAdapter).
