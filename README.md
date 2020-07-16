# Kubernetes Istio Bootstrap
Bootstrap repo to spin up a sample kubernetes cluster with Istio

## Create your Kubernetes Cluster

If you are not using a tool such as Google Kubernetes Engine (GKS) or AWS Elastic Kubernetes Service (EKS), then you can use a tool such as kops to provision your testing cluster.
 
 In production environments for all providers, at any scale you would use additional tooling such as infrastructure as code and deployment pipelines to manage the lifecycle of your clusters, but for demo'ing purposes invocation via the CLI is sufficient.

### Generate new cluster configuration
To create a cluster with some sensible defaults, please note the S3 bucket already exists in the AWS account we are attempting to use.

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

## Install Spinnaker

```bash
 curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/macos/InstallHalyard.sh
```

```bash
sudo bash InstallHalyard.sh
```

```bash
hal config provider kubernetes enable && \
CONTEXT=$(kubectl config current-context) && \
hal config provider kubernetes account add k8s-account --context $CONTEXT
```


```bash
hal version list
```

```bash
hal config version edit --version 1.21.0
```

```bash
hal config storage s3 edit \
    --access-key-id <AWS_ACCESS_KEY_ID> \
    --secret-access-key <AWS_SECRET_ACCESS_KEY> \
    --region eu-central-1
```

```bash
hal deploy apply
```


