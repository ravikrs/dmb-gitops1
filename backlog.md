# Backlog

## Ready
- [ ] Populate eu-staging and eu-prod values files with real env-specific values once available
- [ ] Add CiliumNetworkPolicy per-service egress rules (currently namespace-level only)
  <!-- Context: dmb-service chart already has per-service ingress (who can call me) via networkPolicy.allowIngressFromNamespace.
       But egress is just allowEgressToSameNamespace=true — allows talking to ANY pod in the namespace.
       Goal: replace that with explicit toEndpoints selectors so each service only egresses to the services it actually calls.
       Rough call graph: in-car-api→orchestrator, orchestrator→{in-car-transformer,vin-resolver}, vin-resolver→(external),
       in-car-transformer→(Azure Service Bus via namespace-level FQDN policy), simulator→orchestrator.
       Implementation: add networkPolicy.allowEgressToServices list in dmb-service values; template iterates and emits
       toEndpoints: matchLabels: app.kubernetes.io/name: <svc> per entry. Remove allowEgressToSameNamespace flag once done.
       The namespace-level FQDN egress (dmb-namespace chart) stays as-is — it covers external traffic. -->

## In Progress

## Ideas / Not Ready
- [ ] Evaluate replacing KrakenD with Traefik middleware (KrakenD deprecation path)
- [ ] Adopt KEDA for in-car-transformer (scale to 0 on Azure Service Bus message count)
- [ ] Add per-service egress CiliumNetworkPolicy rules (currently namespace-level only in dmb-namespace)
- [ ] If platform team ever provisions a ClusterSecretStore, remove SecretStore + ServiceAccount from dmb-namespace chart

## Done
- [x] Reliability improvements: startupProbe, /tmp emptyDir, HPA template, PDB staging/prod fix, sed→yq, git push retry, eu-staging smoke render
- [x] Add GitHub Actions CI/CD: helm-lint, promote-image, promote-gateway, promote-stage workflows
- [x] Improve promotion model: semver tag filter for staging, remove promotion-gates.json, stale PR superseding, gateway manual-only staging, fix YAML PR body indentation bug in all promotion workflows
- [x] Wire oauth2-proxy to use Gateway API HTTPRoute instead of nginx Ingress
- [x] Add Dependabot for Helm chart dependency updates (oauth2-proxy upstream)
- [x] Restrict IMAGE validation to `acrdmb.azurecr.io` only in promote-image.yml and promote-gateway.yml
- [x] Bootstrap repo structure with 4 charts and ApplicationSet wiring
- [x] Create dmb-service shared chart (consolidated 5 Spring Boot services)
- [x] Create dmb-namespace chart (SecretStore, ExternalSecrets, SA, NetworkPolicy)
- [x] Create api-gateway chart (KrakenD + HTTPRoute)
- [x] Create oauth2-proxy wrapper chart
- [x] Create teams/ values files for eu-dev, eu-staging, eu-prod
- [x] Create clusters/ ArgoCD manifests + bootstrap Root Apps for all 3 clusters
- [x] Write architecture and design decisions docs
