# Forecastle Deployment

Umbrella Helm chart that packages and configures [Forecastle](https://github.com/stakater/Forecastle), the
Steadforce application dashboard, and exposes it through Istio.

> [!IMPORTANT]
> Never install the content of this repository on our clusters manually. Deployment is fully managed by Argo CD.

## Overview

- Pulls in the `forecastle` chart from the [Stakater chart repository](https://stakater.github.io/stakater-charts)
  as a dependency. The pinned version is set in `Chart.yaml` and `Chart.lock`, and its archive is committed under
  `charts/`.
- Configures Forecastle with the `STEADFORCE APPS` title and header colors, discovers apps through the
  `ForecastleApp` CRD, and selects ingresses from any namespace.
- Creates the release namespace with the `istio-injection: enabled` label.
- Routes `applications.<domain>` through the `applications-acme` Istio `VirtualService` to the `forecastle`
  Service on port `80`, with an HSTS response header. The `VirtualService` is rendered only when the
  `networking.istio.io/v1beta1/VirtualService` API is available. The `applications-acme-gateway` it references in
  the ingress gateway namespace is not part of this chart.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/), for running Helm and helm-unittest in containers.

All commands run from the repository root.

## Repository Layout

| File / Directory | Purpose |
| --- | --- |
| `Chart.yaml` | Declares the `forecastle` chart dependency of this umbrella chart. |
| `Chart.lock` | Pins the resolved dependency version. |
| `charts/` | Committed archive of the `forecastle` dependency. |
| `values.yaml` | Default ACME domain, Istio ingress gateway namespace, and `applications` subdomain. |
| `values-subchart-overrides.yaml` | Overrides for the `forecastle` dependency: dashboard config and resources. |
| `values-local.yaml` | Zero CPU and memory requests and zero CPU limit for the local cluster. |
| `values-development.yaml`, `values-production.yaml` | ACME domains of `sf-k8s01-dev` and `sf-k8s01-prod`. |
| `values-sf-k8s03-dev.yaml` | ACME domain of `sf-k8s03-dev`, layered on top of `values-development.yaml`. |
| `templates/` | The release `Namespace` and the `applications-acme` `VirtualService`. |
| `tests/` | Helm unittest suites, with git-ignored snapshots. |
| `renovate.json` | Renovate configuration for this repository. |
| `.github/workflows/` | CI workflows for unit tests and secret scanning. |

> [!NOTE]
> `values-subchart-overrides.yaml` is kept separate from the environment value files so that unit tests can catch
> incompatible changes in the values of the subchart on their own. This split is necessary because Helm does not
> allow switching off `values.yaml`.

## Environments

Argo CD renders the chart for each cluster. Its value files are configured outside this repository; the unit
tests and the rendering example below use these combinations:

| Cluster | Value Files |
| --- | --- |
| `local` | `values-subchart-overrides.yaml`, `values-local.yaml` |
| `sf-k8s01-dev` | `values-subchart-overrides.yaml`, `values-development.yaml` |
| `sf-k8s01-prod` | `values-subchart-overrides.yaml`, `values-production.yaml` |
| `sf-k8s03-dev` | `values-subchart-overrides.yaml`, `values-development.yaml`, `values-sf-k8s03-dev.yaml` |

## Rendering

Render the manifests of every cluster into the git-ignored `_render_output/<cluster>/` folder:

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     for cluster in \
       local=values-local.yaml \
       sf-k8s01-dev=values-development.yaml \
       sf-k8s01-prod=values-production.yaml \
       sf-k8s03-dev=values-development.yaml,values-sf-k8s03-dev.yaml; do
       helm template \
         -a networking.istio.io/v1beta1/VirtualService \
         -f "values-subchart-overrides.yaml,${cluster#*=}" \
         --include-crds \
         -n forecastle \
         --output-dir "_render_output/${cluster%%=*}" \
         --skip-tests \
         forecastle \
         .
     done
   '
```

`-a networking.istio.io/v1beta1/VirtualService` makes the `VirtualService` render, as it does on clusters with
Istio installed.

## Testing

The dependency archive is committed under `charts/`, so the unit tests run right after cloning:

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest \
   .
```

> [!TIP]
> Add `-t JUnit -o test-output.xml` after `helmunittest/helm-unittest` to also write a JUnit report, as the pipeline
> does. Without `-t`, helm-unittest writes the report in XUnit format. `test-output.xml` is git-ignored.

## Continuous Integration

Both workflows call reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows), pinned to `v4.2.0`.

- `helm-unittest.yaml` runs on every push. It installs the dependency pinned in `Chart.lock`, runs the Helm
  unittest suite including subchart tests, publishes a JUnit test report, and runs `helm lint`.
- `trufflehog.yaml` scans the commits of pushes and pull requests to `main`, and runs on demand, for leaked
  secrets.

### Microsoft Teams Notifications

On branches starting with `renovate/`, the unittest workflow posts its result to Microsoft Teams:

| Result | Repository Secret |
| --- | --- |
| Success | `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` |
| Failure | `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK`, a separate error channel |

Both secrets are optional and hold a Microsoft Teams Workflows webhook URL. When the error webhook is not set,
failures go to `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` instead. Without either secret, no notification is sent.

## Dependency Updates

Renovate opens pull requests for dependency updates, based on `config:recommended` with a dependency dashboard.
Nothing is automerged. With `helmUpdateSubChartArchives`, Renovate also replaces the archive in `charts/` when it
updates the `forecastle` chart.

When changing the dependency version in `Chart.yaml` by hand, update the lock file and the archive, then commit
`Chart.yaml`, `Chart.lock`, and the new archive in `charts/` together:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies) for details on chart dependencies.
