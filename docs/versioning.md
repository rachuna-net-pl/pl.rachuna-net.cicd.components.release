# 🛠 Job template: 📍 Publish Version

Job, który odpowiada za wersjonowanie i publikację zmian w repozytorium za pomocą narzędzia **semantic-release**.
Szablon składa się z dwóch kroków:

1. **🕵 Set Version** – ustawia wersję w trybie dry-run (przygotowanie).
2. **📍 Publish Version** – publikuje nową wersję (release).

> [!NOTE] 
>
> **Wymagania**:
>
> - Obraz kontenera dostępny jest [tutaj](https://gitlab.com/pl.rachuna-net/containers/semantic-release).
> - Stosowanie standardu opisywania commitów [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)


---
## ⚙️ Parametry wejściowe (`inputs`)

| Nazwa                      | Typ    | Domyślna wartość                                                                            | Opis                                                       |
| -------------------------- | ------ | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `docker_image`             | string | `registry.gitlab.com/pl.rachuna-net/containers/semantic-release:2.0.0`                      | Obraz Dockera zawierający **semantic-release**.            |
| `logo_url`                 | string | `https://gitlab.com/pl.rachuna-net/cicd/gitlab-ci/-/raw/main/_configs/_logo?ref_type=heads` | URL logotypu wyświetlanego w logach joba.                  |
| `debug`                    | string | `false`                                                                                     | Włącza tryb debugowania (`--debug`).                       |
| `dry_run`                  | string | `false`                                                                                     | Uruchamia semantic-release w trybie dry-run (`--dry-run`). |
| `component_repo_namespace` | string | `pl.rachuna-net/cicd/components/release`                                                    | Namespace i ścieżka do repozytorium komponentu.            |
| `component_repo_branch`    | string | `main`                                                                                      | Gałąź repozytorium komponentu.                             |
| `releaserc_path`           | string | `_configs/release/.releaserc.js`                                                            | Ścieżka do pliku konfiguracyjnego **semantic-release**.    |

---
## 🧬 Zmienne środowiskowe obsługiwane przez skrypt

Job ustawia i wykorzystuje poniższe zmienne środowiskowe:

* `CONTAINER_IMAGE_SEMANTIC_RELEASE` – obraz Dockera z semantic-release.
* `LOGO_URL` – adres logotypu.
* `DEBUG` – flaga debugowania (dziedziczona z `inputs.debug`).
* `VERSIONING_DEBUG` – włącza dodatkowy debug semantic-release.
* `VERSIONING_DRY_RUN` – uruchamia semantic-release w trybie dry-run.
* `COMPONENT_REPO_NAMESPACE` – namespace repozytorium komponentu.
* `COMPONENT_REPO_BRANCH` – gałąź repozytorium komponentu.
* `RELEASERC_FILE` – ścieżka do pliku `.releaserc.js`.

---
## 📤 Output

Job generuje:

1. **Plik `CHANGELOG.md`** – zawierający opis zmian.
2. **Plik `versioning.env`** (zapisany jako artifact `dotenv`) z informacjami o wersji, np.:

   ```
   VERSION=1.2.3
   VERSION_MAJOR=1
   VERSION_MINOR=2
   VERSION_PATCH=3
   ```
3. Logi semantic-release z informacją o wygenerowanej wersji i ewentualnie o publikacji.

---
## 📝 Przykładowe użycie w pipeline

```yaml
include:
  - component: '$CI_SERVER_FQDN/pl.rachuna-net/cicd/components/release@$COMPONENT_VERSION_RELEASE'
    inputs:
      docker_image: "registry.gitlab.com/pl.rachuna-net/containers/semantic-release:2.0.0"
      debug: "true"
      dry_run: "false"
```

---
## 🔎 Kroki joba

### **🕵 Set Version**

* Stage: `prepare`
* Uruchamia semantic-release w trybie dry-run (`VERSIONING_DRY_RUN: "true"`) w celu przygotowania wersji i changeloga.

### **📍 Publish Version**

* Stage: `release`
* Uruchamia semantic-release w trybie pełnym (publikacja wersji i generacja changeloga).

---
## Jak działa komponent?

### Analiza brancha

W pierwszej kolejności sprawdzane jest - czy można zwersjonować aplikacje na odpowiednim branchu

```json
"branches": [
    { 
      "name": "feature/*", 
      "channel": "feat-${name.replace(/^feature\\//, '').substring(0, 20)}", 
      "prerelease": "feat-${name.replace(/^feature\\//, '').substring(0, 20)}" 
    },
    { 
      "name": "epic/*", 
      "channel": "epic-${name.replace(/^epic\\//, '').replace(/[^A-Za-z0-9]/, '').substring(0, 20)}", 
      "prerelease": "epic-${name.replace(/^epic\\//, '').replace(/[^A-Za-z0-9]/, '').substring(0, 20)}" 
    },
    { "name": "alpha", "channel": "alpha", "prerelease": true },
    { "name": "release", "channel": "rc", "prerelease": true },
    { "name": "hotfix", "channel": "ht", "prerelease": true },
    { "name": "main" }
],
```

---
### Sprawdzanie convensional commits

Komponent sprawdza, czy czyta wszystkie informacje o commitach zgodnie z standardem [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)

```json
"plugins": [
[ "@semantic-release/commit-analyzer",
    {
        "preset": "conventionalcommits",
        "releaseRules": [
            { "breaking": true, "release": "major" },
            { "type": "feat", "release": "minor" },
            { "type": "fix", "release": "patch" },
            { "type": "style", "release": "patch" },
            { "type": "perf", "release": "patch" },
            { "type": "test", "release": "patch" },
            { "type": "docs", "release": "patch" },
            { "type": "sec", "release": "patch" }
        ]
    }
],
```

---
### Generowanie releases notes

Następnie generuje release notes

```json
[
    "@semantic-release/release-notes-generator",
    {
        "preset": "conventionalcommits",
        "presetConfig": {
            "types": [
              { "type": "feat", "section": "✨ Features ✨" },
              { "type": "fix", "section": "🐛 Bug Fixes 🐛" },
              { "type": "revert", "section": "⏮️️ Reverts ⏮️️" },
              { "type": "perf", "section": "🔧 Performance Improvements 🔧" },
              { "type": "docs", "section": "📝 Documentation 📝" },
              { "type": "build", "section": "📀 Build 📀" },
              { "type": "test", "section": "💚 Tests 💚" },
              { "type": "ci", "section": "🏁 CI/CD 🏁" }
            ]
        }
    }
],
```

---
### Generowanie chandelog.md

Generowanie localnie pliku `CHANGELOG.md`

```json
[
    "@semantic-release/changelog",
    {
      "changelogFile": "CHANGELOG.md"
    }
],
```

---
### Integracja z gitlab

Integracja z `gitab`

```json
[
    "@semantic-release/gitlab"
]
```

---
### Ustawienie zmiennych środowiskowych

```js
{
  name: "versioning-variables",
  generateNotes: (pluginConfig, context) => {
    const releaseNote = context.nextRelease.notes;
    const nextVersion = context.nextRelease.version;

    // Check if the version is a final version or a snapshot
    const finalVersionRegex = /(\d+\.\d+\.\d+)/;
    const matchFinalVersion = nextVersion.match(finalVersionRegex);
    let artifactType = "snapshot";
    if (matchFinalVersion) {
      artifactType = "release";
    }

    // Extract JIRA issues from commits
    const jiraRegexp = /\b[A-Z][A-Z0-9_]+-[1-9][0-9]*/g;
    const jiraSet = new Set();

    for (const commit of context.commits) {
      if (commit.message) {
        const matchJIRA = commit.message.match(jiraRegexp);
        if (matchJIRA) {
          matchJIRA.forEach(jira => jiraSet.add(match));
        }
      }
    }

    // Create a file with the environment variables
    const uniqJiraList = Array.from(jiraSet)
    const environmentList = [
      { key: "RELEASE_CANDIDATE_VERSION", value: nextVersion },
      { key: "ARTIFACTS_TYPE", value: artifactType },
      { key: "JIRA_ISSUES_IDS", value: uniqJiraList.join(", ") }
    ]
    const fileContent = environmentList.map(env => `${env.key}=${env.value}`).join("\n");
    fs.writeFileSync("versioning.env", fileContent);
  }
}
```

---
### Linki

- [SemVer](https://semver.org/)
- [semantic-release](https://semantic-release.gitbook.io/semantic-release)
