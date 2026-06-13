# Orange Pi 3 LTS — Work Notes

Summary of the work done to bring **Orange Pi 3 LTS** support into RatOS, what
changed from upstream, and why.

## Source of the merge

- **Upstream / base branch:** `v2.1.x` (RatOS, `origin` = https://github.com/Rat-OS/RatOS.git)
- **Merged from:** https://github.com/jscancella/RatOS.git
  branch `orange_pi_3lts_using_official_image`
- **Working branch created for this work:** `v2.1.x-orange-pi3lts` (off `v2.1.x`)

The fork's branch added Orange Pi 3 LTS support (22 commits, 16 files). It was
originally built around the **Orange Pi OS** image (Debian *Bullseye*). We later
switched it to the **official Armbian Trixie** image, which required reconciling
the modules with the repo's standard Armbian build flow.

## Commits on this branch

1. `Merge remote-tracking branch 'jscancella/orange_pi_3lts_using_official_image'`
   — the merge itself, with conflicts resolved (see below).
2. `fix(orangepi3lts): switch to official Armbian Trixie image and reconcile modules`
   — base-image swap + module reconciliation for Trixie.

---

## Part 1 — Merge conflict resolution

### 1a. `.github/workflows/BuildImages.yml` (two conflicting regions)

| Region | Upstream (HEAD) | Fork | Resolution | Why |
|---|---|---|---|---|
| Push-trigger branches | `v2.x`, `v2.1.x` | `orange_pi_3lts_using_official_image` | **`v2.1.x-orange-pi3lts` only** | CI should fire only on this dedicated dev branch. |
| Matrix-generation step | `setup_matrix.py` (config-driven, reads `workflow_config.yml`) | hardcoded `find … -name orangepi3lts` | **Kept the fork's `find`** | On this branch we only want to build the single Orange Pi 3 LTS image. |

### 1b. `src/modules/cb1spi/start_chroot_script` (committed conflict markers)

Not a normal conflict — the fork had **committed unresolved `<<<<<<<`/`>>>>>>>`
markers** from an earlier botched merge on their side (markers referenced
`upstream/v2.x` and a rename from `spi/start_chroot_script`).

The two sides were a **generic SPI** script vs. the **CB1-specific** script.
Since this file *is* the `cb1spi` module, we kept the **CB1-specific** version.
The generic SPI logic was not lost — the fork had separately added it as a new
`src/modules/spi/` module.

---

## Part 2 — Switch to the official Armbian Trixie image

The branch originally downloaded the Orange Pi OS Bullseye image from the fork's
own GitHub release. We switched to the official Armbian build hosted on a public
mirror:

```
https://mirrors.dotsrc.org/armbian-dl/orangepi3-lts/archive/Armbian_26.5.1_Orangepi3-lts_trixie_current_6.18.33_minimal.img.xz
```

### `config/armbian/orangepi3lts`

| Change | From | To | Why |
|---|---|---|---|
| `DOWNLOAD_URL_IMAGE` | jscancella release (Orange Pi OS / Bullseye) | dotsrc Armbian Trixie image | Use an official, publicly hosted image. |
| `DOWNLOAD_URL_CHECKSUM` | `…img.sha256` | `…img.xz.sha` | The mirror provides `.img.xz.sha` (the `.sha256` URL 404s); contents are standard sha256sum format, so verification works the same. |
| `BASE_DISTRO` | commented out | `export BASE_DISTRO=armbian` | It is now an official Armbian image; `orangepiconfig` branches on this. |
| `BASE_IMAGE_RASPBIAN` | (unset) | `export BASE_IMAGE_RASPBIAN=no` | Matches the repo's Armbian flow (`armbian/default`) — it is not a Raspbian image. |
| `BASE_ROOT_PARTITION` | `1` | `1` (unchanged) | Standard single-partition Armbian Allwinner layout. |
| `MODULES` | `…deb_mirrors…spi…` | `…deb_namserver…` (see below) | Reconcile with the standard Armbian/CB1 module flow. |

### Module changes

| Module | Action | Why |
|---|---|---|
| `deb_mirrors` | **Removed** (dir deleted, replaced by `deb_namserver` in MODULES) | It overwrote `/etc/apt/sources.list` with **Bullseye** repos → breaks apt on a **Trixie** image. It only existed to work around Orange Pi OS's slow China mirrors; the dotsrc mirror needs no such workaround. `deb_namserver` (the RatOS-standard module CB1 uses) just appends a nameserver — harmless. |
| `spi` | **Removed** (dir deleted, dropped from MODULES) | Wrote overlays to `/boot/BoardEnv.txt`, which **does not exist on Armbian** (Armbian uses `/boot/armbianEnv.txt`). Also redundant: `orangepiconfig` already enables SPI by writing the correct overlay to `armbianEnv.txt` — mirroring how `cb1config` handles its board without a separate generic spi module. |
| `orangepiconfig` | **Fixed** (see below) | Make it correct for Armbian/Trixie. |
| `hotspot_orangepi` | Kept | Generic `create_ap` build; works on Trixie. |

### `src/modules/orangepiconfig/start_chroot_script` fixes

- **armbianEnv.txt overlay write:** the original used a multi-line `echo` that
  emitted a malformed, leading-space `  param_spidev_spi_bus=1` line. Rewrote it
  as two clean `echo … >> /boot/armbianEnv.txt` lines. Also fixed the dead
  Orange-Pi-OS `else` branch (it wrote to a bad relative path `orangepiEnv.txt`).
- **`/etc/board-release` symlink:** the original linked to
  `/etc/orangepi-release`, which **does not exist on Armbian** (Armbian exposes
  `/etc/armbian-release`). Now symlinks to `/etc/armbian-release` on Armbian and
  falls back to `/etc/orangepi-release` otherwise. Switched the guard from `-f`
  to `-e` so an existing symlink is detected.

### Deliberately left unchanged

- `check_install_pkgs avahi` in `orangepiconfig` — identical to the working
  `cb1config`, so it is not a bug in this build context.
- Runtime wifi/user setup (`hotspot_orangepi`, `BASE_USER=pi`,
  `BASE_ADD_USER=yes`) — mirrors the proven CB1 structure.

---

## Open items — to validate with a real CI build

These could not be verified locally (the CustomPiOS image build cannot run here);
pushing `v2.1.x-orange-pi3lts` triggers the workflow:

1. **`BASE_ROOT_PARTITION=1`** — correct for a standard single-partition Armbian
   Allwinner image; confirm against this specific image if the resize step fails.
2. **`network` module** — referenced in MODULES (as in CB1) but has no
   `src/modules/network/` directory, so it is resolved from elsewhere at build
   time. Fine if CB1 builds.
3. **First-boot user creation** on the Armbian image with `BASE_ADD_USER=yes` /
   `BASE_USER=pi`.

## Follow-ups

- The image is pulled from a third-party mirror (dotsrc). Consider re-hosting
  under the Rat-OS org for a release.
- A git remote `jscancella` was added during the merge; remove with
  `git remote remove jscancella` if not needed.
