# nix-kubernetes-gitops

GitOps репозиторий для Flux v2. Управляет деплоем приложений и инфраструктуры.

## Структура

```
├── apps/
│   ├── base/                        # Базовые HelmRelease + OCIRepository
│   ├── overlays/{cluster}/services/ # Кластер-специфичные патчи, values, secrets
│   └── image-automation/            # ImageRepository, ImagePolicy, ImageUpdateAutomation
├── clusters/{cluster}/              # Flux Kustomization точки входа для каждого кластера
│   ├── apps.yaml
│   ├── infrastructure.yaml
│   └── flux-system/fluxinstance.yaml
└── infrastructure/
    ├── base/                        # Базовые HelmRelease + OCIRepository
    ├── overlays/{cluster}/
    │   ├── namespaces/              # Namespaces + ServiceAccount
    │   ├── rbac/                    # ClusterRole / ClusterRoleBinding
    │   ├── crds-and-operators/      # HelmRelease операторов и их CRDs
    │   ├── operator-configs/        # Конфигурация операторов (ClusterIssuer, Gateway и т.п.)
    │   └── services/                # HelmRelease сервисов
    └── image-automation/            # ImageRepository, ImagePolicy, ImageUpdateAutomation
```

## Кластеры

Каждый кластер описан в `clusters/{cluster}/`. Flux синхронизируется с веткой `main`.

Порядок reconcile:

```
namespaces
├── rbac
└── crds-and-operators
    ├── infrastructure-image-update-automation
    └── operator-configs
        └── infrastructure
            ├── apps
            └── apps-image-update-automation
```

## Image Automation

Flux автоматически обновляет теги в манифестах через `ImageUpdateAutomation` с Setters стратегией — коммиты делает `fluxcdbot`.

Маркеры в манифестах (обновляются автоматически):
```yaml
tag: "1.0.0000" # {"$imagepolicy": "flux-system:{app}-chart-release:tag"}
```

### Стратегия версионирования (прикладные приложения)

Все чарты и образы хранятся в одном реестре. Разделение на релизы и снапшоты — только через теги:

| Ветка | Docker тег | Helm теги | Описание |
|-------|-----------|-----------|----------|
| `main` | `{version}` | `{version}` | Релиз — обновляет GitOps через ImageUpdateAutomation |
| feature | `{version}-snapshot` | `{version}-snapshot` + `:snapshot` | Снапшот — деплоится напрямую через `flux reconcile` |

**Snapshot деплой (фича ветки):**

CI пушит Helm чарт с тегом `{version}-snapshot` и дополнительно копирует его в мутабельный тег `:snapshot` через `skopeo copy`. OCIRepository на dev кластере жёстко указывает на тег `snapshot` — без ImagePolicy. Деплой инициируется через `flux reconcile helmrelease --with-source`, который форсирует перечитывание digest текущего тега.

```yaml
# apps/overlays/dev/services/{app}/patches/ocirepository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: {app}-chart-repo
spec:
  ref:
    tag: "snapshot"
```

**Release деплой (main ветка):**

ImageUpdateAutomation обновляет тег в GitOps репозитории через `{app}-chart-release` ImagePolicy с диапазоном `>=0.0.1000 <1.0.0000`. Диапазон без суффикса `-0` автоматически исключает pre-release теги (вида `1.0.0-snapshot`) и статичный тег `snapshot`.

```yaml
tag: "1.0.0000" # {"$imagepolicy": "flux-system:{app}-chart-release:tag"}
```

## Secrets

Секреты зашифрованы через [SOPS](https://github.com/getsops/sops) с Age ключом. Все `secrets.values.yaml` и `secret.yaml` файлы в overlays зашифрованы.

Flux расшифровывает при применении через `.sops.yaml` в корне репы.
