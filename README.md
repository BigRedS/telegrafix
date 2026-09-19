# telegrafix

<a href="https://asterix.com/en/characters/pearlclutcha-and-oldogoltrix/"><img src="assets/telegrafix-and-analogine.png" width="300" alt="telegrafix and Analogine"></a>

A lighweight Kustomize base that runs [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/) for simple synthetic monitoring in [Coralogix](https://coralogix.com).

Simple in the SMTP sense - it doesn't do very much, and doesn't need a lot to do that;
It creates a ConfigMap with the `telegraf.conf` in it and a Deployment running Telegraf;
it'll read the secret you created for the Coralogix k8s instrumentation if it can see it.

The real-life configuration on my homelab cluster is here: https://github.com/BigRedS/duloc/tree/main/telegrafix

## Config

This is intentionally a small kustomize config. You'll need two files:

`kustomization.yaml` to set the thing up. Importantly here, the `namespace` is the one you've got your Coralogix instrumentation in,
so it can re-use the secret from there.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: coralogix

resources:
  - https://github.com/BigRedS/telegrafix?ref=v1.0.0

patches:
  - path: telegraf-config.yaml
    target:
      version: v1
      kind: ConfigMap
      name: telegraf-config
  # If you're not in EU2 you will need to specify a domain. If you are, you don't need this
  - target:
      kind: Deployment
      name: telegrafix
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value: {name: CORALOGIX_DOMAIN, value: us1.coralogix.com}
```

`telegraf-config.yaml` contains the agent and input config for telegraf. We get `outputs` for free:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-config
data:
  telegraf.conf: |
    [agent]
      interval = "31s"
      flush_interval = "11s"

    [[inputs.http_response]]
      urls = ["https://example.com"]
      response_status_code = 200
      response_timeout = "6s"
      response_string_match = "Example Domain"

    [[inputs.http_response]]
      urls = ["https://example.net"]
      response_status_code = 200
      response_timeout = "6s"
      response_string_match = "Avoid use in operations"
```

Then apply this and you should have some checks:

```bash
kubectl apply -k .
```

Note that when you make config changes here, you'll need to manually roll the deployment:

```bash
kubectl apply -k . && kubectl -n coralogix rollout restart deployment/telegrafix
```

One thing to bear in mind:

* It reads the `coralogix-keys` secret to get the api-key.
  If you want to install this in a namespace other than the one with your CX distro, create the secret
  (note that this will come up and run with a bad secret, it'll just not-work and error a lot):

```bash
kubectl -n <your-namespace> create secret generic coralogix-keys --from-literal=PRIVATE_KEY=<send-your-data-api-key>
```

For more complex options, see [deployment-scenarios.md](deployment-scenarios.md); for complete examples see `examples/`

## Env vars

There are four env vars, with defaults:

| Variable | Default |
|---|---|
| `CORALOGIX_DOMAIN` | `eu2.coralogix.com` |
| `CORALOGIX_PRIVATE_KEY` | `dummy` |
| `CORALOGIX_APPLICATION` | `telegrafix` |
| `CORALOGIX_SUBSYSTEM` | `synthetics` |

You can override any of these using the same pattern as for `CORALOGIX_DOMAIN`, see  [deployment-scenarios.md](/deployment-scenarios.md) for more detail.

## Potential Beartraps

* ***No auto-reload***; you need to manually roll the pods after reconfiguring, Telegraf doesn't notice the change. Apply with  `kubectl apply -k . && kubectl rollout -n coralogix restart deployment telegrafix`
* ***Kubectl apply -k is being deprecated*** and Flux doesn't support remote bases by default (set `KUSTOMIZE_ALLOW_REMOTE_BASES=true`). I will come up with a workaround when this bites me.
* ***Ping only works because the container starts as root***, grants `cap_net_raw` to the binary and drops to the `telegraf` user. On any sane production infrastructure you will set `runAsNonRoot` and this won't work; maybe switch it to mode = `exec` and permit unprivileged ICMP or just permit root here?
* This repo doesn't specify Telegraf versions, your Kustomize overlay does

## Adding probes

Everything runs off the one `telegraf.conf` key. Add any Telegraf inputs blocks you need (`http_response`, `ping`, `net_response`, `dns_query`, `tcp_response`, `x509_cert`), see the Telegraf config for details on those:  <https://docs.influxdata.com/telegraf/v1/plugins/>.

A default output is configured for Coralogix, there's no need to add any unless you want to send different probes out differently (different application and subsytem names names etc.).
