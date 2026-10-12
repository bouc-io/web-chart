# web-chart

Helm chart to deploy `monochrome-web-ui` — the static bouc.io website served on `www.<domain>` — on Kubernetes.

The app is a fully static Vite/React build served by `serve` on container port 3000 (Service maps 80 → 3000). It uses **no runtime environment variables** and no auth, so unlike `portal-chart`/`agent-chart` there is no `environment:` (`VITE_*`) or `auth:` values block. The chart ships no Istio VirtualService — `www.${CLUSTER_DOMAIN}` routing is handled centrally in the Istio component.

## What it deploys

A Deployment serving the static build, plus a ClusterIP Service mapping port 80 to container port
3000. No database, no cache, and no subchart dependencies.

## Values

| File | Purpose |
|---|---|
| `base.values.yaml` | Defaults shared across all environments |
| `lcl.values.yaml` | Local Kubernetes cluster (pik8s.internal) overrides |
| `snbx.values.yaml` | Sandbox GKE environment overrides |

There is no plain `values.yaml`. Always pass `base` plus exactly one environment file.

> In the cluster, FluxCD supplies values from generated ConfigMaps via `valuesFrom:`, not from these
> files directly. They are the source the ConfigMaps are generated from.

## Probes

`livenessProbe` and `readinessProbe` are left empty (`{}`) in values, so pods count as ready as soon
as the container starts.

## Local usage

```bash
# Lint (no default values.yaml — always pass the values files)
helm lint . -f base.values.yaml -f lcl.values.yaml

# Install (lcl)
helm install web-chart . -f base.values.yaml -f lcl.values.yaml

# Install (snbx)
helm install web-chart . -f base.values.yaml -f snbx.values.yaml
```

## Delivery

The chart is published to the chart registry by CI (`.gitlab-ci.yml`) and consumed by a FluxCD HelmRelease with `valuesFrom` ConfigMaps generated from these values files. Pushing chart source to git alone is not enough: the chart version must be published before FluxCD can reconcile it.

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
