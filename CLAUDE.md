# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is the **GitOps source of truth** for the Behavior-Aware Resilient Kubernetes Platform (master's thesis project). ArgoCD reconciles cluster state from this repo. Do not apply manifests imperatively — all changes go through git, ArgoCD picks them up.

## Key Commands

Validate a Kustomize overlay before pushing:
```bash
kustomize build overlays/primary | kubectl apply --dry-run=client -f -
kustomize build overlays/secondary | kubectl apply --dry-run=client -f -
```

One-time bootstrap (seeds ArgoCD with the root App-of-Apps — never re-run; ArgoCD self-manages after this):
```bash
kubectl apply -f bootstrap/root-app.yaml
```

Check ArgoCD sync status:
```bash
kubectl get applications -n argocd
kubectl get applications -n argocd -o wide   # shows sync/health reason
```

Force an immediate ArgoCD sync (instead of waiting for the poll interval):
```bash
argocd app sync <app-name>
```

## Architecture: App-of-Apps Pattern

```
bootstrap/
  root-app.yaml          ← one-time kubectl apply; points ArgoCD at bootstrap/apps/
  apps/
    platform-*.yaml      ← one Application per platform service
    apps-*.yaml          ← one Application per workload app

platform/<service>/      ← Helm values or plain manifests for each platform service
apps/
  sample-app/            ← workload app manifests
  base/                  ← shared base manifests referenced by Kustomize overlays

overlays/
  primary/               ← Kustomize overlay for the primary cluster
  secondary/             ← Kustomize overlay for the secondary (DR) cluster
```

The root Application in `bootstrap/root-app.yaml` watches `bootstrap/apps/`. Each child Application there points to its own path in `platform/` or `apps/`. This means adding a new service = drop a new Application YAML into `bootstrap/apps/` and create the corresponding `platform/<service>/` directory.

## Cluster Topology

- **primary** — production cluster; this is where all platform services currently run
- **secondary** — disaster-recovery cluster (Phase 3, not yet wired up)

Overlays in `overlays/{primary,secondary}/` use Kustomize patches to customize replica counts, resource limits, and any cluster-specific values on top of shared base manifests.

## Platform Services

Current services tracked under `platform/`:

| Directory | Chart / Tool | Notes |
|---|---|---|
| `platform/cert-manager` | cert-manager Helm chart | |
| `platform/nginx-ingress` | ingress-nginx Helm chart | |
| `platform/longhorn` | Longhorn Helm chart | stateful — adopt via annotation, never helm uninstall |
| `platform/minio` | MinIO Helm chart | stateful — same constraint |
| `platform/monitoring` | kube-prometheus-stack | **Grafana pinned to 12.0.2** — chart default 13.0.1 crashes datasource provisioning |
| `platform/loki` | Loki Helm chart | stateful — same constraint |
| `platform/alloy` | Grafana Alloy Helm chart | log collector → forwards to Loki |

## Stateful Service Adoption Rule

When migrating Longhorn, MinIO, kube-prometheus-stack, or Loki to ArgoCD management, **do not `helm uninstall` first**. Instead, add the ArgoCD adoption annotation to existing resources and let ArgoCD adopt them in-place. Uninstalling stateful services risks data loss and requires rehydration from backup.

## Secrets

Secrets are **never committed**. The `.gitignore` excludes `*.secret.yaml`, `*-secret.yaml`, and the `secrets/` directory. Use Sealed Secrets or External Secrets Operator to manage secrets declaratively without storing plaintext in the repo.

## Grafana Datasource UID

The Loki datasource UID is `P8E80F9AEF21F6940`. In dashboard JSON, reference it by variable rather than hardcoding `"loki"` — the provisioned UID is what ArgoCD will apply, and a literal name lookup will fail silently.

## Broader Project Context

This repo is **Phase 2** of a 6-phase thesis project. The other phases live in a separate Ansible-based infrastructure repo. Phases relevant to what happens here:

- **Phase 1 (Observability)**: complete — Prometheus, Grafana, Loki, Alertmanager already running on the cluster, managed by Ansible today, being migrated to ArgoCD here
- **Phase 2 (CI/CD + GitOps)**: this repo — migrate platform services to ArgoCD management
- **Phase 3 (DR)**: will activate the secondary cluster overlay
- **Phase 4 (Anomaly Detection)**: MLflow + Argo Workflows + KServe + MinIO will be added as the first "ML tenant" under `apps/`
