# Kubernetes Istio Bootstrap
Bootstrap repo to spin up a sample kubernetes cluster with Istio

### Generate new cluster configuration
To create a cluster with some sensible defaults.

```bash
kops create cluster \
--cloud=aws \
--dns private \
--dns-zone <hosted_zone_id> \
--node-count 3 \
--zones=eu-west-1a \
--node-size t3.medium \
--master-size t3.small \
--networking weave \
--topology private \
--bastion="true" \
--state s3://istio-kops-state \
--name cluster.istio-kops.<domain>.<suffix> \
--ssh-public-key ~/.ssh/id_rsa.pub
```

#####  Why Weave?

### Apply Configuration
```bash
kops update cluster --name cluster.istio-kops.<domain>.<suffix> --yes --state s3://istio-kops-state
```


### Generate Istio Config
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


### If demoing, and needing a sample app
```bash
git clone git@github.com:istio/istio.git
```

https://istio.io/latest/docs/setup/getting-started/#downloistioad
