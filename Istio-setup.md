# Istio service mesh

Istio is a service mesh that enables complex update capabilities giving you much more seamless and fine-grained control. You no longer need to manage a large set of selectors, keep proxy information up to date, keep reverse proxy information up to date, etc. among numerous other benefits. I would heavily recommend one, and Istio as it is the industry leader in this regard and cloud agnostic.

I have used it personally and seen it in action and could not image Kubernetes used without a service mesh. It takes all the logic, programming, debugging, etc. out of application code and lifts it up into the service mesh giving you simple to complex configuration options out of the box and configured with YAML. Whats not to love?


Istio can be installed and managed via helm, but support has been depreciated and being dropped in favor of `istioctl`, so keep this in mind going forward. You can (and should) use GitOps to apply and manage these commands and files. Istioctl can be used to generate manifest files for all components that can be checks into version control and applied, so you have a log of every single change and can revert to any point in history at the push of the button.


### Generate Istio Config
Here is a sample Istio config you can generate and use, note this enables Grafana, which is a webUI used to help chart and monitor your services, prometheus which is used for monitoring/alerting and slack integrations (and much more), kiali to help you visualize in a chart your service mesh, and distributed tracing to view requests as they move around your cluster. There are many more options but this is just a baseline. For example, I typically use datadog which is a paid software as a service cloud provider to do all this for me, but this is the "free" option, free as in you don't pay to use it, but you pay by doing the heavy lifting.

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

Please note, if you followed the kops tutorial I provided in another article, [follows these instructions](https://istio.io/latest/docs/setup/platform-setup/kops/) on enabling the istio secret discovery service. Essentially it allows you to mount secrets to your services from an external secret store.

### Apply Istio Config
Inspect the files as needed, then apply.
```bash
 kubectl apply -f ./generated-manifest.yaml
```

If you wish you can simply run `istioctl install` to install Istio into your cluster in a default configuration. Usually this is much easier, but an anti-pattern as you don't have the configuration to view/edit and roll back to should your cluster experience issues.

After these steps, you will have a working base Istio install! You will need to dive a bit and learn the various dashboards and how they work.

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
