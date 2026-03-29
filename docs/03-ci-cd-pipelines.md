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
App CI builds image
       │
       ▼
repository_dispatch ──► promote-image.yml or promote-gateway.yml
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
             [dev gate=true]      [staging gate=true]
             auto-commit           opens staging PR
             to main                     │
                │                        │ (human review + merge)
                ▼                        ▼
         ArgoCD syncs           promote-stage.yml fires
           eu-dev               opens prod PR
                                         │
                                         │ (human review + merge)
                                         ▼
                                  ArgoCD syncs eu-prod
```

- **eu-dev**: fully automated — workflow commits the new tag directly to main.
- **eu-staging**: PR-based — a branch `promote/eu-staging/<service>/<tag>` is opened for review.
- **eu-prod**: PR-based — opened automatically by `promote-stage.yml` when the staging PR is merged.

---

## Helm Lint (`helm-lint.yml`)

Runs in a 4-way matrix (one job per chart). Each job:

1. `helm dependency update` (oauth2-proxy only — has an upstream chart dependency)
2. `helm lint charts/<chart>` — validates chart structure and default values
3. `helm template ...` smoke render using eu-dev values — catches template rendering errors

The eu-dev values files use `CHANGEME` placeholders which render as strings; this is intentional and does not cause failures.

**To extend:** if a new chart is added under `charts/`, add it to the `matrix.chart` list and add a `case` block in the smoke render step.

---

## Promotion Gates (`.github/promotion-gates.json`)

Controls which services are allowed to promote to each environment:

```json
{
  "in-car-api": { "dev": true, "staging": false, "prod": false },
  ...
}
```

- `dev: true` — auto-commit is enabled (default for all services)
- `staging: false` — staging PR will not be opened (default until eu-staging is provisioned)
- `prod` is not read by the workflow directly; prod promotion always goes through `promote-stage.yml`

To enable staging promotion for a service: set `"staging": true` in the gates file.

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
