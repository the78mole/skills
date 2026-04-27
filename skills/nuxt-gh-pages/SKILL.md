---
name: nuxt-gh-pages
description: >
  Nuxt 4 + GitHub Pages expertise: create new static blogs or maintain existing ones.
  Use for: setting up a new Nuxt SSG site on GitHub Pages; troubleshooting CI/CD build
  failures; migrating @nuxt/content v2→v3 API; fixing SSR/hydration mismatches; managing
  Tailwind CSS + @nuxtjs/tailwindcss version conflicts; configuring Renovate for safe
  dependency updates; AdSense consent flow; DSGVO consent banner with nuxt-gtag;
  favicon SVG setup; debugging npm run generate errors.
user-invocable: true
---

# Nuxt 4 + GitHub Pages – Erfahrungswissen

## Stack (bewährt)

| Paket | Version | Hinweis |
|---|---|---|
| nuxt | ^4.x | SSG via `nuxt generate` |
| @nuxt/content | ^3.x | SQLite-basiert, Zod-Schemas nötig |
| @nuxtjs/tailwindcss | ^6.x | Hat tailwindcss `~3.4` als **reguläre Dep** (nicht Peer-Dep) |
| tailwindcss | `^3.4.17` | Nicht auf v4 aktualisieren bis @nuxtjs/tailwindcss v7+ |
| better-sqlite3 | `^12.x` | devDependency, Pflicht für @nuxt/content v3 |
| @nuxt/image | optional | Bildoptimierung |

---

## Neue Seite erstellen

### 1. Projekt-Scaffold

```bash
npx nuxi init <projektname> --template gh-pages
cd <projektname>
npm install
```

### 2. GitHub Pages konfigurieren (`nuxt.config.ts`)

```ts
export default defineNuxtConfig({
  ssr: true,            // SSG: true + nuxt generate
  nitro: { preset: 'static' },
  app: {
    // Nur nötig wenn Repo nicht <user>.github.io, sondern /<repo>
    // baseURL: '/<repo>/',
  },
})
```

### 3. GitHub Actions Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
jobs:
  build-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run generate
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .output/public
      - uses: actions/deploy-pages@v4
        id: deployment
```

### 4. `content.config.ts` – Pflicht für @nuxt/content v3

Benutzerdefinierte Frontmatter-Felder **müssen** im Zod-Schema definiert werden, damit sie als SQLite-Spalten angelegt werden:

```ts
import { defineContentConfig, defineCollection, z } from '@nuxt/content'

export default defineContentConfig({
  collections: {
    blog: defineCollection({
      type: 'page',
      source: 'blog/**/*.md',
      schema: z.object({
        date: z.string().optional(),
        description: z.string().optional(),
        image: z.string().optional(),
        categories: z.array(z.string()).optional(),
      }),
    }),
    pages: defineCollection({
      type: 'page',
      source: 'pages/**/*.md',
      schema: z.object({
        description: z.string().optional(),
      }),
    }),
  },
})
```

---

## @nuxt/content v2 → v3 Migration

| Alt (v2) | Neu (v3) |
|---|---|
| `queryContent('blog')` | `queryCollection('blog')` |
| `.only(['title', '_path'])` | `.select('title', 'path')` |
| `.sort({ date: -1 })` | `.order('date', 'DESC')` |
| `.find()` | `.all()` |
| `.findOne()` | `.path(route.path).first()` |
| `post._path` | `post.path` (kein Unterstrich) |

**Fehler-Diagnose**: `no such column: "date"` → Frontmatter-Feld fehlt im Zod-Schema in `content.config.ts`.

**Fehler-Diagnose**: `better-sqlite3` interaktiver Prompt in CI → `better-sqlite3` als `devDependency` in `package.json` explizit eintragen.

---

## Tailwind CSS – Versions-Konflikt vermeiden

`@nuxtjs/tailwindcss` v6 hat tailwindcss als **reguläre** Dependency `~3.4.17` (nicht Peer-Dep). Wenn Top-Level auf v4 gesetzt wird, installiert npm **beide** Versionen (v4 Root + v3 nested). Das bricht den Build.

**Renovate absichern** (`renovate.json`):

```json
{
  "packageRules": [
    {
      "matchPackageNames": ["tailwindcss"],
      "allowedVersions": "^3.0.0",
      "description": "tailwindcss v4 inkompatibel mit @nuxtjs/tailwindcss v6"
    },
    {
      "matchPackageNames": ["@nuxtjs/tailwindcss"],
      "matchUpdateTypes": ["major"],
      "labels": ["needs-review"],
      "description": "Major-Update prüfen: evtl. tailwindcss v4 Support"
    }
  ]
}
```

**Wann entsperren?** Wenn `@nuxtjs/tailwindcss` tailwindcss als peerDependency (nicht dependency) listet:
```bash
cat node_modules/@nuxtjs/tailwindcss/package.json | python3 -c \
  "import json,sys; p=json.load(sys.stdin); \
   print('dep:', p.get('dependencies',{}).get('tailwindcss','none')); \
   print('peer:', p.get('peerDependencies',{}).get('tailwindcss','none'))"
```

---

## Favicon – SVG statt ICO

**Problem**: Nuxt-Standard-Template schreibt `{ rel: 'icon', href: '/favicon.ico' }`. Wenn nur `favicon.svg` in `public/` liegt, bleibt das Favicon leer.

**Fix** in `nuxt.config.ts`:
```ts
link: [
  { rel: 'icon', type: 'image/svg+xml', href: '/favicon.svg' },
]
```

SVG-Favicons werden von allen modernen Browsern unterstützt und sind vorzuziehen. `type: 'image/svg+xml'` immer mitangeben, sonst ignorieren manche Browser den Link.

---

## DSGVO Consent-Banner mit nuxt-gtag

### Konfiguration (nuxt.config.ts)

```ts
gtag: {
  initMode: 'manual',  // PFLICHT: GA4 nie automatisch laden
  id: process.env.GOOGLE_ANALYTICS_MEAS_ID || 'G-XXXXXXXXXX',
},
```

### components/ConsentBanner.vue – vollständiges Muster

```vue
<script setup lang="ts">
const consent = useCookie<'accepted' | 'declined' | null>('bizzmark-consent', {
  default: () => null,
  maxAge: 60 * 60 * 24 * 365, // 1 Jahr
  sameSite: 'lax',
})
const { initialize } = useGtag()
const showBanner = computed(() => consent.value === null)

function accept() {
  consent.value = 'accepted'
  initialize()
}
function decline() {
  consent.value = 'declined'
}
onMounted(() => {
  if (consent.value === 'accepted') initialize()
})
</script>
```

### Widerruf auf Datenschutz-Seite

```vue
<script setup lang="ts">
const consent = useCookie<'accepted' | 'declined' | null>('bizzmark-consent')
function revokeConsent() { consent.value = null }
</script>
<template>
  <button @click="revokeConsent">Cookie-Einstellungen ändern</button>
</template>
```

### Datenschutzerklärung – Pflichtinhalte für GA4

- Cookie-Name und Gültigkeit nennen (`bizzmark-consent`, 1 Jahr)
- Rechtsgrundlage: Art. 6 Abs. 1 lit. a DSGVO (Einwilligung)
- Hinweis auf IP-Anonymisierung
- Link zu Google Datenschutzerklärung + Opt-out-Add-on
- Widerruf-Schaltfläche direkt auf der Seite einbauen

---

## Hydration-Mismatches – Ursachen & Fixes

### ConsentBanner / Cookie-abhängige Komponenten

**Problem**: `useCookie()` liefert im SSG-Build `null` (kein Request-Kontext). Banner wird in HTML gerendert. Rückkehrende Besucher mit gespeichertem Cookie sehen beim Hydrate einen anderen Zustand → **Mismatch**.

**Fix**: In `<ClientOnly>` wrappen (nie in SSG-HTML rendern):

```vue
<!-- app.vue -->
<ClientOnly>
  <ConsentBanner />
</ClientOnly>
```

### Dynamische Werte die sich je nach Zeitpunkt unterscheiden

**Problem**: `new Date().getFullYear()` – Build-Jahr (z.B. 2025) ≠ Besuch-Jahr (2026) → Mismatch.

**Fix**: Statischen Fallback + `onMounted` Update:

```vue
<script setup>
const currentYear = ref(2026)
onMounted(() => { currentYear.value = new Date().getFullYear() })
</script>
<!-- Template: {{ currentYear }} -->
```

### Ad-Komponenten (AdSense `<ins>`)

`<ins class="adsbygoogle">` darf nie SSR-seitig gerendert werden, da `adsbygoogle.push()` nur client-seitig funktioniert.

**Fix**: `isMounted` Guard:

```vue
<script setup>
const isMounted = ref(false)
onMounted(() => { isMounted.value = true })
</script>
<template>
  <template v-if="isMounted">
    <ins class="adsbygoogle" ... />
  </template>
</template>
```

---

## Kommentare mit giscus (GitHub Discussions)

giscus bettet GitHub Discussions als Kommentarfunktion ein. Besucher brauchen einen GitHub-Account. Keine Datenbank, kein Tracking, Open Source.

### Voraussetzungen

1. Repo muss **public** sein und **Discussions aktiviert** haben (Settings → Features)
2. [giscus App](https://github.com/apps/giscus) auf dem Repo installiert
3. Discussions-Kategorie erstellt (empfohlen: Typ **Announcements**, damit nur giscus/Maintainer neue Threads öffnen können)
4. IDs ermitteln:
   ```bash
   gh api graphql -f query='{ repository(owner:"OWNER", name:"REPO") { id discussionCategories(first:20) { nodes { id name } } } }'
   ```

### Installation

```bash
npm install @giscus/vue
```

### runtimeConfig (`nuxt.config.ts`)

```ts
const giscusRepoId     = process.env.GISCUS_REPO_ID     || 'R_...'
const giscusCategoryId = process.env.GISCUS_CATEGORY_ID || 'DIC_...'

runtimeConfig: {
  public: {
    giscus: {
      repo:       'owner/repo',
      repoId:     giscusRepoId,
      category:   'Blog Comments',
      categoryId: giscusCategoryId,
    },
  },
}
```

### components/GiscusComments.vue

```vue
<script setup lang="ts">
import Giscus from '@giscus/vue'   // NUR default export – kein { GiscusComponent }!
const { public: { giscus } } = useRuntimeConfig()
</script>

<template>
  <ClientOnly>
    <div class="mt-16 pt-8 border-t border-gray-800">
      <Giscus
        :repo="giscus.repo"
        :repo-id="giscus.repoId"
        :category="giscus.category"
        :category-id="giscus.categoryId"
        mapping="pathname"
        strict="0"
        reactions-enabled="1"
        emit-metadata="0"
        input-position="top"
        theme="transparent_dark"
        lang="de"
        loading="lazy"
      />
    </div>
  </ClientOnly>
</template>
```

**Pflicht**: `<ClientOnly>` – giscus rendert ein `<iframe>` clientseitig, kein SSR.

**Export-Falle**: `@giscus/vue` hat nur `default` export. `import { GiscusComponent } from '@giscus/vue'` → Build-Fehler `"GiscusComponent" is not exported`.

### CI-Vars setzen

```bash
gh variable set GISCUS_REPO_ID     --body "R_..." --repo owner/repo
gh variable set GISCUS_CATEGORY_ID --body "DIC_..." --repo owner/repo
```

Im `generate`-Step des Workflows ergänzen:
```yaml
env:
  GISCUS_REPO_ID:     ${{ vars.GISCUS_REPO_ID }}
  GISCUS_CATEGORY_ID: ${{ vars.GISCUS_CATEGORY_ID }}
```

---

## CI-Fehler debuggen

```bash
# Schnellster Weg zu Fehlerdetails:
gh run view <run-id> --log-failed

# Ganzer Log mit grep:
gh run view <run-id> --log | grep -E "(error|Error|ERROR)" | head -30
```

---

## Renovate-Grundkonfiguration

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "dependencyDashboard": true,
  "packageRules": [
    {
      "matchPackageNames": ["tailwindcss"],
      "allowedVersions": "^3.0.0"
    }
  ]
}
```

---

## Checklist: Neues Nuxt-Blog-Projekt

- [ ] `nuxt.config.ts`: `ssr: true`, `nitro.preset: 'static'`
- [ ] `content.config.ts` mit Collections + Zod-Schemas
- [ ] `better-sqlite3` in `devDependencies`
- [ ] `tailwindcss: "^3.4.17"` (nicht v4)
- [ ] `renovate.json` mit tailwindcss v3-Constraint
- [ ] GitHub Actions Workflow mit `actions/deploy-pages`
- [ ] Favicon: `{ rel: 'icon', type: 'image/svg+xml', href: '/favicon.svg' }` (kein .ico)
- [ ] `nuxt-gtag` mit `initMode: 'manual'` (DSGVO)
- [ ] `ConsentBanner.vue` in `<ClientOnly>` in `app.vue`
- [ ] giscus: Discussions aktiviert, App installiert, Kategorie (Announcements-Typ) angelegt
- [ ] `GiscusComments.vue` mit `<ClientOnly>` + `import Giscus from '@giscus/vue'` (default, kein named export)
- [ ] `GISCUS_REPO_ID` + `GISCUS_CATEGORY_ID` als GitHub Vars + im Workflow-`env:`
- [ ] Datenschutz: GA4-Abschnitt + Widerruf-Button
- [ ] Cookie-abhängige Komponenten in `<ClientOnly>`
- [ ] Dynamische Datums-/Zeit-Werte mit `onMounted`-Pattern
- [ ] Ad-Komponenten mit `isMounted`-Guard

## Checklist: Bestehende Seite warten

- [ ] CI-Fehler → `gh run view --log-failed` analysieren
- [ ] `queryContent` Calls nach v3-Migration prüfen
- [ ] Hydration-Warnings in Browser-DevTools → `ConsentBanner` / Cookie-Komponenten in `<ClientOnly>`
- [ ] `no such column` Fehler → Frontmatter-Feld in `content.config.ts` Zod-Schema ergänzen
- [ ] `tailwindcss` Version nach Renovate-PR prüfen (v3 halten)
