# gardenlinux-hcloud — Garden Linux images for Hetzner Cloud & Robot dedicated

Builds custom [Garden Linux](https://github.com/gardenlinux/gardenlinux) flavors for Hetzner:
`hcloud` (cloud, cloud-init + Hetzner datasource — no upstream flavor exists for hcloud) and
`robot` (dedicated bare metal, full kernel, no cloud-init). Broken out of a private Gardener
landscape repo (internal doc references like `docs/NN` point there).

Builder config dir (github.com/gardenlinux/builder layout, seeded from `builder_example`) that
builds our custom **`hcloud`** Garden Linux flavor — the worker OS that retires the Ubuntu-drift
treadmill (containerd pin, apparmor runc kill-denial, netplan MAC drift) and unlocks price-safe
in-place upgrades on grandfathered pets (docs/38 Layer 1, docs/73).

> Public images build via GitHub Actions → ghcr from this repo.
> build on **GitHub Actions → ghcr**, never on Forgejo ([[publish-images-via-github-actions]]).

## Flavors

| cname | Boot | In-place | Use |
|---|---|---|---|
| `hcloud-gardener_prod-amd64` | BIOS+UEFI (`_legacy`) | no | universal fallback, incl. BIOS-only CX types |
| `hcloud-gardener_prod_usi-amd64` | EFI-only (USI/UKI) | **yes** (`gardenlinux-update`) | CPX/CCX/CAX pools, docs/38 Layer-1 upgrades |

Naming follows upstream `flavors.yaml` exactly: platform `hcloud` + element `gardener` + flags
`_prod`/`_usi` (upstream ships e.g. `kvm-gardener_prod_usi-amd64`). arm64 (CAX) is staged but
deferred — docs/73 GL0.

## The label contract (do not break)

Snapshots carry **`gardener.cloud/image-name=gardenlinux-<version>`** (e.g.
`gardenlinux-2150.6.0`). That one selector is shared by (docs/27 §9):
- CloudProfileConfig `providerConfig.machineImages[].versions[].imageName`,
- nodepool `ManagedServer.spec.imageRef`,
- the uploader below (provider fork v0.8.0 resolves by label, newest-by-Created wins).

## Layout: ours vs vendored

```
features/hcloud/          OURS — the Hetzner platform feature (see its README for decisions)
get_repo|version|...      OURS — pins: packages.gardenlinux.io @ 2150.6.0, epoch 1771372800
build.config, .github/    OURS
build, keyring.gpg,
features/<everything else>,
cert/ tooling             VENDORED verbatim from gardenlinux/gardenlinux @ 2150.6.0
```

Vendoring is required, not a style choice: the builder mounts only this dir's `features/`, and
`parse_features` asserts every include/exclude-referenced feature exists locally (21-feature
closure). **Never edit vendored files** — bump `GL_TAG` in `hack/vendor-upstream.sh` (+ the
`get_version`/`get_timestamp` pins) and re-run it. Version bump tripwire: if GL release notes
show **containerd 2.3+** (config v4), Gardener must be ≥ v1.144 first (docs/12, docs/73 §6).

## Build locally

Requires rootless podman (Linux). Classic flavor:

```sh
./build hcloud-gardener_prod-amd64
# → .build/hcloud-gardener_prod-amd64-2150.6.0-<commit8|local>.raw
```

USI flavor (generate self-signed dev certs once first — `_usi/exec.post` bakes
`cert/oci-sign.crt` + the four `secureboot.*.auth` blobs into the image):

```sh
./cert/build oci-sign.crt secureboot.pk.auth secureboot.null.pk.auth secureboot.kek.auth secureboot.db.auth
./build hcloud-gardener_prod_usi-amd64
# → …-2150.6.0-<commit>.raw (EFI-only disk) + ….uki + ….esp.tar
```

macOS: no native podman — use an OrbStack Linux machine (`orb create ubuntu gl-builder`,
`orb -m gl-builder sudo apt-get install -y podman`, then run the builds inside the machine;
copy this dir to a VM-local path first — virtiofs-backed workdirs are slow and can upset
rootless podman mounts).

## Upload to an hcloud project (GL1 — lab project ONLY for now)

[apricote/hcloud-upload-image](https://github.com/apricote/hcloud-upload-image) (rescue-boot +
dd + snapshot; disk zeroed → small snapshot; raw only — qcow2 is capped ~960MB):

```sh
xz -T0 -9 .build/hcloud-gardener_prod-amd64-2150.6.0-*.raw
export HCLOUD_TOKEN=<lab project rw token>
hcloud-upload-image upload \
  --image-path .build/hcloud-gardener_prod-amd64-2150.6.0-*.raw.xz \
  --compression xz \
  --architecture x86 \
  --location nbg1 \
  --description "Garden Linux 2150.6.0 hcloud-gardener_prod" \
  --labels gardener.cloud/image-name=gardenlinux-2150.6.0
```

Same for the USI raw. NOTE: while both variants are labeled `gardenlinux-2150.6.0` the label
resolver takes newest-by-Created — during GL1/GL2 keep only ONE variant per project, or suffix
the trial label (e.g. `…=gardenlinux-usi-2150.6.0`) and reconcile before GL4.

## GL1 boot-test checklist (docs/73 §4 — gate: "a GL server on hcloud runs our cloud-init payload")

**Executed 2026-07-07: PASS on both variants (v2 snapshots, cpx22/nbg1) — results in the landscape repo's implementation log.**

Boot a CPX server from the snapshot. The `--user-data` script must write a marker file **and
`systemctl enable --now ssh.service`** — the gardener_prod preset ships sshd (and containerd)
disabled; on real nodes gardener-node-agent enables them, on a standalone test your user-data
must (it runs as root via cloud-init, so no chicken-and-egg). Then verify:

1. **Datasource:** `cloud-init status --long` clean; `cloud-id` = `hetzner`;
   `/run/cloud-init/ds-identify.log` shows Hetzner found via DMI.
2. **user_data executed:** the marker file exists (proxy for Gardener's OSC provision script).
3. **Networking:** public IPv4 up via networkd DHCP (`networkctl status`); then
   `hcloud server attach-to-network` (lab!) → private NIC gets DHCP lease *without* reboot
   (the 2026-07-06 incident class check); `resolvectl` shows exactly the 3 Hetzner recursors.
4. **Carry-overs:** `sysctl fs.inotify.max_user_instances` = 8192;
   `containerd --version` = 2.2.5 (service disabled by preset — gardener enables it);
   apparmor: `/etc/apparmor.d/runc` **exists on GL too** (2026-07-07 finding — the "absent"
   assumption was wrong); the kill-denial mitigation is containerd ≥ 1.7.24-line behavior,
   i.e. 2.2.x. Verify kill/exec on a pod in GL2 drills, not profile absence here.
5. **USI variant on a CPX (UEFI):** boots EFI-only; `bootctl status` sane;
   `/etc/gardenlinux/usirepo.conf` present (TODO ref), `/etc/gardenlinux/oci_signing_key.pem` baked.
6. **Console:** hcloud web console (noVNC) shows getty (console=tty0 last).

Findings land in the (private) landscape repo's implementation log.

## Automated test suite (run on EVERY image update — GL releases every 2–5 weeks)

```sh
# throwaway-resource scenarios: basic, volume, hot-attach, container(kill-denial), reboot, private-nat
HCLOUD_TOKEN=<lab> ./test/run-tests.sh --image <snapshot-id> --arch x86   # and --arch arm
# grandfathered-pet rebuild (BIOS on CX / arm64 on CAX; price-safe, never delete/change_type):
HCLOUD_TOKEN=<lab> ./test/rebuild-pet-test.sh --server <id> --image <snapshot-id> --power-off-after
```

Everything the suite creates is `gl-test-`-prefixed + run-labeled and swept on exit; cost per
run is cents (2-4 cpx22 for ~15 min + a 10G volume). Platform facts the scenarios encode
(private-net DHCP gives no DNS/route/metadata; fully-private = no metadata at all; GL private
NIC is `enp7s0` not `ens10`; usi `/` immutable + `/root` tmpfs) are documented in the
landscape repo's implementation log (GL1.5). CI wiring (TODO): a `test` job in `.github/workflows/build.yml` that uploads
the fresh raws to the lab project via `hcloud-upload-image` and runs both archs' suites.
