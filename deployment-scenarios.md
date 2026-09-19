# Deployment scenarios

This began with the idea of solving a specific problem of mine - create a series of synthetics checks with application and subsystem names set
per-site.

I've since changed my mind and now run the simple option: all synthetics checks have the same application and subsystem name, and are
distinguished by their labels (e.g. the `server` label that `http_response` sets). The Coralogix Synthetic Monitoring Extension assumes this
shared app/sub setup.

The only-slightly-more complex scenarios are still possible; each Telegraf instance (and so each Telegrafix instance) can have multiple `[[outputs.opentelemetry]]` blocks, so a single deployment can still send different checks to different `application`/`subsystem` names.

| Situation | What to do |
|---        |---         |
| CX OTel distro already in the namespace | Patch CORALOGIX_DOMAIN in the Deployment's `env` if you're not in EU2 (see [Overriding the key or domain](#overriding-the-key-or-domain)), and the `coralogix-keys` secret will be picked up automatically |
| CX OTel distro present, but want to override key and/or domain | Patch the Deployment's `env` (see [Overriding the key or domain](#overriding-the-key-or-domain)); entries merge by name so override one or both. |
| Empty namespace, no CX config at all | Create your own `coralogix-keys` secret and patch `CORALOGIX_DOMAIN` in the Deployment's `env`. |
| Different checks need different app/sub names | Add more `[[outputs.opentelemetry]]` blocks routed with `tagpass` (see [Application / subsystem names](#application--subsystem-names)). |
| Several telegrafix deployments in one namespace anyway (isolation, not just naming) | Set `namePrefix`/`nameSuffix` in your overlay and give each instance a unique selector label, e.g. `labels: [{pairs: {instance: <name>}, includeSelectors: true}]` — otherwise every instance keeps the same `app: telegrafix` selector and Deployments fight over each other's pods. |

## Overriding the key or domain

Patch the Deployment's `env`; entries merge by name, so include only the ones you want to change:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: telegrafix
spec:
  template:
    spec:
      containers:
        - name: telegraf
          env:
            - name: CORALOGIX_DOMAIN
              value: us1.coralogix.com
            - name: CORALOGIX_PRIVATE_KEY
              value: <send-your-data-api-key>
              # Without this the base's secretKeyRef is merged in alongside `value`, and the API rejects the Deployment
              valueFrom: null
```

## Application / subsystem names

There's a few options for application and subsystem names:

1. **Same app/sub names for everything** (default)
   Everything goes under  `telegrafix/synthetics` and you tell which is which using input tags, which are set as metric labels:

   ```toml
   [[inputs.http_response]]
     urls = ["https://avi.st"]
     [inputs.http_response.tags]
       check = "avi.st"
   ```

   This is the pattern the Synthetic Monitoring extension in Coralogix expects

2. **Different app/sub for different probes (pings, http_responses etc.)**
   Replace the ConfigMap's `output.conf` key with multiple `[[outputs.opentelemetry]]` blocks,
   each with its own `namepass` filter (matches on measurement name, e.g. `http_response` vs `ping`) and coralogix names.
   Every block must have a coralogix section, and each output gets a copy of the filtered metrics only — use `namepass` on every block so nothing is sent twice.

   ```yaml
   # patch to the ConfigMap, alongside the telegraf.conf key
   data:
     output.conf: |
       [[outputs.opentelemetry]]
         namepass = ["http_response*"]
         service_address = "https://ingress.${CORALOGIX_DOMAIN:-eu2.coralogix.com}:443/v1/metrics"
         [outputs.opentelemetry.coralogix]
           private_key = "${CORALOGIX_PRIVATE_KEY:-dummy}"
           application = "synthetics"
           subsystem = "web"
       [[outputs.opentelemetry]]
         namepass = ["ping*"]
         service_address = "https://ingress.${CORALOGIX_DOMAIN:-eu2.coralogix.com}:443/v1/metrics"
         [outputs.opentelemetry.coralogix]
           private_key = "${CORALOGIX_PRIVATE_KEY:-dummy}"
           application = "synthetics"
           subsystem = "icmp"
   ```

3. **Split by site/check**
   Perhaps an app-name for each monitored site; tag each input with a routing tag and filter each output block with `tagpass` on that tag:

   ```toml
   # telegraf.conf
   [[inputs.http_response]]
     urls = ["https://avi.st"]
     [inputs.http_response.tags]
       cx_route = "avi-st"

   [[inputs.http_response]]
     urls = ["https://example.org"]
     [inputs.http_response.tags]
       cx_route = "example-org"
   ```

   And patch the ConfigMap, essentially duplicating the defaults with the additional routing based on tags:

   ```yaml
   data:
     output.conf: |
       [[outputs.opentelemetry]]
         service_address = "https://ingress.${CORALOGIX_DOMAIN:-eu2.coralogix.com}:443/v1/metrics"
         [outputs.opentelemetry.tagpass]
           cx_route = ["avi-st"]
         [outputs.opentelemetry.coralogix]
           private_key = "${CORALOGIX_PRIVATE_KEY:-dummy}"
           application = "avi-st"
           subsystem = "synthetics"
       [[outputs.opentelemetry]]
         service_address = "https://ingress.${CORALOGIX_DOMAIN:-eu2.coralogix.com}:443/v1/metrics"
         [outputs.opentelemetry.tagpass]
           cx_route = ["example-org"]
         [outputs.opentelemetry.coralogix]
           private_key = "${CORALOGIX_PRIVATE_KEY:-dummy}"
           application = "example-org"
           subsystem = "synthetics"
   ```

   As with option 2, every output block filters independently so you need to carefully give each input exactly one matching route else you'll (silently) drop metrics that have no route or duplicate metrics that have more than one.
