# forecastle-controller

Repository containing manifests for [forecastle](https://github.com/stakater/Forecastle).
Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart pulls in `forecastle` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

    $ helm dependency update

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## Render resource local

### local

```
 helm template -n forecastle --release-name forecastle --include-crds --skip-tests \
  -a cert-manager.io/v1 \
  -a networking.istio.io/v1beta1/VirtualService \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  --output-dir _render_output/local .
```

### dev

```
 helm template -n forecastle --release-name forecastle --include-crds --skip-tests \
  -a cert-manager.io/v1 \
  -a networking.istio.io/v1beta1/VirtualService \
  -f values-subchart-overrides.yaml \
  -f values-development.yaml \
  --output-dir _render_output/dev .
```

### prod

```
 helm template -n forecastle --release-name forecastle --include-crds --skip-tests \
  -a cert-manager.io/v1 \
  -a networking.istio.io/v1beta1/VirtualService \
  -f values-subchart-overrides.yaml \
  -f values-production.yaml \
  --output-dir _render_output/prod .
```

## Testing

### Usage of values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

### Run helm unittests

```shell
 helm dependency update && \
 docker run -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 helm dependency update && \
 docker run -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```