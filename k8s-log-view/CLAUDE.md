# CLAUDE.md

This file guides Claude Code when working in this repository.

## Overview

This is a **Helm chart** for `k8s-log-view`, a read-only web UI for browsing
Kubernetes namespaces, pods, events and pod logs. The chart was scaffolded with
`helm create` and then adapted to match a hand-written set of manifests
(previously in `k8s.yaml`).

The application is a Next.js app that:
- Listens on container port **3000** (`http`), exposed via a `ClusterIP` Service on port **80**.
- Runs as a non-root user with a read-only root filesystem.
- Needs cluster-wide **read-only RBAC** to `get`/`list` namespaces, pods, events and to `get` pod logs.

## Layout

```
k8s-log-view/
├── Chart.yaml                 # chart metadata (version 0.1.0, appVersion "v1.0.0")
├── values.yaml                # all configurable values (defaults below)
├── CLAUDE.md                  # this file
└── templates/
    ├── _helpers.tpl           # name/label/serviceAccountName helpers
    ├── deployment.yaml        # Deployment (env, probes, securityContext, volumes)
    ├── service.yaml           # ClusterIP Service (port 80 -> http/3000)
    ├── serviceaccount.yaml    # ServiceAccount (gated by serviceAccount.create)
    ├── clusterrole.yaml       # ClusterRole + ClusterRoleBinding (gated by rbac.create)
    ├── ingress.yaml           # Ingress (disabled by default)
    ├── httproute.yaml         # Gateway API HTTPRoute (disabled by default)
    ├── hpa.yaml               # HorizontalPodAutoscaler (disabled by default)
    ├── NOTES.txt              # post-install usage notes
    └── tests/
        └── test-connection.yaml
```

## Key values (`values.yaml`)

| Value | Default | Notes |
|-------|---------|-------|
| `replicaCount` | `1` | Ignored when `autoscaling.enabled`. |
| `image.repository` | `dennissobczak/k8s-log-view` | |
| `image.tag` | `"0.1.0"` | Pinned explicitly; do **not** rely on `appVersion` fallback (appVersion is `v1.0.0`). |
| `image.pullPolicy` | `IfNotPresent` | |
| `containerPort` | `3000` | The port the app listens on; the `http` named port. |
| `service.type` / `service.port` | `ClusterIP` / `80` | Service targets the named `http` port. |
| `env` | `NODE_ENV=production`, `NEXT_TELEMETRY_DISABLED="1"` | List of `name`/`value` pairs. |
| `serviceAccount.create` | `true` | |
| `rbac.create` | `true` | Creates ClusterRole + ClusterRoleBinding. |
| `rbac.rules` | read-only on namespaces/pods/events + pods/log | Edit here to change RBAC scope. |
| `podSecurityContext` | `runAsNonRoot: true`, `runAsUser: 1000`, `runAsGroup: 1000` | |
| `securityContext` | `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]` | |
| `resources` | requests `100m`/`128Mi`, limits `500m`/`512Mi` | |
| `livenessProbe` | `GET /` on `http`, delay `15s`, period `30s` | |
| `readinessProbe` | `GET /` on `http`, delay `5s`, period `10s` | |
| `volumes` / `volumeMounts` | `tmp` -> `/tmp`, `next-cache` -> `/app/.next/cache` (both `emptyDir`) | Required because the root FS is read-only. |
| `ingress.enabled` | `false` | |
| `httpRoute.enabled` | `false` | Gateway API. |
| `autoscaling.enabled` | `false` | |

## Naming

All resource names come from the `k8s-log-view.fullname` helper, which is
`<release>-<chart>` unless the release name already contains the chart name.
**Install the release as `k8s-log-view`** so resource names match the original
manifests exactly:

```sh
helm install k8s-log-view ./k8s-log-view \
  --namespace k8s-log-view --create-namespace
```

> The original `k8s.yaml` created the `k8s-log-view` Namespace itself. The chart
> does **not** template a Namespace — use `--create-namespace` (Helm best
> practice) so the release namespace is managed by Helm.

## Conventions / gotchas

- **Container port vs service port are different** (`3000` vs `80`). Use
  `.Values.containerPort` for the container and the named `http` port for the
  Service `targetPort`. Don't conflate them with `service.port`.
- The read-only root filesystem means any new write path needs a matching
  `emptyDir` entry in both `volumes` and `volumeMounts`.
- RBAC objects are **cluster-scoped**; the binding's subject namespace is
  `.Release.Namespace`.

## Validate changes

```sh
helm lint ./k8s-log-view
helm template k8s-log-view ./k8s-log-view --namespace k8s-log-view
```
