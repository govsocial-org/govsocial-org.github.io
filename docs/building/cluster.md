---
#template: home.html
title: Creating the Cluster
---

# Creating the cluster

None of the How-Tos we found covered the creation of the initial Kubernetes cluster, because this is really down to the choice of hosting environment, and appetite for cost. We created a standard cluster in GKE, with the following cost-saving caveats:

- As tempting as it sounds from an ease of management perspective, **do NOT create an [AutoPilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview) cluster**. The [minimum vCPU and Memory requests](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-resource-requests) for an AutoPilot cluster will bankrupt you before you start (roughly $200/month!).
- **Use [spot VMs](https://cloud.google.com/compute/docs/instances/spot) for the nodes** on which your cluster will be deployed. They offer significant cost savings over standard VMs and allow you to use decent-sized machines for your nodes at a fraction of the cost. We chose the [e2-medium shared core](https://cloud.google.com/compute/docs/general-purpose-machines#e2-shared-core) machines, and set the cluster to auto-scale from 0 to 3 nodes. You will want to make sure to set a [Pod Disruption Budget (PDB)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) to make sure that auto-updating and pre-emption of the Spot VM nodes doesn't disrupt your instance services. We took the advice given [here](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pool-upgrade-strategies) to set `maxSurge=1 maxUnavailable=0`.
- **Start with the smallest cluster available**. You can always add more and beefier nodes to it later.

**IMPORTANT:** You will need to make sure you create your cluster with [Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) enabled. Authenticated access from the cluster service account (which is **NOT** an IAM service account) to other GCP services such as Cloud Storage depends on it.