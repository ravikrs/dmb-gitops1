# Design Decisions

_Session: 2026-03-29_

Key decisions made when designing this repo, with rationale, for future reference.

---

## 1. Single `dmb-service` chart for all 5 Spring Boot services

**Decision:** One shared Helm chart serves `in-car-transformer`, `orchestrator`, `simulator`, `vin-resolver`, and `in-car-api`.

**Why:** All 5 services share the same deployment pattern — identical Deployment/Service/PDB structure, same Spring Boot health probes (`/actuator/health/liveness`), same security context, same image pull secret. The old `digital-manual-infrastructure` repo had 5 separate charts with ~60% duplication. The `dmb-gitops` reference repo already proved a single chart works.

**How differentiation works:** Per-service `values.yaml` files override replicas, resources, grpc port, applicationProperties, and env vars. The `_shared.yaml` per environment provides the common floor.

---

## 2. Single ConfigMap per Spring Boot service (`applicationProperties: {}`)

**Decision:** Replace the 6–13 per-service Spring Boot configmaps with one ConfigMap per service, rendered from `applicationProperties: {}` in the values file.

**Why:** The old approach had 13 separate `application-{profile}.yml` configmaps that Spring Boot activated via profiles. This scattered config across many files and made it hard to see the full config for a service at a glance. A single `application.yaml` mounted at `/config/` and picked up via `SPRING_CONFIG_ADDITIONAL_LOCATION=file:/config/application.yaml` is simpler, easier to diff, and avoids Spring profile activation complexity.

**Trade-off:** Slightly larger values files, but the consolidated view is worth it.

---

## 3. `dmb-namespace` chart name (not `global`)

**Decision:** The namespace bootstrap chart is named `dmb-namespace`, not `global`.

**Why:** `global` is ambiguous — it could mean cluster-global or chart-library-global. `dmb-namespace` is unambiguous: it bootstraps the DMB team's namespace-scoped platform resources (SecretStore, ServiceAccount, ExternalSecrets, CiliumNetworkPolicy egress). It mirrors how you think about it operationally: "apply the namespace setup first."

---

## 4. `api-gateway` (KrakenD) kept as a separate chart

**Decision:** KrakenD is not folded into `dmb-service`. It has its own chart.

**Why:** KrakenD is fundamentally different from the Spring Boot services — different binary (Go, not Java), different health probe (`/__health` not `/actuator/health`), different configmap shape (JSON not YAML), `readOnlyRootFilesystem: true`, HPA. Folding it into `dmb-service` would require enough conditionals to make the chart harder to reason about.

---

## 5. `oauth2-proxy` kept as a separate chart

**Decision:** `oauth2-proxy` stays as its own wrapper chart around the upstream `oauth2-proxy` Helm chart.

**Why:** It wraps an external Helm dependency. The upstream chart controls its own Deployment, Service, Ingress. Our chart only adds a `CiliumNetworkPolicy` and `PodDisruptionBudget` on top. Merging this into `dmb-service` would require fighting against the upstream chart's structure.

---

## 6. Gateway API HTTPRoute replaces nginx Ingress

**Decision:** All external exposure uses Gateway API `HTTPRoute` (Traefik parent). nginx Ingress is removed.

**Why:** The reference repo (`dmb-gitops`) already uses Gateway API + Traefik. Platform team provisions Traefik as the Gateway API implementation. nginx Ingress is being deprecated in the platform.

**KrakenD status:** KrakenD is kept for now (the API gateway function it provides — token validation, multi-tenant blob routing — is still needed). It is exposed via HTTPRoute rather than Ingress. A future backlog item exists to evaluate whether KrakenD can be replaced with Traefik middleware.

---

## 7. No KEDA for in-car-transformer (deferred)

**Decision:** `in-car-transformer` uses fixed `replicaCount: 3`, not KEDA event-driven scaling.

**Why:** The reference repo uses KEDA to scale transformers to 0 based on Azure Service Bus message count (cost optimisation). This team chose to defer KEDA adoption to keep the initial setup simple. The `dmb-service` chart has no KEDA templates; it can be added later if the team decides to adopt it.

---

## 8. Dev team owns the ESO `SecretStore` (namespace-scoped)

**Decision:** The `dmb-namespace` chart creates a namespace-scoped `SecretStore` pointing to Azure Key Vault.

**Why:** The platform team does not provision a `ClusterSecretStore` for this team. The dev team manages its own `SecretStore` inside `dmb-namespace` using Workload Identity (the `ServiceAccount` and managed identity client ID are also in this chart).

**Alternative considered:** If the platform team ever provides a `ClusterSecretStore`, the `SecretStore` and `ServiceAccount` can be removed from `dmb-namespace` and `ExternalSecrets` can reference the cluster-scoped store instead.

---

## 9. ApplicationSet git file-based generator for services

**Decision:** One `ApplicationSet` per cluster using the git file-based generator, discovering `teams/dmb/{env}/services/*.yaml`.

**Why:** Adding a new service requires dropping one YAML file — no changes to charts or ArgoCD manifests. This is the same pattern used in the reference repo for the Rancher cluster. It scales naturally.

**Per-service files** provide `nameOverride` (which ApplicationSet uses for application naming and values file path resolution) plus service-specific overrides on top of `_shared.yaml`.

---

## 10. `_shared.yaml` naming convention (underscore prefix)

**Decision:** The shared values baseline per environment is named `_shared.yaml`, not `shared.yaml`.

**Why (visual sorting):** The `_` prefix sorts before any alphabetically-named service file (`api.yaml`, `worker.yaml`, etc.) in directory listings, making it immediately obvious that this file is a base layer, not a service definition.

**Why (disambiguation):** The ApplicationSet git file generator discovers service entries via `teams/dmb/{env}/services/*.yaml`. The `_shared.yaml` file lives one level up (directly in `teams/dmb/{env}/`) and its name signals clearly that it is infrastructure, not a discoverable service. This prevents any accidental match if glob patterns ever widen.

**Trade-off:** The underscore is a convention, not enforced by tooling. If renamed to `shared.yaml`, ArgoCD would still work — the filename is hardcoded in the ApplicationSet `valueFiles` list, not discovered. The underscore is purely a readability signal.
