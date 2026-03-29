# Backlog

## Ready
- [ ] Populate eu-staging and eu-prod values files with real env-specific values once available
- [ ] Wire oauth2-proxy to use Gateway API HTTPRoute instead of nginx Ingress
- [ ] Add CiliumNetworkPolicy per-service egress rules (currently namespace-level only)
- [ ] Add Renovate / Dependabot for chart dependency updates (oauth2-proxy upstream)

## In Progress

## Ideas / Not Ready
- [ ] Evaluate replacing KrakenD with Traefik middleware (KrakenD deprecation path)
- [ ] Adopt KEDA for in-car-transformer (scale to 0 on Azure Service Bus message count)
- [ ] Add per-service egress CiliumNetworkPolicy rules (currently namespace-level only in dmb-namespace)
- [ ] If platform team ever provisions a ClusterSecretStore, remove SecretStore + ServiceAccount from dmb-namespace chart

## Done
- [x] Reliability improvements: startupProbe, /tmp emptyDir, HPA template, PDB staging/prod fix, sed→yq, git push retry, eu-staging smoke render
- [x] Add GitHub Actions CI/CD: helm-lint, promote-image, promote-gateway, promote-stage workflows
- [x] Bootstrap repo structure with 4 charts and ApplicationSet wiring
- [x] Create dmb-service shared chart (consolidated 5 Spring Boot services)
- [x] Create dmb-namespace chart (SecretStore, ExternalSecrets, SA, NetworkPolicy)
- [x] Create api-gateway chart (KrakenD + HTTPRoute)
- [x] Create oauth2-proxy wrapper chart
- [x] Create teams/ values files for eu-dev, eu-staging, eu-prod
- [x] Create clusters/ ArgoCD manifests + bootstrap Root Apps for all 3 clusters
- [x] Write architecture and design decisions docs
