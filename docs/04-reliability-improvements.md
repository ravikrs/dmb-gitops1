# Reliability Improvements

A batch of reliability and correctness fixes applied across charts, shared values, and CI/CD workflows.

## Chart: dmb-service

### startupProbe (replaces large initialDelaySeconds)

Before: both liveness and readiness probes had `initialDelaySeconds: 120`. This meant a crashed app took up to 120 seconds before Kubernetes attempted a restart, and a slow-starting app that missed the window was killed unnecessarily.

After: a `startupProbe` (same httpGet endpoint, `failureThreshold: 24`, `periodSeconds: 10`) gives the app up to **240 seconds** to start. Once the startup probe succeeds, liveness and readiness probes take over with `initialDelaySeconds: 0` and a tighter `failureThreshold: 3` — a genuinely dead pod is restarted within 90 seconds.

### /tmp emptyDir volume

All services now have an explicit `emptyDir` volume mounted at `/tmp`. Spring Boot's embedded Tomcat writes its work directory under `/tmp`; the emptyDir ensures those writes are isolated per-pod and don't bleed between restarts. Added a comment on `readOnlyRootFilesystem: false` to document this intent.

### HPA template (disabled by default)

Added `charts/dmb-service/templates/hpa.yaml` with `autoscaling/v2` and CPU + memory metrics. Disabled by default (`autoscaling.enabled: false`). Enable per-service via values:

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 6
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 90
```

Particularly relevant for `in-car-transformer` (future KEDA candidate) and `orchestrator`.

## Shared values: eu-staging and eu-prod

### pdb.minAvailable: 0 → 1

`eu-staging/_shared.yaml` and `eu-prod/_shared.yaml` both had `pdb.minAvailable: 0`, copied verbatim from dev. This makes the PDB a no-op — node drains can evict all pods simultaneously. Changed to `1` so at least one pod stays up during disruptions once services move beyond single-replica.

eu-dev intentionally keeps `minAvailable: 0` since it runs single replicas and availability guarantees are not required.

### Comment fix

Both staging and prod `_shared.yaml` files had a copy-paste comment saying "eu-dev". Fixed to correctly say "eu-staging" and "eu-prod".

## CI/CD Workflows

### sed → yq for image tag updates

All three promotion workflows (`promote-image.yml`, `promote-gateway.yml`, `promote-stage.yml`) previously used a regex sed pattern to update the image tag:

```bash
sed -i "s/^\(  tag: \).*/\1${TAG}/" "$FILE"
```

This pattern matches **any** two-space-indented `tag:` field. If a values file gains a second `tag:` key, all occurrences are silently updated.

Replaced with yq targeting the exact field path:

```bash
yq e '.image.tag = strenv("TAG")' -i "$FILE"
```

yq is pre-installed on `ubuntu-latest` GitHub Actions runners (mikefarah v4).

### Git push retry loop (promote-dev jobs)

The `promote-dev` jobs in `promote-image.yml` and `promote-gateway.yml` commit directly to `main`. If two different services are promoted concurrently, both jobs do `git pull --rebase` then `git push`, and the second push gets rejected with "non-fast-forward".

Added a retry loop (up to 3 attempts) that re-pulls and rebases before each retry:

```bash
for attempt in 1 2 3; do
  git push origin main && break || {
    if [[ $attempt -eq 3 ]]; then echo "Push failed after 3 attempts"; exit 1; fi
    git pull --rebase origin main
  }
done
```

Staging/prod jobs open PRs (push to a feature branch) so they are not affected by this race.

### eu-staging smoke render in helm-lint

`helm-lint.yml` previously only rendered charts with eu-dev values. Added a second `Smoke render (eu-staging)` step using eu-staging values files. This catches template rendering bugs introduced by env-specific values that eu-dev doesn't exercise.
