# CI/CD Pipelines

## Overview

Four GitHub Actions workflows cover chart validation and image promotion across the three environments.

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `helm-lint.yml` | PR / push to `charts/**` or `teams/**` | Lint and smoke render all 4 Helm charts |
| `promote-image.yml` | `repository_dispatch: image-updated` | Promote a Spring Boot service image through environments |
| `promote-gateway.yml` | `repository_dispatch: gateway-updated` | Promote the KrakenD gateway image through environments |
| `promote-stage.yml` | Staging PR merge / `workflow_dispatch` | Open prod PR automatically; manual env-to-env promotion |

---

## Promotion Model

```
App CI builds image  (every merge → SHA tag; release cut → v1.2.3)
       │
       ▼
repository_dispatch ──► promote-image.yml or promote-gateway.yml
                              │
                    ┌─────────┴──────────────────────┐
                    ▼                                 ▼
             always: auto-commit            semver tag only (v1.2.3):
             to eu-dev                      opens eu-staging PR
                │                                     │
                │                                     │ (human review + merge)
                ▼                                     ▼
         ArgoCD syncs                    promote-stage.yml fires
           eu-dev                        opens eu-prod PR
                                                      │
                                                      │ (human review + merge)
                                                      ▼
                                             ArgoCD syncs eu-prod
```

- **eu-dev**: fully automated — every tag auto-commits directly to main.
- **eu-staging**: PR-based, **semver tags only** (`v1.2.3`) — a branch `promote/eu-staging/<service>/<tag>` is opened. If a newer semver tag arrives before the PR is merged, the old PR is closed with a "Superseded" comment and a new one is opened.
- **eu-prod**: PR-based — opened automatically by `promote-stage.yml` when the staging PR is merged.
- **api-gateway**: dev is auto as above; staging/prod promotion is **manual only** via `workflow_dispatch` on `promote-gateway.yml`. Gateway does not follow a semver filter.

---

## Helm Lint (`helm-lint.yml`)

Runs in a 4-way matrix (one job per chart). Each job:

1. `helm dependency update` (oauth2-proxy only — has an upstream chart dependency)
2. `helm lint charts/<chart>` — validates chart structure and default values
3. `helm template ...` smoke render using eu-dev values — catches template rendering errors

The eu-dev values files use `CHANGEME` placeholders which render as strings; this is intentional and does not cause failures.

**Scope of linting:** the workflow runs the full 4-way matrix on every trigger — all 4 charts are always linted, regardless of which files changed. The path trigger (`charts/**`, `teams/**`) controls *when* the workflow fires, not which jobs run within it. So changing a single gateway values file lints all 4 charts. This is intentional: the matrix is cheap and avoids the complexity of mapping changed paths to chart names. If the number of charts grows significantly, `dorny/paths-filter` can be added to scope jobs to only affected charts.

**To extend:** if a new chart is added under `charts/`, add it to the `matrix.chart` list and add a `case` block in the smoke render step.

---

## Promotion Gates

There is no gates file. The gate for staging promotion is the image tag itself:

- **SHA tags** (e.g. `sha-abc1234`) → eu-dev only, no staging PR
- **Semver tags** (e.g. `v1.2.3`) → eu-dev auto-commit + eu-staging PR

This is enforced in `promote-image.yml` with a regex check (`^v[0-9]+\.[0-9]+\.[0-9]+$`) before the staging PR steps run. No per-service configuration is needed — all 5 Spring Boot services follow the same policy.

If a service needs to be blocked from staging temporarily, don't cut a semver release tag for it.

---

## Staging and Prod PRs

Both workflows create a branch named `promote/<env>/<service>/<tag>` and open a PR targeting `main`. The PR body includes a table showing service, image, tag, and target environment.

When a staging PR is merged:
- `promote-stage.yml` detects the merge via the `pull_request: closed` trigger
- It parses the branch name to extract service and tag
- It creates `promote/eu-prod/<service>/<tag>` and opens a prod PR

The prod PR must also be reviewed and merged manually.

### Manual promotion

`promote-stage.yml` also supports `workflow_dispatch` for ad-hoc promotions — e.g. if you want to re-promote a specific tag that wasn't auto-promoted, or promote from dev directly to staging.

---

## Dispatching from App CI

To trigger promotion from an application CI pipeline, add a step like:

```yaml
- name: Trigger gitops promotion
  uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.GITOPS_DISPATCH_TOKEN }}
    repository: ravikrs/dmb-gitops1
    event-type: image-updated          # or gateway-updated for api-gateway
    client-payload: |
      {
        "service": "in-car-api",
        "image": "acrdmb.azurecr.io/dmb/dmb-in-car-api",
        "tag": "${{ github.sha }}"
      }
```

`GITOPS_DISPATCH_TOKEN` must be a PAT (or fine-grained token) with `repo` scope on `dmb-gitops1`, stored as a secret in the app repo.

---

## Security Notes

- All GitHub expression values (`client_payload.*`, `inputs.*`) are resolved to shell env vars via `env:` blocks before use in `run:` scripts — never inlined directly.
- Service names and image tags are regex-validated in the `check-gates` job before any downstream job runs.
- The `promote-stage.yml` `open-prod-pr` job parses branch names that were set by the bot — the parsed values are still validated with regex before use.
