# ADR: GitOps Repository Structure

_Date: 2026-03-30_

---

## Status

**Proposed** — for DMB team review.

---

## Context

The DMB team has two generations of Kubernetes deployment configuration:

- **`digital-manual-infrastructure`** — the original repo, combining Azure infrastructure-as-code (Terragrunt/Terraform), Helm charts (one per service), and local dev tooling in a single repository.
- **`dmb-gitops1`** — the new repo, a dedicated GitOps repo with a shared Helm chart pattern, ArgoCD ApplicationSets, and an automated image promotion pipeline.

This ADR records the structural differences between the two approaches, the reasoning behind the new structure, and the trade-offs involved. The goal is to align the team on _why_ the new structure exists so future decisions are made from a shared foundation.

---

## Decision

**Use `dmb-gitops1` as the canonical structure for Kubernetes deployment configuration going forward.**

Specifically, adopt:

1. A dedicated GitOps repo (separate from infrastructure/IaC repos)
2. A single shared Helm chart (`dmb-service`) for all Spring Boot microservices
3. A `teams/{team}/{env}/services/` directory tree for environment-specific values
4. ArgoCD ApplicationSets with a git file generator to manage all services
5. An automated image promotion pipeline (dev → staging PR → prod PR) gated on semver

---

## Comparison

### Repo structure at a glance

| Dimension | `digital-manual-infrastructure` | `dmb-gitops1` |
|---|---|---|
| **Scope** | IaC + Helm + local dev tooling | Helm + ArgoCD only (app-team scope) |
| **Chart pattern** | 8 individual charts (one per service) | 1 shared `dmb-service` + 3 specialized |
| **Env config location** | `values-{env}.yaml` inside each chart | `teams/dmb/{env}/services/*.yaml` |
| **ArgoCD manifests** | Not present | Root App + ApplicationSets |
| **Image promotion** | Manual / not codified | 4 automated workflows (semver-gated) |
| **Multi-env coverage** | Dev fully codified; staging/prod manual | Dev/staging/prod all defined |
| **Ingress approach** | nginx Ingress | Gateway API HTTPRoute (Traefik) |
| **Documentation** | README only | MkDocs site (architecture, decisions, CI/CD) |

---

## Rationale

### 1. Separation of concerns: GitOps repo vs. IaC repo

`digital-manual-infrastructure` mixes two very different concerns:
- **Cloud infrastructure** (Azure RGs, keyvaults, PostgreSQL, Service Bus, VNet) managed by Terragrunt
- **Kubernetes deployment config** (Helm charts, values) consumed by ArgoCD

These change at different rates, have different reviewers, and carry different blast radii. A Helm values change (image tag bump) should not require reviewing Terragrunt HCL. A keyvault change should not touch service deployment config.

Dedicated repos allow each concern to have appropriate CI, access controls, and review processes. `dmb-gitops1` contains only what ArgoCD needs to reconcile state in the cluster — nothing more.

### 2. Shared chart over individual charts

`digital-manual-infrastructure` maintains 5 separate Spring Boot charts (in-car-api, in-car-transformer, orchestrator, simulator, vin-resolver). These charts share:
- Identical Deployment/Service/PDB structure
- The same Spring Boot health probe paths (`/actuator/health/liveness`, `/actuator/health/readiness`)
- The same security context (non-root, drop ALL capabilities)
- The same image pull secret, service account, and ExternalSecret wiring

Maintaining 5 copies means every cross-cutting change (new label, updated probe path, CiliumNetworkPolicy adjustment) requires 5 edits and 5 PRs. In practice this leads to drift — services get out of sync and it becomes unclear which version of a template is canonical.

`dmb-gitops1` deploys one `dmb-service` chart 5 times with different values. Structural changes go in one place. Services diverge only where they genuinely differ (resources, gRPC port, applicationProperties).

### 3. `teams/{env}/services/` over `values-{env}.yaml`

The traditional Helm pattern places `values-dev.yaml`, `values-prod.yaml`, `values-qa.yaml` inside each chart directory. This has two problems:

**Problem 1 — Environment config is scattered across charts.** To see everything running in staging you must open 8 chart directories. To answer "what image tag is deployed in prod for all services?" requires searching across 8 files.

**Problem 2 — Hard to add a service without touching chart layout.** Adding a 6th Spring Boot service requires creating a new chart directory, wiring up values files, and updating whatever deployment tooling references that chart.

`dmb-gitops1` concentrates all environment config in `teams/dmb/{env}/`. Adding a service = dropping one YAML file in `teams/dmb/{env}/services/`. The ApplicationSet git generator discovers it automatically. No other changes required.

The `_shared.yaml` per environment also gives a single place to define the common floor for all services in that environment (Java options, OTel endpoint, PDB settings) without repeating it per service.

### 4. ArgoCD ApplicationSets over imperative Helm

`digital-manual-infrastructure` has no ArgoCD manifests. Kubernetes state is presumably applied by running `helm install/upgrade` manually or via a separate (undocumented) process.

This creates a reconciliation gap: the cluster drifts from git without anyone noticing. There is no automated mechanism to detect or remediate out-of-sync state.

`dmb-gitops1` uses ArgoCD's git file generator ApplicationSet. The cluster state is continuously reconciled against the repo. Drift is surfaced immediately in the ArgoCD UI. Self-heal ensures the cluster always reflects what is in git.

The root Application pattern (`bootstrap/{env}.yaml`) means each ArgoCD instance is fully self-configuring from git — a single `kubectl apply` bootstraps an entire environment.

### 5. Automated promotion pipeline

`digital-manual-infrastructure` has one image-related workflow: `push_oauth2_proxy_image.yml` (builds and pushes a custom oauth2-proxy image). There is no promotion model — services presumably get deployed by committing a new image tag manually.

`dmb-gitops1` has a four-stage promotion chain:
1. `promote-image.yml` — triggered by `repository_dispatch` from app CI; commits new image tag to dev automatically on every push
2. `promote-image.yml` — opens a staging PR only for semver tags (`v1.2.3`); stale staging PRs are superseded on new semver
3. `promote-stage.yml` — opens a prod PR automatically when staging is merged
4. `promote-chart.yml` — handles upstream Helm chart (oauth2-proxy) version bumps via Dependabot, with per-env pinning

This model enforces that:
- Dev is always current
- Only intentional releases (semver tags) reach staging
- Prod promotion requires a deliberate PR merge
- The promotion history is fully visible in git (commit messages, PR descriptions)

### 6. Gateway API HTTPRoute over nginx Ingress

`digital-manual-infrastructure` uses nginx Ingress for the API gateway. `dmb-gitops1` uses Traefik with Gateway API HTTPRoute.

Gateway API is the upstream successor to Ingress in the Kubernetes ecosystem. It provides:
- More expressive routing (path matching, header routing, traffic splitting)
- Role-based separation (GatewayClass owned by platform, HTTPRoute owned by teams)
- Active development; Ingress is in maintenance mode

The oauth2-proxy HTTPRoute also integrates with Traefik's `ForwardAuth` middleware at the Gateway layer, avoiding the need to configure this per-service.

---

## Trade-offs and what we give up

| Trade-off | Detail |
|---|---|
| **No infrastructure in this repo** | Cloud resource provisioning (keyvaults, PostgreSQL, Service Bus) lives in a separate IaC repo. Teams deploying to a new cluster must ensure infrastructure is pre-provisioned. |
| **No local dev tooling** | `digital-manual-infrastructure` includes a `docker-compose.yml` for running Azure Service Bus and other services locally. This tooling is not in `dmb-gitops1`. Teams may need a separate dev tooling repo or to reference the old repo for local setup. |
| **Shared chart coupling** | If one service needs a template change that other services cannot support (e.g., a sidecar only one service needs), the shared chart must be extended carefully with feature flags. This is manageable but requires discipline. |
| **ArgoCD required** | `dmb-gitops1` is not useful without ArgoCD. The old repo could be applied with `helm install` alone. |

---

## Consequences

- New Spring Boot services are added by creating one values file in `teams/dmb/{env}/services/` — no chart changes, no ApplicationSet changes.
- All Kubernetes state for the team is visible and reconciled in ArgoCD across all three environments.
- Image promotions are auditable — every tag that reaches staging/prod has a corresponding PR in git.
- Cross-cutting changes to deployment templates (probes, labels, security context) are made once and apply to all services.
- The repo stays focused on what the app team controls. Infrastructure changes do not pollute deployment config history.
