# nix-kubernetes-gitops

```plain
├── apps
│   ├── base
│   ├── overlays
│   │   └── dev
│   │       └── patches
│   └── sources
│       └── base
│           ├── backend
│           └── frontend
├── clusters
│   └── dev
│       └── flux-system
└── infrastructure
    ├── base
    │   ├── buildkitd
    │   ├── cert-manager
    │   ├── envoy-gateway
    │   ├── headlamp
    │   └── keycloak
    └── overlays
        └── dev
            ├── buildkitd
            ├── cert-manager
            │   └── patches
            ├── envoy-gateway
            │   └── patches
            ├── headlamp
            │   └── patches
            ├── keycloak
            │   └── patches
            ├── keycloak-infra
            │   └── patches
            ├── namespaces
            ├── patches
            └── rbac
```
