# oauth2-proxy HTTPRoute + Dependabot

## Changes

### Wire oauth2-proxy to Gateway API HTTPRoute (Traefik)

The upstream oauth2-proxy chart's nginx Ingress (`ingress.enabled`) was already disabled. This change adds a proper HTTPRoute so Traefik can route external traffic to oauth2-proxy's OIDC endpoints.

**New template:** `charts/oauth2-proxy/templates/httproute.yaml`

Routes the `/oauth2` path prefix to the oauth2-proxy service:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs:
    - name: traefik
      namespace: ingress
      sectionName: websecure
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /oauth2
      backendRefs:
        - name: <release-name>      # upstream subchart fullname = .Release.Name
          port: 80                  # upstream chart default ClusterIP port
```

**Why `/oauth2`:** oauth2-proxy serves its callback (`/oauth2/callback`), sign-in page (`/oauth2/sign_in`), and the ForwardAuth endpoint (`/oauth2/auth`) all under this prefix. Traefik's ForwardAuth middleware calls `/oauth2/auth` internally via ClusterIP; only the callback path needs to be reachable externally.

**Service name convention:** The upstream subchart fullname resolves to `.Release.Name` (not `<release>-oauth2-proxy`) because the release name already contains the chart name (`oauth2-proxy-eu-dev` contains `oauth2-proxy`). The template uses `{{ .Release.Name }}` accordingly.

**CiliumNetworkPolicy:** The existing ingress rule (`fromEndpoints: namespace: ingress`) already covers Traefik — no structural change needed, a clarifying comment was added.

**Values added to `charts/oauth2-proxy/values.yaml`:**

```yaml
httpRoute:
  enabled: false
  parentRefs:
    - name: traefik
      namespace: ingress
      sectionName: websecure
  hostnames: []
  servicePort: 80
```

**Per-env values** (`teams/dmb/{eu-dev,eu-staging,eu-prod}/oauth2-proxy-values.yaml`): `httpRoute.enabled: true` with `hostnames: [CHANGEME]` added to each — fill in the real hostname when the cluster DNS is known.

---

### Dependabot for Helm chart dependencies

**New file:** `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: helm
    directory: /charts/oauth2-proxy
    schedule:
      interval: weekly
```

Tracks the `dependencies:` block in `charts/oauth2-proxy/Chart.yaml` (the upstream oauth2-proxy chart). Does **not** manage image tags — those are handled by `promote-image.yml`.

## What to fill in before going live

| File | Field |
|------|-------|
| `teams/dmb/eu-dev/oauth2-proxy-values.yaml` | `httpRoute.hostnames[0]` — e.g. `dmb-dev.apps.example.com` |
| `teams/dmb/eu-dev/oauth2-proxy-values.yaml` | `oauth2-proxy.extraArgs.redirect-url` — e.g. `https://dmb-dev.apps.example.com/oauth2/callback` |
| `teams/dmb/eu-staging/oauth2-proxy-values.yaml` | same fields for staging hostname |
| `teams/dmb/eu-prod/oauth2-proxy-values.yaml` | same fields for prod hostname |
