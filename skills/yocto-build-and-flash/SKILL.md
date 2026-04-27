---
name: yocto-build-and-flash
description: Guide for building the Yocto myir-image-core for the MYiR MYD-YF13X (STM32MP135) and flashing it to hardware. Use this when configuring Yocto layers, running BitBake builds, locating build artefacts, or flashing images via STM32CubeProgrammer.
---

# Yocto Build & Flash for MYD-YF13X

## BSP Tarball & Layer Extraction

- The vendor BSP layers are shipped as a single tarball: `yf13x-yocto-stm32mp1-5.15.67.tar.bz2` (~360 MB).
- Download via `make bsp-fetch` (uses `oras pull` from GHCR) or place manually in `.release-staging/`.
- Extract with `make bsp-extract`, which runs:
  ```bash
  tar xjf .release-staging/yf13x-yocto-stm32mp1-5.15.67.tar.bz2 -C yocto-bsp/
  ```
- After extraction, `yocto-bsp/layers/` contains:
  - `openembedded-core/` (OE-Core base)
  - `meta-openembedded/` (meta-oe, meta-python, meta-networking, …)
  - `meta-qt5/`
  - `meta-st/` (meta-st-stm32mp, meta-st-stm32mp-addons, meta-st-openstlinux)
  - `meta-myir-st/` (MYiR board support)

## bblayers.conf – Critical Details

- The `bblayers.conf` is pre-configured in `yocto-bsp/build/conf/bblayers.conf`.
- It uses `OEROOT` relative paths derived from `os.path.dirname(d.getVar('FILE'))`.
- **16 layers** are required, grouped as BASELAYERS, BSPLAYER, ADDONSLAYERS, FRAMEWORKLAYERS, OPENEMBEDDED.
- The custom project layer `software/meta-imf` is included via:
  ```
  ${OEROOT}/../software/meta-imf
  ```
- Layers are conditionally included using `os.path.isfile()` checks against `conf/layer.conf`.

### Layer Collection Names (LAYERDEPENDS)

**Critical:** `LAYERDEPENDS` in `layer.conf` files must use **collection names**, not directory names:

| Directory | Collection name |
|-----------|----------------|
| `meta-openembedded/meta-oe` | `openembedded-layer` |
| `meta-openembedded/meta-python` | `meta-python` |
| `meta-openembedded/meta-networking` | `networking-layer` |
| `meta-st/meta-st-stm32mp` | `stm-st-stm32mp` |
| `meta-st/meta-st-stm32mp-addons` | `stm-st-stm32mp-mx` |
| `meta-st/meta-st-openstlinux` | `st-openstlinux` |
| `meta-qt5` | `qt5-layer` |
| `meta-myir-st` | `stm-myir-st` |
| `openembedded-core/meta` | `core` |

Wrong names cause: `Layer '…' depends on layer '…', but this layer is not enabled`.

To find collection names: `grep BBFILE_COLLECTIONS <layer>/conf/layer.conf`

## local.conf – Machine Configuration

- `MACHINE` must be set in `local.conf` or passed via environment.
- Available machines defined in `meta-myir-st/conf/machine/`:
  - `myd-yf13x` – SD card boot (**default**)
  - `myd-yf13x-emmc` – eMMC boot
  - `myd-yf13x-nand` – NAND boot
- `DISTRO` is `nodistro` (no distro layer). This sets `TCLIBC = "glibc"`.

## TMPDIR is tmp-glibc, Not tmp

- Because `DISTRO = "nodistro"` sets `TCLIBC = "glibc"`, BitBake uses **`tmp-glibc`** instead of `tmp`.
- All build artefacts are in `yocto-bsp/build/tmp-glibc/`, not `yocto-bsp/build/tmp/`.
- This is a common source of confusion when searching for images.

## Origin of build/conf/ Files

The files in `yocto-bsp/build/conf/` (`local.conf`, `bblayers.conf`, `templateconf.cfg`) are **not** hand-written from scratch – they originate from the **`oe-init-build-env`** bootstrap mechanism:

1. When you run `source oe-init-build-env <build-dir>`, the script checks whether `<build-dir>/conf/local.conf` already exists.
2. **First run (no conf/):** The script reads `templateconf.cfg` to find the template directory and copies the sample files:
   - `bblayers.conf.sample` → `bblayers.conf`
   - `local.conf.sample` → `local.conf`
   - `conf-notes.txt` → displayed to terminal
3. **Subsequent runs (conf/ exists):** The script leaves all files untouched and only sets up the shell environment (`PATH`, `BUILDDIR`, etc.).

### templateconf.cfg

`yocto-bsp/build/conf/templateconf.cfg` currently points to `meta/conf` (the OE-Core default). MYiR provides its own templates at `meta-myir-st/conf/template/`, but we do not use them because our configuration diverges significantly:

- `bblayers.conf` includes `meta-imf` and uses `OEROOT`-relative paths
- `local.conf` sets `MACHINE = "myd-yf13x"` (vendor default is `myd-yf13x` but the OE-Core template defaults to `qemux86-64`)
- Layer dependencies and collection names have been corrected

### Why build/conf/ is tracked in Git

Because our configuration files contain project-specific customizations that differ from any vendor template, they are **checked into Git**. The `.gitignore` intentionally does **not** ignore `yocto-bsp/build/conf/`. This ensures:

- New contributors get a working build configuration immediately after `git clone`
- No manual `oe-init-build-env` bootstrap is required before `make yocto-build`
- Changes to layer configuration are tracked and reviewable via pull requests

Other `build/` subdirectories (`tmp-glibc/`, `cache/`, `downloads/`, `sstate-cache/`) are generated artefacts and **are** ignored.

## Building

### Via Makefile (recommended)

```bash
make bsp-login       # Authenticate with GHCR (one-time)
make bsp-fetch       # Download BSP tarballs
make bsp-extract     # Extract layers
make yocto-check     # Validate layer configuration
make yocto-build     # Build myir-image-core for myd-yf13x
```

Override machine or image:
```bash
make yocto-build MACHINE=myd-yf13x-emmc IMAGE=myir-image-core
```

### Manually

```bash
source yocto-bsp/layers/openembedded-core/oe-init-build-env yocto-bsp/build
MACHINE=myd-yf13x bitbake myir-image-core
```

### Useful BitBake Commands

```bash
bitbake-layers show-layers       # Verify layer configuration
bitbake-layers show-appends      # Show all .bbappend files
bitbake -e myir-image-core | grep ^TMPDIR=   # Check actual TMPDIR
bitbake -c cleansstate <recipe>  # Force rebuild of a recipe
```

## Build Artefacts Location

```
yocto-bsp/build/tmp-glibc/deploy/images/myd-yf13x/
├── myir-image-core-myd-yf13x.ext4           # Root filesystem
├── myir-image-core-myd-yf13x.tar.xz         # Root filesystem (tarball)
├── st-image-bootfs-nodistro-myd-yf13x.ext4  # Boot partition (kernel + DTB)
├── st-image-vendorfs-nodistro-myd-yf13x.ext4
├── st-image-userfs-nodistro-myd-yf13x.ext4
├── arm-trusted-firmware/                     # TF-A binaries
│   ├── tf-a-myb-stm32mp135x-*-sdcard.stm32
│   ├── tf-a-myb-stm32mp135x-*-usb.stm32
│   └── metadata.bin
├── fip/                                      # FIP images (OP-TEE + U-Boot)
│   └── fip-myb-stm32mp135x-*-optee.bin
└── flashlayout_myir-image-core/optee/        # Flash layout TSV files
    ├── FlashLayout_sdcard_myb-stm32mp135x-256m-optee.tsv
    ├── FlashLayout_sdcard_myb-stm32mp135x-512m-optee.tsv
    ├── FlashLayout_emmc_myb-stm32mp135x-256m-optee.tsv
    └── FlashLayout_emmc_myb-stm32mp135x-512m-optee.tsv
```

## Flashing with STM32CubeProgrammer

### Prerequisites

- [STM32CubeProgrammer](https://www.st.com/en/development-tools/stm32cubeprog.html) v2.13+
- Board connected via USB
- Board boot switches set to **USB DFU mode**

### Flash Command

Run from the `tmp-glibc/deploy/images/myd-yf13x/` directory (TSV paths are relative):

```bash
cd yocto-bsp/build/tmp-glibc/deploy/images/myd-yf13x/

# SD card (512 MB RAM variant)
STM32_Programmer_CLI -c port=usb1 -w \
  flashlayout_myir-image-core/optee/FlashLayout_sdcard_myb-stm32mp135x-512m-optee.tsv

# eMMC (512 MB RAM variant)
STM32_Programmer_CLI -c port=usb1 -w \
  flashlayout_myir-image-core/optee/FlashLayout_emmc_myb-stm32mp135x-512m-optee.tsv
```

### Flash Layout Variants

| File | Storage | RAM |
|------|---------|-----|
| `FlashLayout_sdcard_…-256m-optee.tsv` | SD card | 256 MB |
| `FlashLayout_sdcard_…-512m-optee.tsv` | SD card | 512 MB |
| `FlashLayout_emmc_…-256m-optee.tsv` | eMMC | 256 MB |
| `FlashLayout_emmc_…-512m-optee.tsv` | eMMC | 512 MB |

### Partition Layout (SD Card)

| # | Name | Type | Content |
|---|------|------|---------|
| 1 | fsbl1 | Binary | TF-A first-stage bootloader |
| 2 | fsbl2 | Binary | TF-A backup copy |
| 3 | metadata1/2 | Binary | A/B boot metadata |
| 4 | fip-a | FIP | OP-TEE + U-Boot |
| 5 | bootfs | ext4 | Kernel + device trees |
| 6 | vendorfs | ext4 | Vendor data |
| 7 | rootfs | ext4 | Root filesystem (myir-image-core) |
| 8 | userfs | ext4 | User data |

## Common Build Errors & Fixes

### Upstream Repo Branch Renamed (master → main)

Error: `Unable to find revision … in branch master even from upstream`

Cause: Upstream repository renamed default branch. The BSP's recipe still references the old branch.

Fix: Create a `.bbappend` in `meta-imf` overriding `SRC_URI` with the correct branch:

```bash
# software/meta-imf/recipes-support/libiio/libiio_git.bbappend
SRC_URI = "git://github.com/analogdevicesinc/libiio.git;protocol=https;branch=main \
           file://0001-CMake-Move-include-CheckCSourceCompiles-before-its-m.patch \
          "
```

This was needed for `libiio-0.23` (analogdevicesinc/libiio.git: `master` → `main`).

### Wrong LAYERDEPENDS Collection Names

Error: `Layer 'meta-imf' depends on layer 'meta-oe', but this layer is not enabled`

Fix: Use collection names in `LAYERDEPENDS`, not directory names:
```
# Wrong:
LAYERDEPENDS_meta-imf = "core meta-oe meta-networking meta-myir"

# Correct:
LAYERDEPENDS_meta-imf = "core openembedded-layer networking-layer stm-myir-st"
```

### Multiple Providers Warning

Warning: `Multiple providers are available for runtime nativesdk-fiptool-stm32mp`

Fix (optional): Add to `local.conf`:
```
PREFERRED_RPROVIDER_nativesdk-fiptool-stm32mp = "nativesdk-tf-a-myir"
```

### No bb Files Matched BBFILE_PATTERN

Warning: `No bb files in default matched BBFILE_PATTERN_meta-imf`

This is informational – the custom layer has no `.bb` recipes yet (only `.bbappend` files). Safe to ignore.

### Fetch Failures (ftp.gnu.org etc.)

Warning: `Failed to fetch URL https://ftp.gnu.org/…, attempting MIRRORS if available`

These are transient network issues. BitBake automatically retries via mirror URLs. Usually resolves itself. If persistent, set a local download mirror in `local.conf`:
```
PREMIRRORS:prepend = "https://ftp.gnu.org/.* https://ftpmirror.gnu.org/ \n"
```
