## 📄 **Vault Version Publisher – Dokumentacja techniczna**


### Sekcja `spec.inputs`

Definiuje parametry wejściowe dla szablonu.

| Parametr                       | Domyślna wartość                                            | Opis                                               |
| ------------------------------ | ----------------------------------------------------------- | -------------------------------------------------- |
| `docker_image`                 | `registry.gitlab.com/pl.rachuna-net/containers/vault:1.0.0` | Obraz Dockera z klientem Vault                     |
| `debug`                        | `"false"`                                                   | Flaga debugowania                                  |
| `publish_mount_path`           | `"kv-gitlab"`                                               | Mount point (path) w Vault (np. KV v2)             |
| `publish_vault_secret_path`    | `""`                                                        | Ścieżka do secreta w Vault, w formacie `path:key`  |
| `publish_value_variables_name` | `"1.0.0"`                                                   | Nazwa zmiennej, której wartość ma być opublikowana |

---

### Sekcja `variables`

Ustawia zmienne środowiskowe używane w szablonie na podstawie wartości `spec.inputs`.

| Zmienna                              | Źródło                                |
| ------------------------------------ | ------------------------------------- |
| `CONTAINER_IMAGE_VAULT`              | `inputs.docker_image`                 |
| `PUBLISH_MOUNT_PATH`                 | `inputs.publish_mount_path`           |
| `PUBLISH_VAULT_SECRET_PATH`          | `inputs.publish_vault_secret_path`    |
| `PUBLISH_VAULT_VALUE_VARIABLES_NAME` | `inputs.publish_value_variables_name` |
| `DEBUG`                              | `inputs.debug`                        |

---

### 🔁 `extends: .vault_publish_version:base`

### Opis działania

Ten blok jest głównym szablonem wykonawczym. Uruchamia skrypt w kontenerze Vaulta.

### Wewnątrz `script`

```bash
IFS=":" read -r path secret <<< "$PUBLISH_VAULT_SECRET_PATH"
vault kv patch -mount=$PUBLISH_MOUNT_PATH $path $secret="${!PUBLISH_VAULT_VALUE_VARIABLES_NAME}"
```

– rozdziela `pl.rachuna-net:GITLAB-CI` na:

* `path = pl.rachuna-net`
* `secret = GITLAB-CI`

– wykonuje publikację (`vault kv patch`):

> [!warning]
> `${!PUBLISH_VAULT_VALUE_VARIABLES_NAME}` = **nie wartość literalna**, tylko **wartość zmiennej o nazwie podanej jako string**

---

### 🔧 Wymagania

* Vault mount typu `kv` (v2) pod `kv-gitlab`
* Token Vaulta (przez zmienną `VAULT_TOKEN`) z policy zawierającą:

```hcl
path "sys/internal/ui/mounts/*" {
  capabilities = ["read"]
}
path "kv-gitlab/data/*" {
  capabilities = ["create", "read", "update", "patch"]
}
path "kv-gitlab/metadata/*" {
  capabilities = ["read", "list"]
}
```

---

### 📦 Przykład użycia

W `.gitlab-ci.yml` projektu:

```yaml
include:
  - local: /templates/vault_publish.yml

variables:
  RELEASE_CANDIDATE_VERSION: "1.0.0-abc123"

🎉 Publish version in vault:
  extends: ['.vault_publish_version:base']
  stage: release
  variables:
    publish_vault_secret_path: "pl.rachuna-net:GITLAB-CI"
    publish_value_variables_name: "RELEASE_CANDIDATE_VERSION"
  rules:
    - when: on_success
```

---

## 🧪 Wynik działania

W Vault zostanie zapisany:

* pod: `kv-gitlab/pl.rachuna-net`
* klucz: `GITLAB-CI`
* wartość: `"1.0.0-abc123"`