# web-chart

Helm chart to deploy `monochrome-web-ui` — the static bouc.io website served on `www.<domain>` — on Kubernetes.

The app is a fully static Vite/React build served by `serve` on container port 3000 (Service maps 80 → 3000). It uses **no runtime environment variables** and no auth, so unlike `portal-chart`/`agent-chart` there is no `environment:` (`VITE_*`) or `auth:` values block. The chart ships no Istio VirtualService — `www.${CLUSTER_DOMAIN}` routing is handled centrally in the Istio component.

## Values files

| File | Purpose |
|---|---|
| `base.values.yaml` | Defaults shared across all environments |
| `lcl.values.yaml` | Local Kubernetes cluster (pik8s.internal) overrides |
| `snbx.values.yaml` | Sandbox GKE environment overrides |

## Usage

```bash
# Lint (no default values.yaml — always pass the values files)
helm lint . -f base.values.yaml -f lcl.values.yaml

# Install (lcl)
helm install web-chart . -f base.values.yaml -f lcl.values.yaml

# Install (snbx)
helm install web-chart . -f base.values.yaml -f snbx.values.yaml
```

## Delivery

The chart is published to the GitLab package registry by CI (`.gitlab-ci.yml`) and consumed by a FluxCD HelmRelease with `valuesFrom` ConfigMaps generated from these values files. Pushing chart source to git alone is not enough — the chart version must be published before FluxCD can reconcile it.

## Follow-up checklist (FluxCD wiring — not yet done)

1. Create the GitLab project `bouc-io/application/web/web-chart` (mirror), push, let CI publish the chart tgz to the package registry; note the numeric project id for the HelmRepository URL.
2. In `fluxcdboucio`:
   - Add submodule `clusters/components/web-ui` → this chart repo.
   - `clusters/base/apps/examples/fluxcd-web-chart.yaml`: `ImageRepository` + `ImagePolicy` (semver `1.0.x`) for `web-mirror-ui`, `HelmRepository` (project-id URL), `HelmRelease web-chart-helmrelease` with `valuesFrom` ConfigMaps `web-ui-base-values` (key `base.values.yaml`) + `web-ui-level-values` (key `values.yaml`), image-tag `$imagepolicy` setter, plus `ImageUpdateAutomation` — mirror `fluxcd-portal-chart.yaml`.
   - Local overlay patch in `clusters/local/apps/examples/` (interval 1m, `imagePullSecrets: registry-credentials`, level values = `lcl.values.yaml`) + add to `clusters/local/apps/kustomization.yaml` resources and patches.
   - `clusters/{local,sandbox}/config/kustomization.yaml`: configMapGenerator entries `web-ui-base-values` / `web-ui-level-values` pointing at `../../components/web-ui/{base,lcl|snbx}.values.yaml`.
   - Add the two ConfigMaps to `config-kustomization.yaml` healthChecks and `apps-kustomization.yaml` `postBuild.substituteFrom`.
3. Repoint `www.${CLUSTER_DOMAIN}` routing from the old `static-web-chart` service to `web-chart-service.default.svc.cluster.local:80` (central VirtualService) and retire the `static-website` HelmRelease/component. Gateway `boucio-gateway`, TLS certs, and the apex→www redirect already cover `www` — no changes needed there.
