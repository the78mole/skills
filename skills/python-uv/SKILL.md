---
name: python-uv
description: >
  Modern Python project setup with uv and pyproject.toml. Use this when creating
  new Python scripts or projects, adding dependencies, setting up virtual environments,
  configuring linting/formatting (ruff), or writing GitHub Actions workflows for Python.
  Always look up the latest package versions on PyPI before pinning.
user-invocable: true
---

# Python mit uv & pyproject.toml

## Versionsstrategie: immer aktuell starten

Bevor ein Paket in `pyproject.toml` eingetragen wird, **die aktuelle Version auf PyPI nachschlagen**:

```bash
# Einzelne Version abfragen
curl -s https://pypi.org/pypi/<paket>/json | python3 -c \
  "import json,sys; d=json.load(sys.stdin); print(d['info']['version'])"

# Oder direkt mit uv (zeigt neueste verfügbare Version)
uv add <paket>          # fügt hinzu und pinnt auf aktuelle Version
uv add "<paket>>=x.y"   # mit Untergrenze
```

Bekannte Ausgangswerte (Stand 2026-04-27, regelmäßig prüfen):

| Paket | Aktuelle Version |
|---|---|
| `uv` | 0.11.7 |
| `ruff` | 0.15.12 |
| Python | 3.12 (LTS) / 3.13 (latest) |

---

## Neues Projekt anlegen

```bash
# Projekt mit pyproject.toml initialisieren
uv init <projektname>
cd <projektname>

# Oder in ein bestehendes Verzeichnis
uv init .
```

Erzeugt:
```
<projektname>/
├── .python-version     ← Python-Version (z.B. "3.12")
├── pyproject.toml
├── README.md
└── src/
    └── <projektname>/
        └── __init__.py
```

---

## pyproject.toml – Standardstruktur

```toml
[project]
name = "mein-projekt"
version = "0.1.0"
description = "Kurzbeschreibung"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    # Laufzeit-Dependencies hier (uv add <paket> trägt sie automatisch ein)
]

[project.optional-dependencies]
dev = [
    "ruff>=0.15",
    "pytest>=8.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]   # pycodestyle, pyflakes, isort, pyupgrade

[tool.ruff.format]
quote-style = "double"
```

---

## Virtuelle Umgebung & Abhängigkeiten

```bash
# Umgebung erstellen + Deps installieren (alles in einem)
uv sync

# Laufzeit-Paket hinzufügen
uv add requests httpx

# Dev-Paket hinzufügen
uv add --dev ruff pytest

# Paket entfernen
uv remove <paket>

# Alle Pakete auf neueste kompatible Version bringen
uv lock --upgrade
uv sync

# Skript direkt ausführen (ohne manuelles Aktivieren)
uv run python src/mein_projekt/main.py
uv run pytest
```

### Wann `uv sync` vs. `uv run`?

- `uv sync` – Umgebung explizit aktualisieren (z.B. nach `git pull` oder `uv add`)
- `uv run <cmd>` – Befehl in der Projekt-Umgebung ausführen, synct automatisch

---

## Einzel-Skripte mit Inline-Dependencies

Für kleine Standalone-Skripte ohne eigenes Projekt:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#   "httpx>=0.28",
#   "rich>=13",
# ]
# ///

import httpx
from rich import print

resp = httpx.get("https://api.github.com")
print(resp.json())
```

```bash
uv run script.py   # installiert Deps on-the-fly in isolierter Umgebung
```

---

## Linting & Formatting mit ruff

```bash
# Formatieren
uv run ruff format .

# Lint + Auto-Fix
uv run ruff check --fix .

# Beide in einem (CI-geeignet, kein Fix)
uv run ruff format --check . && uv run ruff check .
```

---

## GitHub Actions Workflow

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
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v5
        with:
          version: "0.11.7"          # aus pyproject.toml / Renovate hält das aktuell
          enable-cache: true

      - name: Set up Python
        run: uv python install

      - name: Install dependencies
        run: uv sync --frozen        # --frozen: schlägt fehl wenn uv.lock veraltet

      - name: Lint (ruff)
        run: uv run ruff check . && uv run ruff format --check .

      - name: Test (pytest)
        run: uv run pytest
```

> **`astral-sh/setup-uv`** cacht die uv-Binary und die virtuelle Umgebung.
> Renovate aktualisiert die `version`-Zeile automatisch (Manager: `github-actions`).

---

## Renovate für uv-Projekte

Renovate erkennt `uv.lock` und `pyproject.toml` automatisch über den `uv`-Manager
(ab Renovate 38+). Für die Gruppierung von Minor/Patch-Updates in einen PR
die `renovate.json` wie folgt ergänzen (oder das Template aus `templates/renovate.json`
in diesem Repo verwenden – dort ist die Regel bereits enthalten):

```json
{
  "packageRules": [
    {
      "description": "Group Python non-major dependency updates (uv + pip)",
      "matchManagers": ["uv", "pip_requirements", "pip-compile", "pipenv", "poetry"],
      "matchUpdateTypes": ["minor", "patch"],
      "groupName": "Python dependencies (non-major)"
    }
  ]
}
```

---

## Häufige Fehler & Fixes

| Fehler | Ursache | Fix |
|---|---|---|
| `No solution found` beim `uv sync` | Versionskonflikte zwischen Paketen | `uv add` mit expliziter Version; `uv tree` zur Analyse |
| `uv.lock is outdated` in CI | `--frozen` + veraltetes Lock-File | Lokal `uv lock` ausführen und committen |
| `ModuleNotFoundError` trotz `uv sync` | Skript läuft außerhalb der venv | `uv run python ...` statt `python ...` |
| ruff findet keine Config | `pyproject.toml` fehlt im CWD | `ruff check --config pyproject.toml .` |
| `requires-python` Konflikt | System-Python < Mindestanforderung | `.python-version` Datei anlegen; `uv python install 3.12` |
