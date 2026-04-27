---
name: embedded-git-hygiene
description: Best practices for making embedded Linux projects (Yocto, KiCad, STM32MP1) Git-ready. Use this when structuring repositories for embedded development, handling vendor BSP files, or setting up .gitignore for Yocto and KiCad projects.
---

# Git Hygiene for Embedded Projects

## Large Binary Files

- Vendor BSP tarballs (kernel, U-Boot, TF-A, OP-TEE, Yocto DL cache) do **not** belong in Git.
- Strategy: Host in GHCR as OCI artifacts + provide a download script in the repo.
- The Yocto `downloads/` directory (`DL_DIR`) is reproducible and can be re-downloaded at any time.
- GitHub Releases have a **2 GB per-file limit** – GHCR via `oras` is the alternative for larger files (see skill `ghcr-oci-artifacts`).

## Identifying and Removing Redundancies

- Manufacturers often ship **ZIP archives + already extracted directories**. Keep only one (prefer extracted).
- Typical examples:
  - `BSP.zip` (14 GB) + extracted `BSP/` folder → delete the ZIP
  - `03_Tools.zip` + extracted `03_Tools/` → delete the ZIP
  - `HardwareFiles.zip` + extracted `HardwareFiles/` → delete the ZIP
- Detection method: `find . -name "*.zip" -exec sh -c 'dir="${1%.zip}"; [ -d "$dir" ] && echo "REDUNDANT: $1"' _ {} \;`

## Recommended Directory Structure

Clearly separate custom development from unmodified vendor data (`vendor/`):

```
hardware/
├── kicad/              # Custom PCB files (KiCad projects)
├── bom/                # Bill of materials
└── vendor/myir/        # Vendor reference data
    ├── datasheets/     # Datasheets
    ├── pcb/            # Reference PCB files
    └── manuals/        # User manuals
software/
├── meta-imf/           # Custom Yocto layer
├── config/             # Custom configuration (firewall, WireGuard)
└── vendor/             # BSP download instructions + fetch script
    ├── README.md       # Vendor source documentation
    └── fetch-vendor-bsp.sh  # Download script (oras pull)
scripts/                # Project-wide scripts
docs/                   # Documentation
```

## .gitignore for Yocto/KiCad Projects

### Yocto Build System
```gitignore
# Yocto build outputs
build/
sstate-cache/
downloads/
tmp/
poky/

# Yocto stamps and signatures
*.do_*
*.sigdata.*
*.sigbasedata.*

# Build images
*.wic
*.wic.bz2
*.wic.gz
*.ext4
*.rootfs.*
```

### KiCad
```gitignore
# KiCad backups
*-backups/
*.kicad_pcb-bak
*.kicad_sch-bak
_autosave-*
fp-info-cache
*.kicad_prl
```

### Large Binary Files
```gitignore
# Compressed archives (hosted externally)
*.tar.bz2
*.tar.xz
*.tar.gz
*.tar.zst
*.rar
*.zip

# Staging directory for external uploads
.release-staging/
```

## Vendor BSP Management

### Provide a Fetch Script

Create a `scripts/fetch-vendor-bsp.sh` that downloads vendor files from GHCR via `oras pull`:

```bash
#!/usr/bin/env bash
set -euo pipefail

REGISTRY="ghcr.io/<OWNER>/<REPO>/vendor-bsp"
TAG="1.0"
TARGET_DIR="software/vendor/bsp"

mkdir -p "$TARGET_DIR"

PACKAGES=(
    "package-name|Original-Filename.tar.bz2"
)

for entry in "${PACKAGES[@]}"; do
    IFS='|' read -r pkg file <<< "$entry"
    if [[ -f "$TARGET_DIR/$file" ]]; then
        echo "SKIP: $file (exists)"
        continue
    fi
    echo "Downloading $file..."
    (cd "$TARGET_DIR" && oras pull "$REGISTRY/$pkg:$TAG")
done
```

### Push via GitHub Actions (Recommended)

If packages need to be automatically public and linked to the repo, use a GitHub Actions workflow. The `GITHUB_TOKEN` ensures that packages inherit the repository's visibility.
