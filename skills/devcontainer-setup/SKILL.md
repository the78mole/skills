---
name: devcontainer-setup
description: >
  Best practices for creating and maintaining Dev Container configurations
  (.devcontainer/devcontainer.json). Use this when setting up a new devcontainer,
  adding features to an existing one, or adapting the container to a specific
  project type (Python/uv, Node.js, embedded/Yocto, Java, Rust, Terraform).
  Base image is always ubuntu:26.04 unless the project requires otherwise.
user-invocable: true
---

# Dev Container Setup

## Basis-Image

Standardmäßig wird **Ubuntu 26.04** als Basis verwendet:

```json
"image": "mcr.microsoft.com/devcontainers/base:ubuntu-26.04"
```

Alternativen nur bei zwingenden Gründen (z.B. Yocto benötigt spezifische Distro):

| Bild | Wann |
|---|---|
| `base:ubuntu-26.04` | Standard – alle neuen Projekte |
| `base:ubuntu-24.04` | Wenn ein Tool noch nicht Ubuntu 26 unterstützt |
| `base:debian-12` | Nur wenn Debian-spezifische Pakete benötigt werden |

---

## Minimale devcontainer.json-Vorlage

Datei anlegen unter `.devcontainer/devcontainer.json`:

```json
{
  "name": "<Projektname>",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-26.04",

  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {
      "installZsh": true,
      "configureZshAsDefaultShell": true,
      "installOhMyZsh": true,
      "installOhMyZshConfig": true,
      "upgradePackages": false,
      "username": "ubuntu"
    },
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/git:1": {}
  },

  "remoteUser": "ubuntu",

  "customizations": {
    "vscode": {
      "extensions": [],
      "settings": {}
    }
  }
}
```

`common-utils` und `git` sind **immer** dabei. Alle weiteren Features werden
projektabhängig ergänzt (siehe Abschnitt "Feature-Auswahl nach Projekttyp").

---

## Feature-Auswahl nach Projekttyp

### Python-Projekte (mit uv)

```json
"ghcr.io/devcontainers/features/python:1": {
  "version": "3.12",
  "installTools": false
},
"ghcr.io/devcontainers/features/github-cli:1": {}
```

`installTools: false` – pip/virtualenv werden nicht benötigt, da uv alles übernimmt.
uv selbst im `postCreateCommand` installieren:

```json
"postCreateCommand": "curl -LsSf https://astral.sh/uv/install.sh | sh"
```

### Node.js-Projekte

```json
"ghcr.io/devcontainers/features/node:1": {
  "version": "22",
  "nodeGypDependencies": false
}
```

### Projekte mit Docker-Builds (CI-ähnlich)

```json
"ghcr.io/devcontainers/features/docker-in-docker:2": {
  "version": "latest",
  "moby": true
}
```

> **Docker-in-Docker vs. Docker-outside-of-Docker**: `docker-in-docker` startet
> eine eigene Docker-Instanz im Container. `docker-outside-of-docker` teilt den
> Host-Socket. Für CI-ähnliche Builds ist `docker-in-docker` zu bevorzugen.

### Embedded / Yocto-Projekte

```json
"ghcr.io/devcontainers/features/github-cli:1": {}
```

Yocto-Build-Dependencies über `postCreateCommand` als apt installieren:

```json
"postCreateCommand": "sudo apt-get update && sudo apt-get install -y gawk wget git-core diffstat unzip texinfo gcc-multilib build-essential chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping file lz4 zstd"
```

### Java-Projekte

```json
"ghcr.io/devcontainers/features/java:1": {
  "version": "21",
  "jdkDistro": "ms"
}
```

### Kubernetes / Terraform

```json
"ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
  "version": "latest",
  "helm": "latest",
  "minikube": "none"
},
"ghcr.io/devcontainers/features/terraform:1": {}
```

### Rust-Projekte

```json
"ghcr.io/devcontainers/features/rust:1": {
  "version": "latest",
  "profile": "minimal"
}
```

---

## Projekt-Feature: precommit-setup (lokal)

Für Projekte mit eigenem `./features/precommit-setup`:

```json
"./features/precommit-setup": {
  "version": "latest",
  "installMarkdownlint": true,
  "installPyYAML": true
}
```

Lokale Features liegen unter `.devcontainer/features/<name>/devcontainer-feature.json`.

---

## Vollständiges Beispiel: Python-Projekt mit CI-Fähigkeit

```json
{
  "name": "my-python-project",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-26.04",

  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {
      "installZsh": true,
      "configureZshAsDefaultShell": true,
      "installOhMyZsh": true,
      "installOhMyZshConfig": true,
      "upgradePackages": false,
      "username": "ubuntu"
    },
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.12",
      "installTools": false
    },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },

  "postCreateCommand": "curl -LsSf https://astral.sh/uv/install.sh | sh && uv sync",

  "remoteUser": "ubuntu",

  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "charliermarsh.ruff",
        "GitHub.copilot",
        "GitHub.copilot-chat"
      ],
      "settings": {
        "python.defaultInterpreterPath": ".venv/bin/python",
        "[python]": {
          "editor.formatOnSave": true,
          "editor.defaultFormatter": "charliermarsh.ruff"
        }
      }
    }
  }
}
```

---

## Vollständiges Beispiel: Node.js / Nuxt-Projekt

```json
{
  "name": "my-nuxt-project",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-26.04",

  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {
      "installZsh": true,
      "configureZshAsDefaultShell": true,
      "installOhMyZsh": true,
      "installOhMyZshConfig": true,
      "upgradePackages": false,
      "username": "ubuntu"
    },
    "ghcr.io/devcontainers/features/git:1": {},
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": {
      "version": "22"
    }
  },

  "postCreateCommand": "npm ci",

  "remoteUser": "ubuntu",

  "forwardPorts": [3000],

  "customizations": {
    "vscode": {
      "extensions": [
        "Vue.volar",
        "GitHub.copilot",
        "GitHub.copilot-chat",
        "dbaeumer.vscode-eslint"
      ]
    }
  }
}
```

---

## Aktuelle Feature-Versionen (Stand 2026-04-27)

Immer vor Eintragen auf dem aktuellen Stand prüfen:
```bash
curl -s "https://raw.githubusercontent.com/devcontainers/features/main/src/<feature>/devcontainer-feature.json" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['version'])"
```

| Feature | Version | OCI-Referenz |
|---|---|---|
| `common-utils` | 2.5.7 | `ghcr.io/devcontainers/features/common-utils:2` |
| `git` | 1.3.5 | `ghcr.io/devcontainers/features/git:1` |
| `github-cli` | 1.1.0 | `ghcr.io/devcontainers/features/github-cli:1` |
| `python` | 1.8.0 | `ghcr.io/devcontainers/features/python:1` |
| `node` | 1.7.1 | `ghcr.io/devcontainers/features/node:1` |
| `docker-in-docker` | 2.16.1 | `ghcr.io/devcontainers/features/docker-in-docker:2` |
| `rust` | 1.5.0 | `ghcr.io/devcontainers/features/rust:1` |
| `java` | 1.8.0 | `ghcr.io/devcontainers/features/java:1` |
| `kubectl-helm-minikube` | 1.3.1 | `ghcr.io/devcontainers/features/kubectl-helm-minikube:1` |
| `terraform` | 1.4.2 | `ghcr.io/devcontainers/features/terraform:1` |

> **Renovate** hält die Major-Pins (`:1`, `:2`) automatisch aktuell, wenn der
> `devcontainer` Manager in `renovate.json` aktiv ist (in `config:standard` enthalten).

---

## Häufige Fehler & Fixes

| Fehler | Ursache | Fix |
|---|---|---|
| Container baut, aber User hat keine sudo-Rechte | `remoteUser` nicht gesetzt | `"remoteUser": "ubuntu"` in `devcontainer.json` |
| `postCreateCommand` schlägt fehl | Läuft als root, PATH nicht gesetzt | `"remoteUser"` vor `postCreateCommand` sicherstellen |
| Oh-My-Zsh startet nicht | `username` in `common-utils` falsch | Muss mit `remoteUser` übereinstimmen (`ubuntu`) |
| `docker: command not found` im Container | Feature fehlt | `docker-in-docker` oder `docker-outside-of-docker` hinzufügen |
| Feature-Version nicht auflösbar | Major-Tag veraltet | Versionstabelle oben prüfen; Major-Tag im Ref aktualisieren |
| uv nicht gefunden nach `postCreateCommand` | PATH nicht neu geladen | `source $HOME/.local/bin/env` oder neues Terminal öffnen |
