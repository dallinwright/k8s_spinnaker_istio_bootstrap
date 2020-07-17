# Kubernetes Cluster Creation

This is a sample of how to setup and provides the foundation for your very own Kubernetes cluster. The process of using KOPS in this tutorial is not much different then managing a long lived cluster and provides a good baseline to get started.

### Disclaimer
In my opinion, you should use industry leading Kubernetes management from [Google Kubernetes Engine (GKS)](https://cloud.google.com/kubernetes-engine) or [AWS Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/), then use a tool such as  [tool such as Kops](https://github.com/kubernetes/kops) to provision your testing cluster as a last resort. I would heavily recommend against rolling your own cluster, as management over time can become complex and time-consuming (in large companies they have whole teams dedicated to managing and running the clusters, it is a full time job.) in the end being generally not worth it unless you have a very specific use case that warrants it.
 
In production environments for all providers, at any scale you would use additional tooling such as infrastructure as code and deployment pipelines to manage the lifecycle of your clusters, but for demo'ing purposes invocation via the CLI is sufficient, so without further ado, let's get started.

### Generate new cluster configuration
To create a KOPS cluster with some sensible defaults, please note this assumes AWS usage, and an S3 bucket already exists in the AWS account we are attempting to use. You can use GKS, EKS, or even minikube with minimal adjustments. For this, you also need to have AWS access keys in place, with permissions according to the KOPS documentation.

### Export state store
Before anything, export the state store to an env var. This will reduce the copy + paste a lot. This is a bucket that should be created via the UI or some other mechanism.

```bash
export KOPS_STATE_STORE=s3://yourstatestore-custom-bucket
```

### Generate the cluster config
This generates the config, feel free to adjust sizes, etc. Make sure you have an RSA ssh key and the various options understood.

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

This configuration sets up a bastion host (a ssh jump host) along with a dedicated VPC, route53 dns entries, load balancers, etc. I would recommend reading up on the KOPS documentation if interested and to fully understand how it works.

####  Why Weave?

Weave is just an example and one I would recommend for sensitive workloads on powerful machines. It offers one thing that most other CNI's do not and that is network encryption. This does add overhead to network requests and slows down significantly communication when compared to layer 3/4 CNI's and if you handle encryption another way, or do not need inter-cluster encryption Calico is what I would recommend, as it is used internally by Google and very fast.

### Apply Configuration
The previous command only outputs the resources it will create but does not apply it. If it looks good, apply it via the following command.
```bash
kops update cluster --name cluster.test-k8s.<domain>.<suffix> --yes
```

### Validate Cluster
Please note, this can take 5+ minutes to register as healthy. You will receive 5xx errors before if you run it and it is not yet ready.
```bash
kops validae cluster --name cluster.test-k8s.<domain>.<suffix>
```

## Congratulations!

That's all there is to it, the provisioning of the cluster is the easy part once you understand the pieces. In the real world, you have these options and commands fully automated via what is called 'GitOps', where on a merge to the master branch of a git repository your changes are automatically applied after being linted (code styling checks) and tested.