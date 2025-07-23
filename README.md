# <img src=".gitlab/avatar.png" alt="avatar" height="20"/> release

[![](https://gitlab.com/pl.rachuna-net/cicd/components/release/-/badges/release.svg)](https://gitlab.com/pl.rachuna-net/cicd/components/release/-/releases)
[![](https://gitlab.com/pl.rachuna-net/cicd/components/release/badges/main/pipeline.svg)](https://gitlab.com/pl.rachuna-net/cicd/components/release/-/commits/main)

Komponent do zarządzania wersjami i publikacji bibliotek i aplikacji na środowiska produkcyjne


---
## Vault


---
## Versioning

**Wymagania**:

- Obraz kontenera dostępny jest [tutaj](https://gitlab.com/pl.rachuna-net/containers/semantic-release).
- Stosowanie standardu opisywania commitów [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)

---
### Jak działa komponent?

#### Analiza brancha

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
#### Sprawdzanie convensional commits

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
#### Generowanie releases notes

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
#### Generowanie chandelog.md

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
#### Integracja z gitlab

Integracja z `gitab`

```json
[
    "@semantic-release/gitlab"
]
```

---
#### Ustawienie zmiennych środowiskowych

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

Opublikowanie zmienny środowiskowych w `.gitlab-ci.yml`

```yaml
versioning:base:
  (...)
  artifacts:
    reports:
      dotenv: versioning.env
```

---
### Przykładowe użycie w `.gitlab-ci.yml`

```yml
default:
  tags:
    - onprem

include:
  - component: $CI_SERVER_FQDN/pl.rachuna-net/cicd/components/release/semantic-release@main

stages:
  - prepare
  - publish

# Ustawia wersje, ale proces nie zakłada jeszcze obiektów w gitlab
🕵 Set Version:
  stage: prepare
  variables:
    VERSIONING_DRY_RUN: "true"
  extends: ['.versioning:base']

📍 Publish Version:
  stage: publish
  extends: ['.versioning:base']
```

---
## Inputs

| Input                      | Env                                | Opis                                    | Wartość domyślna                                                       |
|----------------------------|------------------------------------|-----------------------------------------|------------------------------------------------------------------------|
| `docker_image`             | `CONTAINER_IMAGE_SEMANTIC_RELEASE` | Obraz z `semantic-release`              | `registry.gitlab.com/pl.rachuna-net/containers/semantic-release:2.0.0` |
| `debug`                    | `VERSIONING_DEBUG`                 | Tryb debug                              | `false`                                                                |
| `component_repo_namespace` | `COMPONENT_REPO_NAMESPACE`         | namespace komponentu release(full_path) | `pl.rachuna-net/cicd/components/release`                               |
| `component_repo_branch`    | `COMPONENT_REPO_BRANCH`            | nazwa brancha komponentu release        | `main`                                                                 | 
| `dry_run`                  | `VERSIONING_DRY_RUN`               | Tryb `dry run`                          | `false`                                                                |

---
### Outputs

| Env                        | Wartość domyślna     | Opis                                    |
|----------------------------|----------------------|-----------------------------------------|
| `RELEASE_CANDIDATE_VERSION`| `${CI_COMMIT_TAG}`   | Nadany numer wersji                     |
| `ARTIFACTS_TYPE`           |                      | Typ artefaktu `snapshot` lub `release`  |
| `JIRA_ISSUES_IDS`          |                      | Lista zgłoszeń z JIRA                   |

---
### Linki

- [SemVer](https://semver.org/)
- [semantic-release](https://semantic-release.gitbook.io/semantic-release)


---
## Contributions
Jeśli masz pomysły na ulepszenia, zgłoś problemy, rozwidl repozytorium lub utwórz Merge Request. Wszystkie wkłady są mile widziane!
[Contributions](CONTRIBUTING.md)

---
## License
Projekt licencjonowany jest na warunkach [Licencji MIT](LICENSE).

---
# Author Information
### &emsp; Maciej Rachuna
# <img src="https://gitlab.com/pl.rachuna-net/gitlab-profile/-/raw/main/assets/logo/website_logo_transparent_background.png" alt="rachuna-net.pl" height="100"/>

