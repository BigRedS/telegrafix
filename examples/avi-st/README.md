# A simple example: probe avi.st

An overlay that configures two content checks:

- `https://avi.st` must return 200 and contain `cookie.jpg`
- `https://pics.avi.st` must return 200 and contain `Recent photos`

These will both appear under the same app/sub names, so we only need `telegraf.conf`
I deploy this into the `coralogix` ns alongside the coralogix k8s instrumentation, so the send-your-data key is read from the `coralogix-keys` (

## Deploy

```bash
kubectl kustomize . | kubectl apply -f -
kubectl rollout restart deployment/telegrafix
```

## Test the tests locally

Render the configmaps, extract the config files and give them to a running telegraf container:

```bash
kubectl kustomize . > rendered.yaml
yq -r 'select(.kind == "ConfigMap") | .data["telegraf.conf"]' rendered.yaml > telegraf.conf
yq -r 'select(.kind == "ConfigMap") | .data["output.conf"]' rendered.yaml > output.conf

podman run --rm --entrypoint telegraf \
  -v "$PWD/telegraf.conf:/etc/telegraf/telegraf.conf:ro" \
  -v "$PWD/output.conf:/etc/telegraf/output.conf:ro" \
  docker.io/library/telegraf:1.40-alpine \
  --config /etc/telegraf/telegraf.conf --config /etc/telegraf/output.conf --test
```
You expect two lines of output here (one per check):
```
> http_response,...,result=success,server=https://avi.st,... response_string_match=1i,...
> http_response,...,result=success,server=https://pics.avi.st,... response_string_match=1i,...
```

The telegraf image wants to start as root and drop to the telegraf user, which fails under
rootless podman; `--entrypoint telegraf` works around this.

In the cluster the metrics arrive in Coralogix with app/sub of `telegrafix/synthetics`
(or whatever you have set `CORALOGIX_APPLICATION` and `CORALOGIX_SUBSYSTEM` to),
and the 'server' label is set to the URL of the check.
