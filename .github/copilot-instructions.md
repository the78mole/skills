# GitHub Copilot Instructions

These instructions govern how Copilot should assist in this repository and any project derived from it.
They enforce modern, automation-first standards for dependency management and CI/CD workflows.

---

## Dependency Management: Renovate over Dependabot

- **Always** create a `renovate.json` (or `renovate.json5`) at the repository root when setting up a new project.
- **Never** create or suggest a `dependabot.yml` for version updates. Dependabot may only be used for security alerts if Renovate is not yet configured.
- Use the template from `templates/renovate.json` in this repository as the starting point.
- Reference the [Renovate Docs](https://docs.renovatebot.com/) for advanced configuration.

```jsonc
// Minimal renovate.json to bootstrap a new project
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:standard",
    ":pinDigests"
  ]
}
```

---

## GitHub Actions: Version Requirements

- **Always** use `actions/checkout@v4` or newer. Never use v1, v2, or v3.
- **Always** use `actions/setup-node@v4` or newer.
- **Always** use `actions/setup-python@v5` or newer.
- **Always** use `actions/upload-artifact@v4` and `actions/download-artifact@v4` or newer.
- For any GitHub-owned action (`actions/*`, `github/*`), use **v4+** as the minimum version.
- Pin action versions to their **full SHA digest** in security-sensitive workflows (Renovate will keep these up to date).

### Node.js

- Default to **Node.js 22** (current LTS) for new projects.
- Use `node-version: '22'` in `actions/setup-node`.

### Python

- Default to **Python 3.12** or newer for new projects.
- Use `python-version: '3.12'` in `actions/setup-python`.

---

## Web-Search Agent: Version Verification

- Before pinning any package version or action tag in a workflow or configuration file, use the **`@web-search` agent** to verify the latest stable major release.
- Prompt pattern: `@web-search What is the latest stable release of <package/action>?`
- This applies to:
  - npm packages (`package.json`, `package-lock.json`)
  - PyPI packages (`requirements.txt`, `pyproject.toml`)
  - GitHub Actions (`.github/workflows/*.yml`)
  - Docker base images (`Dockerfile`)
  - Renovate itself

---

## Workflow Best Practices

- Use `permissions:` blocks to apply **least-privilege** to all workflows.
- Set `contents: read` as the default; only escalate where required (e.g., `contents: write` for deployments).
- Always add `concurrency:` groups to prevent duplicate runs on rapid pushes.
- Prefer composite actions or reusable workflows over copy-pasted job steps.

### Minimal Workflow Template

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

---

## Renovate Dashboard

- Enable the **Renovate Dependency Dashboard** issue in every repository so that pending updates are visible at a glance.
- Set `"dependencyDashboard": true` in `renovate.json`.
- Schedule all update PRs for weekends to avoid weekday workflow disruption.

---

## Summary Checklist for New Projects

- [ ] `renovate.json` created from `templates/renovate.json`
- [ ] No `dependabot.yml` version-update entries exist
- [ ] All GitHub Actions use v4+
- [ ] Node.js ≥ 22 / Python ≥ 3.12 in workflows
- [ ] Workflow permissions follow least-privilege
- [ ] `concurrency:` group defined in every workflow
- [ ] `dependencyDashboard: true` in `renovate.json`
