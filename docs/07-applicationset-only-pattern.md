# ApplicationSet-Only Pattern

## What changed

Replaced the App of Apps pattern (`bootstrap/` + `clusters/`) with an `applicationsets/` directory
organised by env, containing only `ApplicationSet` resources.

**Before:**
```
bootstrap/eu-dev.yaml          ← root Application (App of Apps)
  └── clusters/eu-dev/
        ├── dmb-namespace.yaml     ← Application (created by root app)
        ├── api-gateway.yaml       ← Application (created by root app)
        ├── oauth2-proxy.yaml      ← Application (created by root app)
        └── dmb-services.yaml      ← ApplicationSet (created by root app)
```

**After:**
```
applicationsets/
  eu-dev/
    dmb-namespace.yaml    ← ApplicationSet, applied to eu-dev ArgoCD
    api-gateway.yaml
    oauth2-proxy.yaml
    dmb-services.yaml     ← git files generator → self-serve for new services
  eu-staging/
    (same 4 files)
  eu-prod/
    (same 4 files)
```

## Why

The platform team does not support the App of Apps pattern. They do not allow the DMB team to
create `Application` objects in the `argocd` namespace autonomously. They do support
`ApplicationSet` resources, which they apply directly.

Each cluster has its own ArgoCD instance. Organising by env subfolder makes it unambiguous
which files belong to which cluster — there is no cross-cluster assumption baked into the YAMLs.
All files use `server: https://kubernetes.default.svc` (the cluster the ArgoCD runs on).

## How to onboard (platform team, per cluster)

```bash
# eu-dev cluster
kubectl apply -f applicationsets/eu-dev/

# eu-staging cluster
kubectl apply -f applicationsets/eu-staging/

# eu-prod cluster
kubectl apply -f applicationsets/eu-prod/
```

After that, the DMB team is self-serve:

- **New service:** add a YAML to `teams/dmb/{env}/services/` — the git files generator in
  `dmb-services.yaml` picks it up automatically, no platform team involvement.
- **Config change to namespace/api-gateway/oauth2-proxy:** edit the values file in `teams/dmb/{env}/`
  — ArgoCD syncs automatically.
- **oauth2-proxy chart promotion:** `promote-chart.yml` updates `chartRevision` in
  `applicationsets/{env}/oauth2-proxy.yaml` — no platform team involvement.

Adding a new file to the `applicationsets/{env}/` folder does **not** auto-deploy it.
The platform team must explicitly `kubectl apply` it. This is intentional — they retain
control over what ApplicationSets exist in the `argocd` namespace.

## Control boundary

```
Platform team controls:           DMB team controls:
────────────────────────          ─────────────────────────────────────
Which ApplicationSets exist   →   What deploys inside each one
(kubectl apply, one-time)         (push to git, any time)
```

## oauth2-proxy per-env chart pinning

Each env's `oauth2-proxy.yaml` has a `chartRevision` field in its list generator element.
`promote-chart.yml` patches this value when promoting a chart version across envs.

| env        | chartRevision                  | meaning                        |
|------------|-------------------------------|--------------------------------|
| eu-dev     | `HEAD`                        | canary — always latest         |
| eu-staging | `chart-oauth2-proxy-10.1.4`   | promoted by CI after dev soak  |
| eu-prod    | `chart-oauth2-proxy-10.1.4`   | promoted by CI after staging   |

## When would you need the platform team again?

Only when adding a **new category** of workload that needs a new ApplicationSet (e.g., a new
shared infrastructure component). For new services — the most common case — the git files
generator handles it automatically.

## Design note: was the ApplicationSet restructure necessary?

Short answer: **not strictly, but it's the cleaner long-term position.**

The original `clusters/eu-dev/` contained 3 plain `Application` objects + 1 `ApplicationSet`.
The platform team could have applied that folder directly:

```bash
kubectl apply -f clusters/eu-dev/
```

That would have created all 4 resources without any root app involved. The only actual problem
was `bootstrap/` — the root Application that was auto-creating the child objects (App of Apps).
Simply deleting `bootstrap/` and telling the platform team to apply `clusters/eu-dev/` directly
would have resolved the App of Apps concern with minimal change.

We went further and converted everything to `ApplicationSet` because:

- The platform team's stated preference is ApplicationSets over plain Applications.
- It removes ambiguity about whether a file is "meant to be applied directly" or
  "meant to be created by a parent app" — everything in `applicationsets/` is always applied directly.
- It makes the control boundary explicit in the resource kind, not just in folder naming convention.

If the platform team has no objection to plain `Application` resources (only to App of Apps),
the `clusters/` structure with `bootstrap/` removed would have been equally valid.
