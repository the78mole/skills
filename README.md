# Skills – Persönliche Copilot Skill-Sammlung

Dieses Repository ist die **zentrale Ablage für meine eigenen GitHub Copilot Skills**.
Es dient als Single Source of Truth: Skills werden hier gepflegt und von hier aus in
die lokale User-Skill-Konfiguration (`~/.copilot/skills/`) eingespielt, damit sie in
allen Projekten auf dem Rechner verfügbar sind.

Zusätzlich enthält das Repo Renovate- und CI/CD-Templates, die als Vorlage für neue
Projekte dienen, sowie die globale `copilot-instructions.md`.

---

## Repository Layout

```
.
├── .github/
│   ├── copilot-instructions.md   ← Globale Copilot-Regeln (CI/CD, Renovate)
│   └── skills -> ../skills       ← Symlink → skills/ (für Repo-lokale Nutzung)
├── skills/                       ← Primärer Ablageort aller Skills
│   ├── cjk-filename-renaming/   ← Umbenennen von CJK/GBK-Dateinamen
│   ├── devcontainer-setup/      ← Dev Container Konfiguration & Features
│   ├── embedded-git-hygiene/    ← Git-Hygiene für Embedded-Projekte (Yocto/KiCad)
│   ├── ghcr-oci-artifacts/      ← Große Binaries als OCI-Artefakte in GHCR
│   ├── nuxt-gh-pages/           ← Nuxt 4 + GitHub Pages Deployment
│   ├── python-uv/               ← Python mit uv & pyproject.toml
│   ├── gh-pages-maintenance/    ← Betrieb & Wartung von GitHub Pages Sites
│   └── yocto-build-and-flash/   ← Yocto Build & Flash für MYD-YF13X
└── templates/
    ├── renovate.json              ← Renovate-Basiskonfiguration
    └── gh-pages-renovate-workflow.yml  ← Optimierter GitHub Pages Workflow
```

---

## Skills in die lokale User-Konfiguration einbinden

Copilot lädt User-Skills aus `~/.copilot/skills/`. Der einfachste Weg, dieses Repo
als Quelle zu nutzen, ist ein **Symlink** vom User-Skills-Ordner auf das ausgecheckte
Repo:

```bash
# Einmalig einrichten (Repo muss bereits geklont sein)
SKILLS_REPO=~/GIT/the78mole/skills

mkdir -p ~/.copilot
ln -sfn "$SKILLS_REPO/skills" ~/.copilot/skills
```

Danach zeigt `~/.copilot/skills/` direkt auf den `skills/`-Ordner dieses Repos.
Ein `git pull` im Repo genügt, um alle Skills auf dem neuesten Stand zu halten –
kein manuelles Kopieren erforderlich.

> **Alternativ – einzelne Skills gezielt kopieren:**
> ```bash
> cp -r ~/GIT/the78mole/skills/skills/nuxt-gh-pages ~/.copilot/skills/
> ```

### Neuen Skill hinzufügen

1. Ordner unter `skills/<skill-name>/` anlegen.
2. `SKILL.md` mit YAML-Frontmatter (`name`, `description`) erstellen.
3. Änderung committen und pushen → steht sofort lokal zur Verfügung.

Minimales Frontmatter-Template:

```markdown
---
name: mein-skill
description: >
  Kurze Beschreibung wann dieser Skill relevant ist.
  Copilot nutzt diese Beschreibung zur automatischen Auswahl.
---

# Skill-Titel

## Abschnitt
...
```

---

## Templates verwenden

### Renovate-Konfiguration

```bash
cp templates/renovate.json /path/to/your-repo/renovate.json
```

Dann die [Renovate GitHub App](https://github.com/apps/renovate) installieren.

> **Tipp:** Umbenennen in `renovate.json5` erlaubt Inline-Kommentare.

### GitHub Pages Workflow

```bash
cp templates/gh-pages-renovate-workflow.yml \
   /path/to/your-repo/.github/workflows/deploy-pages.yml
```

Den `Build site`-Schritt ans eigene Build-Tool anpassen, dann GitHub Pages unter
**Settings → Pages → Source: GitHub Actions** aktivieren.

### Globale Copilot-Anweisungen

Die `.github/copilot-instructions.md` wird von Copilot automatisch eingelesen, wenn
sie im `.github/`-Ordner eines geklonten Repos liegt. Für eine **globale** Wirkung
in VS Code:

1. `Strg+,` → nach **"GitHub Copilot: Instructions"** suchen.
2. Pfad zur lokalen `copilot-instructions.md` eintragen.

---

## Enthaltene Skills im Überblick

| Skill | Beschreibung |
|---|---|
| `cjk-filename-renaming` | Dateien mit chinesischen (GBK/UTF-8) Zeichen in Dateinamen umbenennen |
| `devcontainer-setup` | Dev Container auf Ubuntu 26.04 mit projektspezifischen Features konfigurieren |
| `embedded-git-hygiene` | `.gitignore`, Verzeichnisstruktur und Vendor-Handling für Yocto/KiCad |
| `ghcr-oci-artifacts` | Große Binaries (>2 GB) per `oras` in GHCR hochladen und verwalten |
| `nuxt-gh-pages` | Nuxt 4 SSG-Blogs auf GitHub Pages betreiben inkl. Content v3 Migration |
| `python-uv` | Python-Projekte mit `uv` und `pyproject.toml`; aktuelle PyPI-Versionen abfragen |
| `gh-pages-maintenance` | CI/CD, Redirects, AdSense, Content-Schema für GH Pages Sites (Nuxt) |
| `yocto-build-and-flash` | Yocto-Build für MYiR MYD-YF13X (STM32MP135), Flash via STM32CubeProgrammer |

---

## Renovate Dependency Dashboard

Sobald Renovate mindestens einmal gelaufen ist, erstellt es ein
**Dependency Dashboard**-Issue als Kontrollpanel für alle ausstehenden Updates:

| Bereich | Inhalt |
|---|---|
| **Rate-limited / Blocked** | PRs, die durch `prConcurrentLimit` zurückgehalten werden |
| **Awaiting Schedule** | Updates, die auf das Wochenend-Fenster warten |
| **Open PRs** | Aktuell offene Renovate Pull Requests |
| **Detected dependencies** | Vollständiger Überblick aller getrackten Abhängigkeiten |

---

## Contributing

Pull Requests und Issues sind willkommen. Bitte die Konventionen aus
`.github/copilot-instructions.md` beachten.
