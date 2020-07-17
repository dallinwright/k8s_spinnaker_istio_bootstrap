# Istio service mesh

Istio is a service mesh that enables complex update capabilities giving you much more seamless and fine-grained control. You no longer need to manage a large set of selectors, keep proxy information up to date, keep reverse proxy information up to date, etc. among numerous other benefits. I would heavily recommend one, and Istio as it is the industry leader in this regard and cloud agnostic.

Istio can be installed and managed via helm, but support has been depreciated and being dropped in favor of `istioctl`.

### Generate Istio Config
Here is a sample Istio config you can generate and use.
```bash
istioctl manifest generate \
--set profile=default \
--set values.gateways.enabled=true \
--set values.gateways.istio-ingressgateway.enabled=true \
--set values.gateways.istio-ingressgateway.sds.enabled=true \
--set values.grafana.enabled=true \
--set values.prometheus.enabled=true \
--set values.tracing.enabled=true \
--set values.kiali.enabled=true \
--set values.tracing.enabled=true > generated-manifest.yaml
```

### Apply Istio Config
Inspect the files as needed, then apply.
```bash
 kubectl apply -f ./generated-manifest.yaml
```

After these steps, you will have a working base istio install! You will need to dive a bit and learn the various dashboards and how they work.

One this to not forget and use, is allow Istio to manage and automatically inject an envoy proxy into each pod via the namespace label like so.

```bash
kubectl label namespace default istio-injection=enabled
```

#### Learn more about Istio

```bash
git clone git@github.com:istio/istio.git
```

Then follow the tutorials here, in particular the tutorials regarding updates and rollbacks are very good.
https://istio.io/latest/docs/setup/getting-started/#downloistioad