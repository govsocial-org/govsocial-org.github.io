---
#template: home.html
title: Flux and Helm
---

# Flux and Helm

One of the best decisions we made as we started our first platform install (Mastodon), was to follow the advice in [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/mastodon/) and use [Flux](https://fluxcd.io/flux/) to bootstrap our cluster and deploy [Helm charts](https://helm.sh/) and [Flux Kustomizations](https://fluxcd.io/flux/components/kustomize/kustomization/) from a private GitHub repo. Doing this made it relatively easy to install what we needed from existing Helm charts. It also allowed us to quickly rebuild our cluster (something we did at least twice!) as we made poor configuration choices and mistakes while we figured things out.

## Install Flux-CLI

To start, you will need to [install the Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli). Rather than install locally, we spun up an `e2-micro` VM in Google Compute Engine to host this, `kubectl`, and the other tools we would need for our build. Rather than messing with the Docker image, we installed the binaries with:

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

!!! Note
    To run `flux-cli` from a [Compute Engine VM](https://cloud.google.com/compute/docs/instances/create-start-instance), your `Compute Engine default service account` needs the `Kubernetes Engine Admin` role. You can edit the roles assigned to service account principals in [IAM](https://cloud.google.com/iam/docs/granting-changing-revoking-access#single-role).

You should also get used to running this each time you connect to your VM:

```bash
gcloud container clusters get-credentials {my-cluster} --region {region}
```

This will authenticate you to your cluster.

You should also install the `kubectl` and `google-cloud-sdk-gke-gcloud-auth-plugin` packages at this point, from your machine's package manager. Don't forget to periodically run `sudo apt-get update && sudo apt-get upgrade` on your machine to keep your packages up to date - the Google Cloud packages update quite often.

## Bootstrap Flux on the Cluster

These instructions assume you are using a GitHub personal or organizational repo. Additional instructions for other git repositories can be found [here](https://fluxcd.io/flux/installation/#bootstrap).

### GitHub Setup

Before we bootstrap Flux on the cluster, a word about your GitHub repo setup. You will need a **private** GitHub personal or organizational repo.

!!! Danger "Danger, Will Robinson" 
    **Make sure the repo is private**, as your instance configurations, **including secrets**, will be stored there.
    
When bootstrapped as detailed below, your repo will be used for all your Flux deployments, so avoid naming it for a particular cluster or platform.

#### Personal Access Token (PAT)

Flux uses a `classic` Personal Access Token (PAT) created for the personal GitHub account you use to access this repo. PATs are created and maintained in `<> Developer Settings` at the very bottom of the Settings menu for your account (accessed by clicking on your avatar at the top right of your GitHub window and choosing `Settings`). The PAT will need the full `repo` role.

!!! Warning
    **Make a note of the token (it will start with** `gpg_`**) - you will only be shown it once.**

By default, the PAT will expire every 30 days. You will get an email from GitHub 7 days before it expires. [To rotate it](https://github.com/fluxcd/flux2/discussions/2161):

- Delete the auth secret from the cluster with:
```bash
kubectl -n flux-system delete secret flux-system
```
- Rerun flux bootstrap with the same args as you use to set it up
- Flux will generate a new secret and will update the deploy key if you’re using SSH deploy keys (you will be). You should get another email from GitHub telling you that a new SSH key has been created, and you're all set.

### Bootstrap Flux

Bootstrap Flux on your cluster by running the following:

```bash
flux bootstrap github \
  --owner={my-github-username} \
  --repository={my-repository} \
  --path=clusters/{my-cluster} \
  --personal
```

!!! Note
    If you are using an organizational repo, omit `--personal` from the options.

You will be prompted for your PAT, which you can copy from where you stored it and paste into the command prompt before hitting `[Enter]` (you won't see anything when you paste it in).

If all goes well, you should see the following by typing `kubectl get pods -n flux-system` at the command prompt:

```bash
kubectl get pods -n flux-system
```
``` {.bash .no-copy}
helm-controller-564bf65b8b-fswms           1/1     Running     0
kustomize-controller-7cbf46474d-5qxnw      1/1     Running     0
notification-controller-7b665fb8bd-t9vmr   1/1     Running     0
source-controller-84db8b78b9-xjkfm         1/1     Running     0
```

## Flux Repository Structure

Having successfully carried out the above steps, you will now have started a Flux repo with the following structure plan (this will get built as we journey through the deployment of a platform):

``` {.bash .no-copy}
/clusters
  |/{cluster1}
    |/flux-system # <-- This controls the Flux installation on the cluster
    |/[git|helm]repositories # <-- This is where the Helm chart sources for the platforms are defined
    |/namespaces # <-- This is where the k8s namespaces for the platforms are defined
    |/kustomizations
      |kustomization-{platform1}.yaml #)     We create these when we deploy platforms like Mastodon
      |kustomization-{platform3}.yaml #) <-- They tell flux to look in the matching directory in the
      |kustomization-{platform4}.yaml #)     repo root for instructions
  |/{cluster2}
    |/flux-system # <-- This controls the Flux installation on the cluster
    |/[git|helm]repositories # <-- This is where the Helm chart sources for the platforms are defined
    |/namespaces # <-- This is where the k8s namespaces for the platforms are defined
    |/kustomizations
      |kustomization-{platform2}.yaml #)     We create these when we deploy platforms like Mastodon
      |kustomization-{platform5}.yaml #) <-- They tell flux to look in the matching directory in the
      |kustomization-{platform6}.yaml #)     repo root for instructions               |
/{platform1} #)                                                                       |
/{platform2} #)                                                                       |
/{platform3} #) <--We create these to hold configuration details for each platform <--|
/{platform4} #)
/{platform5} #)
/{platform6} #)
```
This is an extremely powerful and flexible plan that allows different platforms to be deployed on multiple clusters from the same Flux repo.

With all that set up, we can have some fun!