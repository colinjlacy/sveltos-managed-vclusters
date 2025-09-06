# Sveltos vCluster Automation on EKS

This repo contains everything you'll need to test out automting the installation and management of vCluster instances on EKS, using Sveltos. All you need is an AWS account and a Kind cluster.

You can also opt to install the vClusters in a different host cluster, but just note that the K8s configs in this repo have only been tested on EKS. There may be subtle changes that you'll have to handle on your own.

## Standing up the K8s clusters

You'll need two clusters. I used Kind to create a management cluster, and EKS for my host cluster.

### Management Cluster in Kind

This step is pretty simple as long as you have [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/) installed:

```sh
kind create cluster --name management
```

This will add the `kind-management` cluster and context to your default Kubeconfig file, as well as set it as the current context.

From there, you can install Sveltos by following their installation instructions: https://projectsveltos.github.io/sveltos/getting_started/install/install/. If you prefer to use Helm (which is what I did), you can run:

```sh
helm repo add projectsveltos https://projectsveltos.github.io/helm-charts
helm repo update
helm install projectsveltos projectsveltos/projectsveltos -n projectsveltos --create-namespace --set agent.managementCluster=true
````

**Super Important:** Sveltos will automatically register the cluster it lives in as a managed cluster. But in order to make the vCluster registration work for the Sveltos event-handling resources in this repo, you'll nee to label the cluster properly. You can do that by running:

```sh
kubectl label sveltoscluster mgmt "sveltos-cluster=management" -n mgmt
```

### EKS Cluster

The files in `tofu-eks/` come from [this repo](https://github.com/hashicorp-education/learn-terraform-provision-eks-cluster), which has the MPL 2.0 license. For the most part the files are the same, however I'm using OpenTofu instead of Terraform.

Using the AWS CLI, configure your local profile to target an AWS account with the appropriate permissions needed to set up EKS, VPC, IAM, etc. If you're unsure, admin-level permissions work as long as you understand the risks of provisioning that much power to your user account.

With the local profile set, you can run:

```sh
cd tofu-eks
tofu init
tofu plan --out plan
tofu apply plan
```

That should stand up your EKS cluster. Just know that it can take a few minutes for everything to come online.

You can add the EKS cluster to your default Kubeconfig by running:

```sh
aws eks update-kubeconfig --region <region-code> --name <cluster-name>
```

This will use the active AWS profile to generate a token for authn/authz every time you run `kubectl` against the EKS context.

It will also set the EKS cluster as your current context.

### EKS Default Storage Class

**This is super important!!! If you don't do this, your vCluster pods will be stuck in a Pending state!!!**

One thing I noticed about these .tf files is that they fail to set the EBS storage class as the default. I'll admit, there's probably a configuration you can set very quickly to make that happen, but I haven't looked that up yet.

What I did instead was I added the right annotation to the StorageClass YAML that sets EBS as the default.

Make sure you're configured to target your EKS cluster, and run the following command to apply the change:

```sh
kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Register that EKS cluster in Sveltos

For this step, you'll need to have `svletosctl` installed. You can find instructions for that here: https://projectsveltos.github.io/sveltos/main/getting_started/sveltosctl/sveltosctl/.

You'll need to create a ServiceAccount and corresponding token Secret that Sveltos can use to access the EKS cluster's control plane API. The script in the root of this repo, `create-sa-token.sh`, will do this for you, as well as create a kubconfig file that can be used to access the EKS cluster with that service account's token. 

Make sure you're configured to target your EKS cluster, and run:

```sh
sh create-sa-token.sh
```

You'll see a file named `admin.config` created in the root of this repo. This is the kubeconfig file that you'll use to register the EKS cluster with Sveltos. 

Change your `kubectl` context back to `kind-management`, and run:

```sh
sveltosctl register cluster \
    --namespace=default \
    --cluster=eks-vcluster \
    --kubeconfig=./admin.config \
    --labels=environment=development,vcluster=host
```

The labels are important, as they will be used later to target this cluster for vCluster installation.

## EVENTS!!!

Now the fun part. Change your `kubectl` context back to `kind-management`, and run the following from the root of this repo:

```sh
# applies the event source for load balancer creation
kubectl apply -f sveltos-resources/lb
# applies the event source for exported secret creation
kubectl apply -f sveltos-resources/vcluster
```

These will create the necessary Sveltos resources to listen for events that will kick off the automated workflows.

## Did it work?

Change your `kubectl` context back to your EKS cluster, and run:

```sh
helm upgrade --install developer-load-balancers ./charts
```

That's everything you need to do to get the automation going. You can check the status of the vCluster installation by running:

```sh
kubectl get pods -A | grep vcluster
```

From here, you can label the clusters registered in Sveltos to apply different ClusterProfiles, which will in turn install different software. You can use the ClusterProfiles in `/sveltos-resources/profiles` as examples, or create your own.
