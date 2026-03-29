# oauth2-proxy Chart Promotion Model

## Problem

Dependabot bumps `charts/oauth2-proxy/Chart.yaml` (the upstream helm chart dependency) by opening a PR against `main`. Before this change, all three ArgoCD Applications pointed to `targetRevision: HEAD`. When the Dependabot PR merged, ArgoCD would sync the new chart version to **all environments simultaneously** — no staging bake time, no gate.

The existing `promote-image.yml` / `promote-stage.yml` pipelines only operate on values files (image tags). They do not touch `Chart.yaml`, so Dependabot upgrades bypassed the promotion model entirely.

## Design decision: own charts vs third-party charts

| Chart type | Strategy | Reason |
|---|---|---|
| `dmb-service`, `api-gateway`, `dmb-namespace` | `targetRevision: HEAD` + env-specific values | Team controls all changes; PR review is the gate; no uncontrolled promotion risk |
| `oauth2-proxy` (wraps upstream) | Pinned `targetRevision` per env (staging/prod) | Dependabot can auto-merge without a human deciding to promote to prod; upstream changes are uncontrolled |

The rule: **gate what you don't control; trust your own review process for what you do**.

## How it works

```
Dependabot PR merges to main
        │
        ▼
charts/oauth2-proxy/Chart.yaml changes
        │
        ▼ (promote-chart.yml trigger: push to main, path filter)
Creates git tag: chart-oauth2-proxy-<version>
        │
        ├─► eu-dev: targetRevision=HEAD → ArgoCD auto-syncs immediately
        │
        └─► Opens staging PR
              Branch: promote-chart/eu-staging/oauth2-proxy/<version>
              Updates: clusters/eu-staging/oauth2-proxy.yaml targetRevision
                │
                ▼ (human reviews + merges)
              Staging PR merged
                │
                ▼ (promote-chart.yml trigger: PR closed, branch pattern)
              Opens prod PR
                Branch: promote-chart/eu-prod/oauth2-proxy/<version>
                Updates: clusters/eu-prod/oauth2-proxy.yaml targetRevision
                  │
                  ▼ (human reviews + merges)
                eu-prod syncs
```

## What the git tag points to

The tag `chart-oauth2-proxy-<version>` is created on the `main` commit where `Chart.yaml` contains that dependency version. ArgoCD uses the tag to checkout `charts/oauth2-proxy/` at that exact state when rendering the chart for staging/prod.

The values source (`ref: values`) always stays on `HEAD` in all environments. This means OIDC configuration, hostnames, and other values changes deploy immediately without needing a chart promotion. Only the chart templates/dependencies are gated.

## ArgoCD Application structure

```yaml
sources:
  # Pinned to chart tag in staging/prod; HEAD in dev.
  - repoURL: https://github.com/ravikrs/dmb-gitops1
    path: charts/oauth2-proxy
    targetRevision: chart-oauth2-proxy-10.1.4   # bumped by promote-chart.yml
    helm:
      valueFiles:
        - $values/teams/dmb/<env>/oauth2-proxy-values.yaml
  # Always HEAD — values changes are not gated.
  - repoURL: https://github.com/ravikrs/dmb-gitops1
    targetRevision: HEAD
    ref: values
```

## Bootstrap: creating the initial tag

When this promotion model was introduced, the current dependency version was `10.1.4`. The initial tag must be created manually once:

```bash
git tag chart-oauth2-proxy-10.1.4
git push origin chart-oauth2-proxy-10.1.4
```

All future version bumps are tagged automatically by `promote-chart.yml`.

## Workflow: `.github/workflows/promote-chart.yml`

| Trigger | Job | Action |
|---|---|---|
| `push` to `main`, path `charts/oauth2-proxy/Chart.yaml` | `tag-and-open-staging-pr` | Creates tag, opens staging PR |
| `pull_request` closed+merged, branch `promote-chart/eu-staging/*` | `open-prod-pr` | Opens prod PR |

The workflow is idempotent: if the tag or staging PR already exists (e.g. Dependabot re-proposes the same version), it skips without error.

## Comparison to image promotion

| Aspect | Image promotion | Chart promotion |
|---|---|---|
| Trigger | `repository_dispatch` from app CI | `push` to main (Dependabot merge) |
| Dev | Auto-commit to main | HEAD (ArgoCD auto-syncs) |
| Staging gate | Semver tag regex + manual PR | Manual PR |
| Prod gate | Auto PR after staging merge | Auto PR after staging merge |
| What is bumped | `image.tag` in values file | `targetRevision` in ArgoCD Application YAML |
