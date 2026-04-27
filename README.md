# Skills – Copilot & Renovate Skill Collection

A curated collection of templates and instructions for keeping GitHub repositories
up-to-date with **Renovate** as the primary dependency-management tool and
**GitHub Copilot** tuned to modern CI/CD standards.

---

## Repository Layout

```
.
├── .github/
│   └── copilot-instructions.md   ← Copilot governance rules
└── templates/
    ├── renovate.json              ← Best-practice Renovate config
    └── gh-pages-renovate-workflow.yml  ← Optimised GitHub Pages workflow
```

---

## Quick Start

### 1 – Add the Renovate configuration

Copy `templates/renovate.json` to the **root** of your target repository:

```bash
cp templates/renovate.json /path/to/your-repo/renovate.json
```

Then install the [Renovate GitHub App](https://github.com/apps/renovate) on that
repository (or enable it via your organisation's Renovate bot).

> **Tip:** Rename the file to `renovate.json5` if you want to keep the inline
> comments – Renovate supports both formats.

### 2 – Copy the Copilot instructions into your IDE

The `.github/copilot-instructions.md` file is picked up automatically by
GitHub Copilot in VS Code and other supported editors when the file is present
in the repository root's `.github/` folder.

To apply the same rules **globally** across all your repositories:

1. Open VS Code Settings (`Ctrl+,` / `Cmd+,`).
2. Search for **"GitHub Copilot: Instructions"**.
3. Add the path to your local copy of `copilot-instructions.md`.

Or use the GitHub Copilot CLI:

```bash
gh copilot config set instructions "$(cat .github/copilot-instructions.md)"
```

### 3 – Set up GitHub Pages deployment (optional)

Copy the workflow template to your repository:

```bash
cp templates/gh-pages-renovate-workflow.yml \
   /path/to/your-repo/.github/workflows/deploy-pages.yml
```

Adjust the `Build site` step to match your project's build tooling, then enable
GitHub Pages in **Settings → Pages → Source: GitHub Actions**.

---

## Using the Renovate Dependency Dashboard

Once Renovate is installed and has run at least once, it creates a
**Dependency Dashboard** issue in your repository.  This single issue acts as a
control panel for all pending updates:

| Section | What it shows |
|---|---|
| 🔴 **Rate-limited / Blocked** | PRs that are on hold due to `prConcurrentLimit` |
| 🟡 **Awaiting Schedule** | Updates queued for the weekend schedule |
| 🟢 **Open PRs** | Currently open Renovate pull requests |
| ⚪ **Detected dependencies** | Full inventory of every dependency Renovate tracks |

**Useful actions:**

- **Tick a checkbox** next to a pending update to trigger it immediately,
  bypassing the schedule.
- **Comment `rebase`** on any Renovate PR to force a rebase against the base branch.
- **Close and reopen** the Dashboard issue to force Renovate to re-evaluate all rules.

---

## Design Principles

| Principle | Implementation |
|---|---|
| **No noise** | Weekend-only schedule; non-major updates grouped into a single PR |
| **Safety first** | Major updates require manual review; vulnerability alerts bypass the schedule |
| **Automation** | Minor/patch updates for trusted sources automerge when CI is green |
| **Auditability** | SHA-pinned actions; Dependency Dashboard issue for full visibility |
| **Modern stack** | Node.js 22, Python 3.12, Actions v4+ enforced via Copilot instructions |

---

## Contributing

Pull requests and issues are welcome.  Please follow the conventions described
in `.github/copilot-instructions.md` when contributing.
