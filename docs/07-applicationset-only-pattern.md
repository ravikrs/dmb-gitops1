# ApplicationSet-Only Pattern

## What changed

Replaced the App of Apps pattern (`bootstrap/` + `clusters/`) with a flat `applicationsets/` directory containing only `ApplicationSet` resources.

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
  dmb-namespace.yaml             ← ApplicationSet, list generator (all 3 envs)
  api-gateway.yaml               ← ApplicationSet, list generator (all 3 envs)
  oauth2-proxy.yaml              ← ApplicationSet, list generator (all 3 envs, per-env chartRevision)
  dmb-services-eu-dev.yaml       ← ApplicationSet, git files generator
  dmb-services-eu-staging.yaml   ← ApplicationSet, git files generator
  dmb-services-eu-prod.yaml      ← ApplicationSet, git files generator
```

## Why

The platform team does not support the App of Apps pattern — they do not allow the DMB team to create `Application` objects in the `argocd` namespace autonomously. They do support `ApplicationSet` resources, which they apply directly.

App of Apps requires a root `Application` that spawns child `Application` objects in the `argocd` namespace. This is exactly what `bootstrap/` + `clusters/` was doing.

## How to onboard (platform team)

The platform team applies the 6 ApplicationSets once per cluster:

```bash
kubectl apply -f applicationsets/dmb-namespace.yaml
kubectl apply -f applicationsets/api-gateway.yaml
kubectl apply -f applicationsets/oauth2-proxy.yaml
kubectl apply -f applicationsets/dmb-services-eu-dev.yaml
kubectl apply -f applicationsets/dmb-services-eu-staging.yaml
kubectl apply -f applicationsets/dmb-services-eu-prod.yaml
```

After that, the DMB team is self-serve:

- **New service:** add a YAML to `teams/dmb/{env}/services/` — the git files generator picks it up automatically.
- **Config change to namespace/api-gateway/oauth2-proxy:** edit the values file in `teams/dmb/{env}/` — ArgoCD syncs automatically.
- **oauth2-proxy chart promotion:** `promote-chart.yml` updates `chartRevision` in `applicationsets/oauth2-proxy.yaml` — no platform team involvement.

## oauth2-proxy per-env chart pinning

The `oauth2-proxy` ApplicationSet uses a `chartRevision` field in the list generator elements. This replaces the per-env `targetRevision` that was previously hardcoded in separate `Application` files:

```yaml
generators:
  - list:
      elements:
        - env: eu-dev
          chartRevision: HEAD          # canary — always latest
        - env: eu-staging
          chartRevision: chart-oauth2-proxy-10.1.4   # promoted by CI
        - env: eu-prod
          chartRevision: chart-oauth2-proxy-10.1.4   # promoted by CI
```

`promote-chart.yml` must be updated to patch `chartRevision` in this file (replacing the old per-file `targetRevision` patches).

## When would you need the platform team again?

Only when adding a **new category** of workload that doesn't fit an existing ApplicationSet (e.g., a new shared infrastructure component outside the DMB service pattern). For new services, the git files generator handles it automatically.
