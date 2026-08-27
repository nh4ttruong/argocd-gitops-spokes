# ArgoCD GitOps — Spokes

Workload repo for the spoke clusters. Nothing here is applied by hand. The
ApplicationSets in the [hub repo](https://github.com/nh4ttruong/argocd-gitops-hub)
(`appsets/<env>/workloads/`) discover these folders by convention, per
environment: `dev` · `staging` · `production`.

An app exists in an environment if and only if its `app.yaml` for that
environment exists. Delete `workloads/backend/charts/staging/app.yaml` and
`staging-backend` is pruned.

## Apps

| App | Engine | Namespace | Environments |
| --- | --- | --- | --- |
| `backend` | Helm — podinfo wrapper | `shop` | dev · staging · production |
| `homepage` | Helm — podinfo wrapper | `shop` | dev · staging · production |
| `webhook` | Helm — podinfo wrapper | `shop` | dev · staging · production |
| `whoami` | Kustomize — base + overlays | `whoami` | dev · staging · production |

`backend`, `homepage` and `webhook` are one product sharing the `shop`
namespace, declared per env in their `app.yaml`.

## Layout

Helm app:

```
workloads/backend/
├── charts/<env>/Chart.yaml     # Wrapper chart — Dependency on the real chart
├── charts/<env>/app.yaml       # namespace: shop
└── values/
    ├── common.yaml             # Merged first
    ├── <env>/values.yaml       # Then this
    └── <env>/version.yaml      # Then this — Image tag, the promote unit
```

Kustomize app:

```
workloads/whoami/
├── base/                       # Deployment + Service
└── envs/<env>/
    ├── kustomization.yaml      # Overlay — Replicas, labels, images[].newTag
    └── app.yaml                # namespace: whoami
```

## Conventions

| Concern | Convention |
| --- | --- |
| App manifest | `app.yaml` beside the chart or overlay — Its presence enables the app in that env, and its `namespace` key becomes the Application destination |
| Generated Application | `{env}-{app}` · Project `{env}-apps` · Namespace from `app.yaml` |
| Promote a version | Helm → Image tag in `values/<env>/version.yaml` · Kustomize → `images[].newTag` in the overlay |
| Sync | Automated for dev and staging · Production waits for a manual sync |

## Add an app

1. Copy `workloads/backend/` (Helm) or `workloads/whoami/` (Kustomize) to `workloads/<name>/`.
2. Point the chart dependency, or the kustomize base, at the real service.
3. Keep only the `charts/<env>/` or `envs/<env>/` folders for the environments it should run in.
4. Set `namespace` in each env's `app.yaml` — Reuse an existing one to co-locate the services of one product.

Merge to `main` and the hub picks it up on the next generator poll, roughly
three minutes.

> [!NOTE]
> Demo workloads. The Helm apps wrap [podinfo](https://github.com/stefanprodan/podinfo)
> and the kustomize app runs `traefik/whoami`. Point them at real charts and
> images for actual use.
