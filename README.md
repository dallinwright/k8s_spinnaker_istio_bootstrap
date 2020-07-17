# Kubernetes Spinnaker Istio - Bootstrap
Bootstrap repo to spin up a sample kubernetes cluster with Istio

## Create your Kubernetes Cluster

#### Disclaimer
If you are not using a tool such as Google Kubernetes Engine (GKS) or AWS Elastic Kubernetes Service (EKS), then you can use a tool such as kops to provision your testing cluster. I would heavily recommend against rolling your own cluster, as management over time can become complex and time consuming.
 
In production environments for all providers, at any scale you would use additional tooling such as infrastructure as code and deployment pipelines to manage the lifecycle of your clusters, but for demo'ing purposes invocation via the CLI is sufficient. so without further ado, let's get started.

#### Generate new cluster configuration
To create a KOPS cluster with some sensible defaults, please note this assumes AWS usage, and an S3 bucket already exists in the AWS account we are attempting to use. You also need to have AWS access keys in place, with permissions according to the kops documentation.

#### Export state store
Before anything, export the state store to an env var. This will reduce the copy + paste a lot.

```bash
export KOPS_STATE_STORE=s3://yourstatestore
```

Now, create the cluster config.

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
--name cluster.cluster.test-k8s.<domain>.<suffix> \
--ssh-public-key ~/.ssh/id_rsa.pub
```

This configuration sets up a bastion host (an ssh jump host) along with a dedicated VPC, route53 dns entries, load balancers, etc. I would recommend reading up on the kops documentation if interested and to fully understand how it works.

####  Why Weave?

Weave is just an example and one I would recommend for sensitive workloads on powerful machines. It offers one thing that most other CNI's do not and that is network encryption. This does add overhead to network requests and slows down signifigantly communication when compared to layer 3/4 CNI's and if you handle encryption another way, or do not need intercluster encryption Calico is what I would recommend, as it is used internally by Google and very fast.

#### Apply Configuration
The previous command only outputs the resources it will create but does not apply it. If it looks good, apply it via the following command.
```bash
kops update cluster --name cluster.test-k8s.<domain>.<suffix> --yes
```

#### Validate Cluster
```bash
kops validae cluster --name cluster.test-k8s.<domain>.<suffix>
```


## Install Istio service mesh

Istio is a service mesh that enables complex update capabilities giving you much more seamless and fine-grained control. You no longer need to manage a large set of selectors, keep proxy information up to date, keep reverse proxy information up to date, etc. among numerous other benefits. I would heavily recommend one, and Istio as it is the industry leader in this regard and cloud agnostic.

#### Generate Istio Config
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

#### Apply Istio Config
```bash

```



#### Learn more about Istio

```bash
git clone git@github.com:istio/istio.git
```

Then follow the tutorials here
https://istio.io/latest/docs/setup/getting-started/#downloistioad

## Install Spinnaker

Spinnaker is a Continuous Delivery (CD) tool that was spun out of Netflix that integrates natively with Kunernetes and offers a large range of integration and feature support. It has a wide range on configuration options, widely adopted, and quickly becoming the preferred CD tool for operation teams to deploy artifacts when new version are published to an artifact repository, into Kubernetes.

The command can be either run from your local machine, or the bastion k8s node. In production or enterprise environments it will be the bastion.


#### Ensure `.kube` directory exists
##### Note: Ensure the config file exists as well in `~/.kube/config`
```bash
mkdir ~/.kube
```

#### Ensure `.hal` directory exists
Ensure the /root/.hal directory exists so the container can store and and persist the Hal config
```bash
mkdir /root/.hal && chmod 0766 /root/.hal
```


#### Start Halyard Container
This command starts and mounts the `.hal` and `.kube` directories to persist configs between restarts.
```bash
sudo docker run --name halyard -v ~/.hal:/home/spinnaker/.hal -v ~/.kube/config:/home/spinnaker/.kube/config -d gcr.io/spinnaker-marketplace/halyard:stable
```

#### SSH into Hal Container
```bash
sudo docker exec -it halyard bash
```

#### Create Hal account for k8s and your cluster
```bash
hal config provider kubernetes enable && \
CONTEXT=$(kubectl config current-context) && \
hal config provider kubernetes account add k8s --context $CONTEXT
```

#### Set Kubernetes as the cloud provider
```
hal config provider kubernetes enable
```

#### Sets where to install Spinnaker to your k8s cluster
```
hal config deploy edit --type distributed --account-name k8s
```

#### List available versions of Spinnaker
```bash
hal version list
```

#### Select your Spinnaker version
Ensure you use a version >= 1.20.0 otherwise additional configuration will be needed.
```bash
hal config version edit --version 1.21.0
```

#### Create an S3 bucket via Hal
This command will create you a bucket in the region you specify to store spinnaker build and deployment information. This are json files that container build related configuration. If you use AWS, you can use S3, otherwise Spinnaker recommends MinIO as the storate backend as it uses Kubernetes native object storage.
```bash
hal config storage s3 edit \
    --access-key-id $ACCESS_KEY_ID \
    --secret-access-key \
    --region $REGION;
```

#### Set Hal to use S3
```bash
hal config storage edit --type s3
```

#### Deploy Spinnaker and all components
```bash
hal deploy apply
```

## If you wish to not use S3, use MinIO
#### Install Helm on local machine
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
#### Add Helm stable repo
```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```
#### Install minio in kubernetes cluster
```bash
kubectl create ns spinnaker && \
helm install minio --namespace spinnaker --set accessKey="minioadmin" --set secretKey="minioadmin" --set persistence.enabled=false stable/minio
```

### Back inside the halyard container
#### For minio, disable s3 versioning
```bash
mkdir ~/.hal/default/profiles
echo "spinnaker.s3.versioning: false" > ~/.hal/default/profiles/front50-local.yml
```
#### Set the storage type to minio/s3
```bash
hal config storage s3 edit --endpoint http://minio:9000 --access-key-id "minioadmin" --secret-access-key "minioadmin"
hal config storage s3 edit --path-style-access true
hal config storage edit --type s3
```

#### Change the service type to either Load Balancer or NodePort
```bash
kubectl -n spinnaker edit svc spin-deck
kubectl -n spinnaker edit svc spin-gate
```

#### Update config and redeploy
```bash
hal config security ui edit --override-base-url "http://<LoadBalancerIP>:9000"
hal config security api edit --override-base-url "http://<LoadBalancerIP>:8084"
hal deploy apply
```
*If you used NodePort*
```

hal config security ui edit --override-base-url "http://<worker-node-ip>:<nodePort>"
hal config security api edit --override-base-url "http://worker-node-ip>:<nodePort>"
hal deploy apply
```
