
---
title: "Getting Started with Karpenter"
linkTitle: "Getting Started with Karpenter"
weight: 10
description: >
  Set up a cluster and add Karpenter
---

Karpenter automatically provisions new nodes in response to unschedulable pods. Karpenter does this by observing events within the Kubernetes cluster, and then sending commands to the underlying cloud provider.

This guide shows how to get started with Karpenter by creating a Kubernetes cluster and installing Karpenter.
To use Karpenter, you must be running a supported Kubernetes cluster on a supported cloud provider.

## Create a cluster and add Karpenter

It should take less than 1 hour to complete, and cost less than $0.25.
Follow the clean-up instructions to reduce any charges.

### 1. Install utilities

Karpenter is installed in clusters with a Helm chart.

Install these tools before proceeding:

1. [AZ CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html)
2. `kubectl` - [the Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
3. `helm` - [the package manager for Kubernetes](https://helm.sh/docs/intro/install/)

### 2. Set environment variables

After setting up the tools, set the Karpenter and Kubernetes version:

```bash
export KARPENTER_VERSION=v0.32.1
export K8S_VERSION={{< param "latest_k8s_version" >}}
```

Then set the following environment variable:

{{% script file="./content/scripts/getting-started/step01-config.sh" language="bash"%}}

{{% alert title="Warning" color="warning" %}}
If you open a new shell to run steps in this procedure, you need to set some or all of the environment variables again.
To remind yourself of these values, type:

```bash
echo $KARPENTER_VERSION $K8S_VERSION $CLUSTER_NAME $DEFAULT_REGION $TEMPOUT
```

{{% /alert %}}


### 3. Create a Cluster


* Run helm to install karpenter

{{% script file="./content/scripts/getting-started/step02-create-cluster.sh" language="bash"%}}

{{% script file="./content/scripts/getting-started/step06-add-spot-role.sh" language="bash"%}}


### 4. Install Karpenter

{{% script file="./content/scripts/getting-started/step08-apply-helm-chart.sh" language="bash"%}}

{{% alert title="Warning" color="warning" %}}
Karpenter creates a mapping between CloudProvider machines and CustomResources in the cluster for capacity tracking. To ensure this mapping is consistent, Karpenter utilizes the following tag keys:

* `karpenter.sh/managed-by`
* `karpenter.sh/nodepool`
* `kubernetes.io/cluster/${CLUSTER_NAME}`

Because Karpenter takes this dependency, any user that has the ability to Create/Delete these tags on CloudProvider machines will have the ability to orchestrate Karpenter to Create/Delete CloudProvider machines as a side effect. We recommend that you [enforce tag-based IAM policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_tags.html) on these tags against any EC2 instance resource (`i-*`) for any users that might have [CreateTags](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_CreateTags.html)/[DeleteTags](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DeleteTags.html) permissions but should not have [RunInstances](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_RunInstances.html)/[TerminateInstances](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_TerminateInstances.html) permissions.
{{% /alert %}}

### 5. Create NodePool

A single Karpenter NodePool is capable of handling many different pod shapes. Karpenter makes scheduling and provisioning decisions based on pod attributes such as labels and affinity. In other words, Karpenter eliminates the need to manage many different node groups.

Create a default NodePool using the command below. This NodePool uses `securityGroupSelectorTerms` and `subnetSelectorTerms` to discover resources used to launch nodes. We applied the tag `karpenter.sh/discovery` in the `eksctl` command above. Depending on how these resources are shared between clusters, you may need to use different tagging schemes.

The `consolidationPolicy` set to `WhenUnderutilized` in the `disruption` block configures Karpenter to reduce cluster cost by removing and replacing nodes. As a result, consolidation will terminate any empty nodes on the cluster. This behavior can be disabled by setting `consolidateAfter` to `Never`, telling Karpenter that it should never consolidate nodes. 

Note: This NodePool will create capacity as long as the sum of all created capacity is less than the specified limit.

{{% script file="./content/scripts/getting-started/step12-add-nodepool.sh" language="bash"%}}

Karpenter is now active and ready to begin provisioning nodes.

## First Use

Create some pods using a deployment and watch Karpenter provision nodes in response.

### Scale up deployment

This deployment uses the [pause image](https://www.ianlewis.org/en/almighty-pause-container) and starts with zero replicas.

{{% script file="./content/scripts/getting-started/step13-automatic-node-provisioning.sh" language="bash"%}}

### Scale down deployment

Now, delete the deployment. After a short amount of time, Karpenter should terminate the empty nodes due to consolidation.

{{% script file="./content/scripts/getting-started/step14-deprovisioning.sh" language="bash"%}}

## Add optional monitoring with Grafana

This section describes optional ways to configure Karpenter to enhance its capabilities.
In particular, the following commands deploy a Prometheus and Grafana stack that is suitable for this guide but does not include persistent storage or other configurations that would be necessary for monitoring a production deployment of Karpenter.
This deployment includes two Karpenter dashboards that are automatically onboarded to Grafana. They provide a variety of visualization examples on Karpenter metrics.

{{% script file="./content/scripts/getting-started/step09-add-prometheus-grafana.sh" language="bash"%}}

The Grafana instance may be accessed using port forwarding.

{{% script file="./content/scripts/getting-started/step10-add-grafana-port-forward.sh" language="bash"%}}

The new stack has only one user, `admin`, and the password is stored in a secret. The following command will retrieve the password.

{{% script file="./content/scripts/getting-started/step11-grafana-get-password.sh" language="bash"%}}

## Cleanup

### Delete Karpenter nodes manually

If you delete a node with kubectl, Karpenter will gracefully cordon, drain,
and shutdown the corresponding instance. Under the hood, Karpenter adds a
finalizer to the node object, which blocks deletion until all pods are
drained and the instance is terminated. Keep in mind, this only works for
nodes provisioned by Karpenter.

{{% script file="./content/scripts/getting-started/step15-delete-node.sh" language="bash"%}}

### Delete the cluster
To avoid additional charges, remove the demo infrastructure from your AWS account.

{{% script file="./content/scripts/getting-started/step16-cleanup.sh" language="bash"%}}
