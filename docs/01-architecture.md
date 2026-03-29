# Architecture

## Overview

This repository is the GitOps source of truth for the DMB dev team. It manages all Helm charts and ArgoCD ApplicationSets for three environments:

| Environment | Namespace | ArgoCD |
|---|---|---|
| eu-dev | `dmb-dev` | eu-dev ArgoCD instance |
| eu-staging | `dmb-staging` | eu-staging ArgoCD instance |
| eu-prod | `dmb-prod` | eu-prod ArgoCD instance |

The platform team provisions each cluster (AKS), ESO operator, Traefik + Gateway API CRDs, and Cilium CNI. The dev team owns everything in their namespace.

---

## Repo Structure

```
dmb-gitops1/
├── bootstrap/          # Root Apps — applied once manually per cluster
├── clusters/           # Per-cluster ArgoCD Application/ApplicationSet manifests
├── charts/             # Helm charts
│   ├── dmb-service/    # Shared chart for all 5 Spring Boot services
│   ├── api-gateway/    # KrakenD API gateway
│   ├── oauth2-proxy/   # OAuth2 proxy wrapper
│   └── dmb-namespace/  # Namespace bootstrap (secrets, SA, network policy)
└── teams/
    └── dmb/
        ├── eu-dev/
        │   ├── _shared.yaml              # Common values for all 5 services
        │   ├── dmb-namespace-values.yaml
        │   ├── api-gateway-values.yaml
        │   ├── oauth2-proxy-values.yaml
        │   └── services/                 # One values file per Spring Boot service
        ├── eu-staging/
        └── eu-prod/
```

---

## Charts

### `dmb-service` — Shared Spring Boot Chart

Used by all 5 microservices: `in-car-transformer`, `orchestrator`, `simulator`, `vin-resolver`, `in-car-api`.

**Key design decisions:**

**Single ConfigMap** — replaces the 6–13 per-service Spring Boot configmaps from the old `digital-manual-infrastructure` repo. Each service values file has an `applicationProperties: {}` block. The chart renders it as one ConfigMap with key `application.yaml`, mounted at `/config/application.yaml`. Spring Boot picks it up via:

```
SPRING_CONFIG_ADDITIONAL_LOCATION=file:/config/application.yaml
```

**Conditional features** — all optional resources are gated behind `enabled` flags:

| Feature | Values key | Default |
|---|---|---|
| gRPC port + service port | `grpc.enabled` | `false` |
| Gateway API HTTPRoute | `httpRoute.enabled` | `false` |
| ESO ExternalSecret | `externalSecret.enabled` | `false` |
| OTel auto-instrumentation | `opentelemetry.enabled` | `false` |
| CiliumNetworkPolicy | `networkPolicy.enabled` | `true` |

**Templates:** `_helpers.tpl`, `deployment.yaml`, `service.yaml`, `configmap.yaml`, `external-secret.yaml`, `httproute.yaml`, `cilium-network-policy.yaml`, `pod-disruption-budget.yaml`

---

### `api-gateway` — KrakenD Chart

Standalone chart because it has unique resources: KrakenD JSON configmap, HPA, `/__health` probes (not Spring Boot actuator), `readOnlyRootFilesystem: true`.

**Ingress:** nginx Ingress is removed. Traffic enters via a Gateway API `HTTPRoute` pointing to Traefik (set `httpRoute.enabled: true` in the env values file).

---

### `oauth2-proxy` — Wrapper Chart

Thin wrapper around the upstream `oauth2-proxy` Helm chart (pinned at `10.1.4`). Adds a `CiliumNetworkPolicy` and `PodDisruptionBudget` on top.

The `existingSecret` in values must point to the `{stage}-oauth2-proxy-secrets` secret created by the `dmb-namespace` chart.

---

### `dmb-namespace` — Namespace Bootstrap Chart

Deployed first (sync wave 0) before any service. Contains:

- **`SecretStore`** — namespace-scoped ESO provider pointing to Azure Key Vault (Workload Identity auth)
- **`ExternalSecret`** — 8 shared secrets: postgres, servicebus, blob, idk-client, cloud-idp, api-gateway, dmcu, oauth2-proxy
- **`ServiceAccount`** — Azure Workload Identity SA referenced by all pods
- **`CiliumNetworkPolicy`** — namespace-level egress rules to external Azure and VW Group FQDNs

All services reference secrets by name (e.g. `dev-postgres-secrets`) via `envFrom` in their values files.

---

## ApplicationSet Pattern

The 5 Spring Boot services are managed by a single git file-based `ApplicationSet` per cluster. Adding a new service = drop a new values file in `teams/dmb/{env}/services/`.

```
ApplicationSet (dmb-services-eu-dev)
  git generator → teams/dmb/eu-dev/services/*.yaml
    → dmb-in-car-transformer-eu-dev  (uses _shared.yaml + in-car-transformer.yaml)
    → dmb-orchestrator-eu-dev        (uses _shared.yaml + orchestrator.yaml)
    → dmb-simulator-eu-dev           (uses _shared.yaml + simulator.yaml)
    → dmb-vin-resolver-eu-dev        (uses _shared.yaml + vin-resolver.yaml)
    → dmb-in-car-api-eu-dev          (uses _shared.yaml + in-car-api.yaml)
```

The `dmb-namespace`, `api-gateway`, and `oauth2-proxy` are individual Applications (not ApplicationSets) since there is one of each per cluster.

---

## Sync Wave Order

ArgoCD applies resources in wave order within a cluster:

| Wave | Application | Chart |
|---|---|---|
| 0 | `dmb-namespace-{env}` | `dmb-namespace` |
| 1 | `api-gateway-{env}` | `api-gateway` |
| 1 | `oauth2-proxy-{env}` | `oauth2-proxy` |
| 2 | `dmb-services-{env}` ApplicationSet → 5 apps | `dmb-service` |

Wave 0 must complete first so that the `SecretStore`, `ServiceAccount`, and shared `ExternalSecrets` exist before any service pod starts.

---

## Bootstrap — Getting Started

Each cluster's ArgoCD is independent. Bootstrap once per cluster:

```bash
# eu-dev
kubectl --context eu-dev apply -f bootstrap/eu-dev.yaml -n argocd

# eu-staging
kubectl --context eu-staging apply -f bootstrap/eu-staging.yaml -n argocd

# eu-prod
kubectl --context eu-prod apply -f bootstrap/eu-prod.yaml -n argocd
```

The Root App watches its `clusters/{env}/` directory and auto-creates the child Applications and ApplicationSets.

---

## Adding a New Service

1. Create `teams/dmb/eu-dev/services/{service-name}.yaml` with `nameOverride` and overrides
2. Repeat for `eu-staging` and `eu-prod`
3. Push — the ApplicationSet picks it up automatically

No changes to charts or ArgoCD manifests needed.

---

## Populating Real Values

All `CHANGEME` placeholders need to be filled per environment in:

- `teams/dmb/{env}/dmb-namespace-values.yaml` — AKV name, tenant ID, SA client ID, blob account names
- `teams/dmb/{env}/api-gateway-values.yaml` — image, VAS issuer/JWK URL, blob account, hostname
- `teams/dmb/{env}/oauth2-proxy-values.yaml` — OIDC issuer URL, redirect URL
- `teams/dmb/{env}/services/*.yaml` — image repository/tag, DB names, hostnames

---

## Local Validation

```bash
# Lint charts
helm lint charts/dmb-service
helm lint charts/dmb-namespace
helm lint charts/api-gateway

# Render a service (dry run)
helm template dmb-in-car-transformer charts/dmb-service \
  -f teams/dmb/eu-dev/_shared.yaml \
  -f teams/dmb/eu-dev/services/in-car-transformer.yaml

# Render namespace bootstrap
helm template dmb-namespace charts/dmb-namespace \
  -f teams/dmb/eu-dev/dmb-namespace-values.yaml
```
