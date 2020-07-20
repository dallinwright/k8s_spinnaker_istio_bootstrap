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

Here is an example of what the output would look like for the command above.

```bash
I0720 09:55:20.306916    9733 dns.go:94] Private DNS: skipping DNS validation
I0720 09:55:21.515355    9733 executor.go:103] Tasks: 0 done / 122 total; 52 can run
I0720 09:55:22.208372    9733 vfs_castore.go:728] Issuing new certificate: "apiserver-aggregator-ca"
I0720 09:55:22.213514    9733 vfs_castore.go:728] Issuing new certificate: "etcd-peers-ca-main"
I0720 09:55:22.285300    9733 vfs_castore.go:728] Issuing new certificate: "ca"
I0720 09:55:22.289450    9733 vfs_castore.go:728] Issuing new certificate: "etcd-clients-ca"
I0720 09:55:22.323851    9733 vfs_castore.go:728] Issuing new certificate: "etcd-peers-ca-events"
I0720 09:55:22.329399    9733 vfs_castore.go:728] Issuing new certificate: "etcd-manager-ca-main"
I0720 09:55:22.424402    9733 vfs_castore.go:728] Issuing new certificate: "etcd-manager-ca-events"
I0720 09:55:27.584256    9733 executor.go:103] Tasks: 52 done / 122 total; 29 can run
I0720 09:55:28.823646    9733 vfs_castore.go:728] Issuing new certificate: "apiserver-aggregator"
I0720 09:55:28.859247    9733 vfs_castore.go:728] Issuing new certificate: "kube-proxy"
I0720 09:55:28.867684    9733 vfs_castore.go:728] Issuing new certificate: "kubelet-api"
I0720 09:55:28.869307    9733 vfs_castore.go:728] Issuing new certificate: "kops"
I0720 09:55:28.885868    9733 vfs_castore.go:728] Issuing new certificate: "apiserver-proxy-client"
I0720 09:55:28.955031    9733 vfs_castore.go:728] Issuing new certificate: "kubelet"
I0720 09:55:28.964529    9733 vfs_castore.go:728] Issuing new certificate: "kube-scheduler"
I0720 09:55:28.986712    9733 vfs_castore.go:728] Issuing new certificate: "kube-controller-manager"
I0720 09:55:29.067008    9733 vfs_castore.go:728] Issuing new certificate: "kubecfg"
I0720 09:55:34.242768    9733 executor.go:103] Tasks: 81 done / 122 total; 31 can run
I0720 09:55:35.639973    9733 executor.go:103] Tasks: 112 done / 122 total; 7 can run
I0720 09:55:36.372733    9733 vfs_castore.go:728] Issuing new certificate: "master"
I0720 09:55:38.469348    9733 executor.go:103] Tasks: 119 done / 122 total; 3 can run
I0720 09:55:38.545399    9733 natgateway.go:286] Waiting for NAT Gateway "nat-xxxxxxxxxxxxxxxxxxxxx" to be available (this often takes about 5 minutes)
I0720 09:57:24.562449    9733 executor.go:103] Tasks: 122 done / 122 total; 0 can run
I0720 09:57:24.562487    9733 dns.go:155] Pre-creating DNS records
I0720 09:57:27.342671    9733 update_cluster.go:305] Exporting kubecfg for cluster

```

####  Why Weave?

Weave is just an example and one I would recommend for sensitive workloads on powerful machines. It offers one thing that most other CNI's do not and that is network encryption. This does add overhead to network requests and slows down significantly communication when compared to layer 3/4 CNI's and if you handle encryption another way, or do not need inter-cluster encryption Calico is what I would recommend, as it is used internally by Google and very fast.

### Apply Configuration
The previous command only outputs the resources it will create but does not apply it. If it looks good, apply it via the following command.
```bash
kops update cluster --name cluster.test-k8s.<domain>.<suffix> --yes
```

### Validate Cluster
Wait between five and ten minutes as the cluster comes up and initializes. You can recieve various errors and 500 codes until it is ready but do not worry too much. After some time (depends on cluster node size among other things) you can run this command to validate the cluster.

```bash
$ kops validate cluster
```

kops uses what is called the "context" from kubectl to show you information for your nodes. You should receive output similar to the following if you followed the command above during creation.

```bash
Using cluster from kubectl context: cluster.test-k8s.<domain>.<suffix>

Validating cluster cluster.test-k8s.<domain>.<suffix>

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
bastions		Bastion	t3.micro	1	1	utility-eu-west-1a
master-eu-west-1a	Master	t3.small	1	1	eu-west-1a
nodes			Node	t3.medium	3	3	eu-west-1a

NODE STATUS
NAME						ROLE	READY
ip-172-20-45-151.eu-west-1.compute.internal	node	True
ip-172-20-47-209.eu-west-1.compute.internal	node	True
ip-172-20-51-93.eu-west-1.compute.internal	node	True
ip-172-20-52-201.eu-west-1.compute.internal	master	True

Your cluster cluster.test-k8s.<domain>.<suffix> is ready

```

## Congratulations!
That's all there is to it, the provisioning of the cluster is the easy part once you understand the pieces. In the real world, you have these options and commands fully automated via what is called 'GitOps', where on a merge to the master branch of a git repository your changes are applied automatically after being linted (code styling checks) and tested.
