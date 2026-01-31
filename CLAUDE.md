# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Flux CD v2 GitOps repository managing a Kubernetes dev cluster. Uses Flux Operator (FluxInstance API), Kustomize overlays, Helm releases, and SOPS/Age encryption for secrets.

## Architecture

**Reconciliation chain** (each step depends on the previous):
1. `dev-infrastructure-namespaces` — creates namespaces (`infrastructure/overlays/dev-namespaces`)
2. `dev-infrastructure-operators` — deploys Flux Operator + RabbitMQ Operator (`infrastructure/overlays/dev-operators`)
3. `dev-infrastructure` — deploys all infra services (`infrastructure/overlays/dev`)
4. `dev-apps` — deploys applications (`apps/overlays/dev`), depends on `dev-infrastructure`

Entry point: `clusters/dev/infrastructure.yaml` and `clusters/dev/apps.yaml` define Flux Kustomizations that point to overlay paths.

**Base/Overlay pattern:**
- `infrastructure/base/<component>/` — reusable HelmRelease + OCIRepository/HelmRepository definitions
- `infrastructure/overlays/dev/<component>/` — environment-specific values, secrets, patches, and extra resources (HTTPRoutes, etc.)
- Overlays reference bases via `resources: [../../../base/<component>]` in kustomization.yaml

**HelmRelease customization pattern:**
- Base defines the HelmRelease with chart reference and default values
- Overlay injects `valuesFrom` via JSON Patch (`patches/helmrelease.yaml`) pointing to:
  - `ConfigMap` generated from `values.yaml` (public config)
  - `Secret` generated from `secrets.values.yaml` (SOPS-encrypted)
- Generator options always set `disableNameSuffixHash: true` and label `reconcile.fluxcd.io/watch: Enabled`

## Secrets

- Encrypted with SOPS using Age key `age1l535lzqsjcgzlpt4wy7galk8hhx8zw0wezqh46xaxp3n3fuakymsdnq80c`
- Files matching `secrets.values.yaml` or `secret.yaml` in overlays are auto-encrypted per `.sops.yaml` rules
- Flux decrypts using secret `nix-kubernetes-gitops-dev-sops` (replicated across namespaces via Ember Reflector)
- Encrypt: `sops --encrypt --in-place <file>`
- Decrypt for editing: `sops --decrypt --in-place <file>` or `sops <file>` (opens in editor)

## Key Conventions

- Gateway API (not Ingress) for routing — services exposed via HTTPRoute resources referencing the shared `envoy-gateway` Gateway
- Domain pattern: `*-dev.nix-fit.ru` for dev services, `*-infra.nix-fit.ru` for infra services
- TLS terminated at Gateway level via cert-manager ClusterIssuer `letsencrypt-prod` (HTTP-01 challenge)
- RBAC: `nix-admin` (full access) and `nix-developer` (read-only + pod delete) ClusterRoles defined in `infrastructure/overlays/dev/rbac/`
- Flux sync interval: 5m across all Kustomizations and HelmReleases

## Adding a New Infrastructure Component

1. Create base in `infrastructure/base/<name>/` with `kustomization.yaml`, `helmrelease.yaml`, `ocirepository.yaml`
2. Create overlay in `infrastructure/overlays/dev/<name>/` with `kustomization.yaml`, `values.yaml`, `patches/helmrelease.yaml`
3. If secrets needed: create `secrets.values.yaml`, encrypt with `sops --encrypt --in-place secrets.values.yaml`
4. Add namespace to `infrastructure/overlays/dev-namespaces/namespaces/namespace.yaml`
5. Add overlay path to `infrastructure/overlays/dev/kustomization.yaml` resources list
6. If component needs an HTTPRoute: create it in the overlay directory and add to its kustomization resources
