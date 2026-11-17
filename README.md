# renovate-config

Gemeinsame [Renovate](https://docs.renovatebot.com/)-Presets für alle Repositories von
TorstenAmshove. Läuft über die **Mend Renovate App (Free)** auf **GitHub Free**.

## Verwendung

Jedes Repository hat nur eine kleine `.github/renovate.json5`:

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: [
    "local>TorstenAmshove/renovate-config",            // immer
    "local>TorstenAmshove/renovate-config:container",  // Repos, die ein Image nach GHCR releasen
    // "local>TorstenAmshove/renovate-config:ansible", // Ansible-Repos
  ],
  // repo-spezifische Regeln, z. B. Auto-Merge von Minor-Updates
}
```

Die Config liegt bewusst unter `.github/`: Die Release-Workflows ignorieren `.github/**`, eine
Änderung an der Renovate-Config erzeugt so kein neues Image.

| Preset | Datei | Genutzt von |
|---|---|---|
| `local>TorstenAmshove/renovate-config` | `default.json` | alle |
| `…:container` | `container.json` | birthday_reminder, file_dropin, immich_inbox_cleaner, scanner-remote, HomeAtlas |
| `…:ansible` | `ansible.json` | ansible-docker.lan, ansible-opencode.lan, ansible-cloudflared.lan, ansible-portainer-stack |

Änderungen an diesem Repo wirken beim nächsten Renovate-Lauf auf **alle** Repos.
`validate.yml` prüft die Presets mit `renovate-config-validator --strict`.

## Regeln (`default.json`)

| Update | Verhalten |
|---|---|
| **Major** | Eigener PR, Label `major`, nie Auto-Merge. Der PR (mit Assignee) löst die GitHub-Benachrichtigung aus – ein neuer Eintrag im *Dependency Dashboard* allein tut das nicht, weil GitHub bei Änderungen am Issue-Text nicht benachrichtigt. Nicht gewollte Majors: PR schließen, Renovate schlägt diese Major-Version dann nicht erneut vor. |
| **Minor** | npm und pip: ein gruppierter PR pro Woche (montags). Auto-Merge nur, wenn das Repo es selbst einschaltet (ausreichende Tests). Andere Ökosysteme: einzelne PRs, manuell. |
| **Patch, Digest** | Auto-Merge, sobald alle Checks grün sind. |
| **Pin** (exakte Version / Digest ergänzen) | Immer Auto-Merge, ändert zur Laufzeit nichts. Repos mit Ausschlusslisten müssen Pin danach wieder erlauben (siehe ansible-docker.lan). |
| **GitHub Actions** | Monatlich (am 1.), ein Sammel-PR für alles außer Majors (Auto-Merge) und einer für Majors (manuell), 7 Tage Mindestalter, Commit-Typ `ci`. Actions bleiben auf Commit-SHA gepinnt (`helpers:pinGitHubActionDigests`). |
| **Lockfile-Maintenance** | Monatlich (am 1.), Auto-Merge, Commit-Typ `fix` (aktualisiert transitive Abhängigkeiten, die ins Image gehen). |
| **Sicherheitslücken** | Sofort, ohne Mindestalter und Zeitplan, Label `security` (GitHub-Dependabot-Alerts + OSV). |

Feste Versionen überall:

- **Docker-Images**: exakte Version + Digest (`node:26.13.1-alpine@sha256:…`, `docker:pinDigests`).
  Versions-Updates kommen als lesbare PRs (`26.13.1 → 26.14.0`), Neubauten desselben Tags
  (Sicherheitsfixes im Basissystem) als Digest-PRs. Kein `latest`, `lts`, `stable`: Bei solchen
  Tags sieht Renovate nur Digests, ein Major-Sprung wäre ein unsichtbarer, automatisch gemergter
  Digest-Wechsel.
- **Node**: Renovates `node`-Versionierung zählt ungerade und noch nicht LTS gewordene Majors als
  instabil – es kommen nur LTS-Majors, als eigener PR.
- **nginx**: nur Stable-Versionen (gerade Minor-Version), Mainline (ungerade) wird ignoriert.
- **Laufzeit-Versionen gemeinsam**: `node`/`python` in Dockerfile und in `setup-node`/
  `setup-python` (Workflows) bilden je eine Gruppe „node runtime“/„python runtime“.
- **Runner**: `runs-on: ubuntu-26.04` statt `ubuntu-latest`; ein neues Runner-Image ist ein Major.
- **npm/pip-Ranges** werden auf exakte Versionen gepinnt (`:pinAllExceptPeerDependencies`).
- **Annotierte Pins** (Regex-Manager für alle `*.yml`/`*.yaml` und Dockerfiles):

  ```yaml
  # renovate: datasource=npm depName=@mgcrea/mcp-unifi-network
  unifi_mcp_version: "0.5.1"
  ```
  ```dockerfile
  # renovate: datasource=github-tags depName=agl/jbig2enc
  ARG JBIG2ENC_VERSION=0.29
  ```
- **npx in Workflows**: `npx --package renovate@44.248.1 -- …` wird ebenfalls aktualisiert.

Allgemein:

- **Mindestalter** 3 Tage, npm 7 Tage (Schutz vor kompromittierten Releases, die meist binnen
  Tagen zurückgezogen werden). `timestamp-optional`, weil ghcr.io keine Release-Zeitpunkte liefert.
- **Auto-Merge macht Renovate selbst** (`platformAutomerge: false`): Auf GitHub Free gibt es für
  private Repos weder Branch-Schutz noch natives Auto-Merge. Renovate mergt nur bei grünen Checks.
- **Merge per PR** (`automergeType: "pr"`): Die Release-Workflows hängen an
  `pull_request: closed`. Ein Merge über das Token der Renovate-App löst sie aus
  (`automergeType: "branch"` würde keinen Release erzeugen).
- **Kein Zeitfenster** für Auto-Merge: Merges passieren rund um die Uhr.
- **Keine engen Zeitpläne**: Die Free-Version der Mend-App läuft nicht in festen kurzen Abständen,
  Fenster von wenigen Stunden werden leicht verpasst. Zeitpläne umfassen daher ganze Tage.
- Commit-Messages: Conventional Commits (`fix(deps)` für Laufzeit-Abhängigkeiten,
  `chore(deps)` für devDependencies, `ci` für Actions und Lint-Tooling).

## `container.json`

- Base-Image-Digest-Updates: Auto-Merge ohne Mindestalter. Build und Smoke-Test des PRs sind das
  Gate.
- Commit-Typ `fix` → Patch-Release → `ansible-docker.lan` übernimmt das neue Image-Tag (Patch,
  Auto-Merge) → Portainer deployt. Sicherheitsupdates des Base-Images landen damit ohne Handgriff
  in Produktion.

## `ansible.json`

- Regex-Manager für `pip install "paket==x.y.z"` in `.github/workflows/*.yml`.
- **Lint tooling**: `ansible-core`, `ansible-lint`, `yamllint` und die Python-Version des
  Lint-Jobs (`setup-python`) immer gemeinsam in einem PR, inkl. Major, Auto-Merge. Betrifft nur CI.

## Manuelle GitHub-Einstellungen je Repo

- *Settings → Code security*: **Dependabot alerts an**, **Dependabot security updates aus**
  (Renovate übernimmt die PRs, sonst gibt es doppelte).
- *Settings → General*: **Automatically delete head branches an**.
