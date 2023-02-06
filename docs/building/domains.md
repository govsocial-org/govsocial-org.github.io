---
#template: home.html
title: DNS Domains
---

# DNS Domains

You will need a public DNS domain for your Fediverse instances if you want them to be accessible from the Internet. Because we wanted to host more than one Fediverse platform under the GOVSocial umbrella, we choose `govsocial.org` as our root domain, with `{instance}.govsocial.org` as the instance-specific hostname.

For Mastodon, there are a couple of things to be aware of when choosing a domain in general, and this configuration in particular:

- **Be aware of recent (December 21, 2022) changes to the [Mastodon Trademark Policy](https://joinmastodon.org/trademark)**. You will need written permission from Mastodon gGmbH to use 'Mastodon' or any derivatives (e.g. "mastodoon", "mast0don", "mstdn") in your domain or hostname[^1].
- **We wanted to have our user accounts use `@govsocial.org` for all our instances** instead of the actual instance hostname. As you will see [later](../mastodon/#load-balancer), this has implications for how both Mastodon and our load balancer are configured.

Register your domain (we use [Google Domains](https://domains.google/)), set [Google Cloud DNS](https://cloud.google.com/dns/docs/overview/) as the nameserver, and enable DNSSEC. The console will guide you through the steps outlined [here](https://cloud.google.com/dns/docs/set-up-dns-records-domain-name).

If you are fine with manually creating DNS records for your platforms in Cloud DNS (which is what we do), you're done at this point.

## ExternalDNS

The [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/kubernetes/external-dns/) implements [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) to manage the DNS entries required for the Fediverse instances. We *really* tried to get this to work (and pretty much did), but in the end, it was a lot of trouble-shooting to get it working on GKE with Cloud DNS, we were unable to get it to manage everything we needed without resorting to manual entry anyway, and it's not as if your DNS entries should change often enough to make it either necessary or worthwhile.

However, if you insist, here are some tips we learned during our efforts to get it working.

### Service Account

One of the challenges about deploying in different components of a cloud service provider is granting authenticated access between these components, particularly between services running in pods on your cluster and other services such as Cloud DNS.

The service accounts inside a GKE cluster are not part of the GCP IAM system that controls access to these other services. To grant services running GKE pods access to those services, the GKE service account in a pod needs to be mapped to an IAM account with the correct roles.

The first thing to do is install the `IAM Service Account Credentials API` into your GCP project. You can do this by selecting the `IAM & Admin` item on the GCP Console menu. If the API is not already installed, you will be prompted to add it. Once it is installed, navigate to `Service Accounts`, and create a new service account principal. You will need to assign this account the `DNS Administrator` role.

Next, you will need to map the GKE service account in the ExternalDNS pod to this IAM service account. The instructions for doing that are [here](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#authenticating_to). The GKE service account for ExternalDNS will be `external-dns`, if you are following [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/kubernetes/external-dns/).

### Custom Resource Definitions (CRDs)

If you are using [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to manage your DNS records (again, from the Cookbook), you will likely encounter this error:

```bash
error listing resources in GroupVersion \"externaldns.k8s.io/v1alpha1\": the server could not find the requested resource
```

To fix this, run this from your CLI machine:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/external-dns/master/docs/contributing/crd-source/crd-manifest.yaml
```

You will also likely encounter this error:

```bash
DNSEndpoint ("dnsendpoints.externaldns.k8s.io is forbidden: User \"system:serviceaccount:external-dns:external-dns\" cannot list resource \"dnsendpoints\" in API group \"externaldns.k8s.io\" at the cluster scope")
```

To this this one, run this from your CLI machine:

```bash
kubctl get clusterrole external-dns-external-dns -o yaml > {file}
```

Edit the file (with `vi` or your editor of choice - not getting into *that* debate here!) to add the following lines at the end:

```yaml
- apiGroups: ["externaldns.k8s.io"]
  resources: ["dnsendpoints"]
  verbs: ["get","watch","list"]
- apiGroups: ["externaldns.k8s.io"]
  resources: ["dnsendpoints/status"]
  verbs: ["*"]
```

Then apply the new configuration with:

```bash
kubectl apply -f {file}
```

Are we having fun yet?

Good, because we're not done.

### Record Ownership Tracking

You will experience conflicts between the default `TXT` DNS records that ExternalDNS uses to track the DNS records it manages, and any actual `TXT` records that you need (for [DKIM](https://www.dkim.org/), for example). You will need to set the `txtSuffix:` value to `"txt"` in the external-dns overrides file to avoid [this issue](https://github.com/kubernetes-sigs/external-dns/issues/262/). `txtPrefix:` doesn't seem to work.

Even after all this, we were unable to get two things to work at all:

- An `A` record for our "naked domain"
- The `MX` and `TXT` records necessary for our custom domain to work with AWS SES

This is the point at which we gave up and reverted to manual DNS record entry. YMMV.

[^1]: GOVSocial obtained this permission for our use of `mastodon.govsocial.org` on January 10, 2023