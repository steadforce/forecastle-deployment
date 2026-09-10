# Forecastle Deployment

Helm umbrella chart that packages and configures [Forecastle](https://github.com/stakater/Forecastle) for
STEADFORCE Kubernetes clusters, exposing it through an Istio `VirtualService` on a per-environment ACME domain.

> [!IMPORTANT]
> Never install the content of this repository on a cluster manually. Deployments are managed by ArgoCD.

## Overview

Forecastle is a control panel that discovers and lists the web applications running in a Kubernetes namespace.
This chart wraps the upstream `forecastle` chart as a dependency and adds:

- An Istio `VirtualService` (`templates/acme-virtual-service.yaml`) exposing Forecastle under
  `applications.<acme-domain>`.
- A `Namespace` manifest (`templates/namespace.yaml`) with Istio sidecar injection enabled.
- Per-environment values files that override the ACME domain and, for local development, the container
  resource requests and limits.

## Chart Structure

| Path                          | Description                                              |
|-------------------------------|------------------------------------------------------------|
| `Chart.yaml`                  | Chart metadata and the `forecastle` dependency declaration. |
| `charts/`                     | Downloaded dependency archive (`forecastle-v1.0.159.tgz`). |
| `templates/`                  | Root templates (namespace, Istio virtual service).         |
| `values.yaml`                 | Default values shared by all environments.                  |
| `values-<environment>.yaml`   | Per-environment overrides (see [Environments](#environments)). |
| `tests/`                      | Helm unittest suites (see [Running Tests](#running-tests)). |

## Environments

| Values File                     | Environment    | ACME Domain                        |
|----------------------------------|----------------|-------------------------------------|
| `values-local.yaml`              | Local          | `steadops.local.steadforce.com` (default, zero resource requests) |
| `values-development.yaml`        | Development    | `dev.k8s01.steadforce.com`          |
| `values-production.yaml`         | Production     | `k8s01.steadforce.com`              |
| `values-sf-k8s03-dev.yaml`       | sf-k8s03-dev   | `k8s03-dev.steadforce.com`          |
| `values-sf-k8s04-dev.yaml`       | sf-k8s04-dev   | `k8s04-dev.steadforce.com`          |

## Dependencies

This chart pulls in `forecastle` as a dependency. The version used is specified in `Chart.yaml` under
`dependencies`. If you change that version, update the downloaded archive in `charts/` and commit the
result alongside the altered `Chart.yaml` and `Chart.lock`:

```sh
 docker run \
  --rm \
  -e HOME=/tmp \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  alpine/helm dependency update .
```

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies) for details.

## Rendering Templates Locally

Render the manifests for a given environment with `helm template`, including CRDs and the Istio/cert-manager
API versions the templates depend on:

### Local

```sh
 docker run \
  --rm \
  -e HOME=/tmp \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  alpine/helm template forecastle . \
  --api-versions cert-manager.io/v1 \
  --api-versions networking.istio.io/v1beta1/VirtualService \
  --include-crds \
  --namespace forecastle \
  --output-dir _local/local \
  --skip-tests \
  --values values-local.yaml
```

### Development

```sh
 docker run \
  --rm \
  -e HOME=/tmp \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  alpine/helm template forecastle . \
  --api-versions cert-manager.io/v1 \
  --api-versions networking.istio.io/v1beta1/VirtualService \
  --include-crds \
  --namespace forecastle \
  --output-dir _local/dev \
  --skip-tests \
  --values values-development.yaml
```

### Production

```sh
 docker run \
  --rm \
  -e HOME=/tmp \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  alpine/helm template forecastle . \
  --api-versions cert-manager.io/v1 \
  --api-versions networking.istio.io/v1beta1/VirtualService \
  --include-crds \
  --namespace forecastle \
  --output-dir _local/prod \
  --skip-tests \
  --values values-production.yaml
```

> [!TIP]
> Use `--values values-sf-k8s03-dev.yaml` or `--values values-sf-k8s04-dev.yaml` to render the dedicated
> dev-cluster environments the same way.

## Running Tests

Suites live under `tests/` and cover the namespace, virtual service host per environment, and the subchart
deployment/configmap rendering driven by the root values files.

```sh
 docker run \
  --rm \
  -e HELM_CACHE_HOME=/tmp/helm/.config \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  helmunittest/helm-unittest .
```

To produce a JUnit report instead:

```sh
 docker run \
  --rm \
  -e HELM_CACHE_HOME=/tmp/helm/.config \
  -u $(id -u) \
  -v "$(pwd):/apps" \
  -w /apps \
  helmunittest/helm-unittest -o test-output.xml .
```

## Continuous Integration

- `.github/workflows/helm-unittest.yaml` runs the Helm unittest suites on every push.
- `.github/workflows/trufflehog.yaml` scans the repository for leaked secrets on pushes and pull requests
  targeting `main`, and can be triggered manually.
