---
name: ghcr-oci-artifacts
description: Guide for uploading and managing large binary files as OCI artifacts in GitHub Container Registry (GHCR) using oras CLI. Use this when working with large vendor binaries, BSP tarballs, or any files exceeding the GitHub Releases 2GB limit.
---

# GHCR – OCI Artifacts (Not Container Images)

## oras CLI

- `oras` (OCI Registry As Storage) can push arbitrary files as OCI artifacts to container registries – not just Docker images.
- Installation:
  ```bash
  curl -fSL https://github.com/oras-project/oras/releases/download/v1.3.0/oras_1.3.0_linux_amd64.tar.gz \
    | sudo tar xz -C /usr/local/bin oras
  ```
- Login via `gh`:
  ```bash
  gh auth token | oras login ghcr.io -u <USERNAME> --password-stdin
  ```
- **Important**: `oras push` expects **relative paths**. For absolute paths, either set `--disable-path-validation` or change into the directory first:
  ```bash
  (cd /path/to/directory && oras push ghcr.io/owner/repo/pkg:tag file.tar.gz:application/octet-stream)
  ```

## GHCR Has No File Size Limit

- GitHub **Releases** have a **2 GB per-file limit** (HTTP 422: `size must be less than 2147483648`).
- GHCR (Container Registry) has **no such limit** – 13 GB files work without issues.
- Therefore, GHCR via `oras` is the better choice for large vendor binaries.

## Packages Are Private by Default

- New GHCR packages are **always created as private**, regardless of repository visibility.
- This is a deliberate GitHub security decision.
- **Exception**: Packages pushed via a GitHub Actions workflow using the `GITHUB_TOKEN` inherit the repository's visibility and are automatically linked to the repo.

## Visibility Changes Only via WebUI

- The GitHub REST API provides **no PATCH endpoint** for user packages (`/user/packages/container/{name}`).
- GET and DELETE work, but visibility changes are only possible through the package settings page in the WebUI:
  ```
  https://github.com/users/<USER>/packages/container/package/<ENCODED_NAME>/settings
  ```
  → Danger Zone → "Change package visibility" → Public.
- For nested package names (containing slashes), use `%2F` in the URL.
- Even with a classic PAT (`ghp_...`) with all package scopes (`read:packages`, `write:packages`, `delete:packages`), PATCH returns 404.
- The PATCH endpoint reportedly exists **for organization packages**, but definitely not for user packages.

## OAuth Token (gho\_) vs. PAT (ghp\_)

- `gh auth` generates OAuth Device Flow Tokens (`gho_...`).
- These tokens have limited functionality for some API endpoints.
- Classic PATs (`ghp_...`) generally have broader API access, but don't help with visibility changes either (the endpoint doesn't exist).
- Check token scopes:
  ```bash
  curl -sI -H "Authorization: Bearer $TOKEN" https://api.github.com/user | grep x-oauth-scopes
  ```

## gh auth – Extend Scopes

- Request additional scopes for the `gh` token:
  ```bash
  gh auth refresh --scopes write:packages,read:packages,delete:packages
  ```
- Opens a device flow in the browser for confirmation.

## Recommended Workflow for Large Binaries

1. Move files to a staging directory (e.g., `.release-staging/`)
2. Run `oras push` with relative paths
3. Provide a `fetch-vendor-bsp.sh` script in the repo that uses `oras pull`
4. If packages need to be public: Push via WebUI or use a GitHub Actions workflow (`GITHUB_TOKEN` inherits repo visibility)
