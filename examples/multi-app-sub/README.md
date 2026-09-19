# Example: two apps, one deployment

An overlay that creates two content checks, with different application_names:

- `https://avi.st` -> application `avi-st`
- `https://pics.avi.st` -> application `pics`

This was exactly the setup that initially caused me to write this, but in working this out I realised I didn't want it after all.

You can have it though!

In this config both have a subsystem of 'synthetics' but you can override that, too.

Each input is tagged with a `cx_route` tag, and each `outputs.opentelemetry` block filters on that tag with `tagpass`

See [Deployment scenarios](../../deployment-scenarios.md) for some background here and why this is better than
running multiple deployments

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
> http_response,cx_route=avi-st,...,server=https://avi.st,... response_status_code_match=1i,...
> http_response,cx_route=pics,...,server=https://pics.avi.st,... response_status_code_match=1i,...
```

And then in Coralogix you'll get avi.st metrics with app/sub of `avi-st/synthetics` and pics.avi.st metrics `pics/synthetics`
