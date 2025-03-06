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

## run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render resource local

```
 helm template -n forecastle --release-name forecastle --include-crds --skip-tests \
  -a cert-manager.io/v1 \
  -a networking.istio.io/v1beta1 \
  -f values-local.yaml --output-dir _local .
```
