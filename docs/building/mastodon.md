---
#template: home.html
title: Building the Mastodon Instance
---

# Building the Mastodon Instance

In building our Mastodon instance, we basically followed [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/mastodon/), but with a few important changes. Some of these related to the specifics of deploying in GCP, and some related to our implementation choices for this instance.

## Helm Chart Repository

The first thing to do is to decide which Helm chart to use. We used the "official" one from [the Mastodon repo](https://github.com/mastodon/chart). As the Cookbook points out, this isn't a Helm repository, but a regular GitHub repository with the Helm chart in it. There is also a [Bitnami Helm chart](https://github.com/bitnami/charts/tree/main/bitnami/mastodon), which did not exist at the time we started (it was created on December 15, 2022), and which we have not tried.

**NOTE:** In all the file examples below, the path is from the root of your repo. Refer to the [Flux repo plan](/building/fluxhelm/#flux-repository-structure) if you're unsure where all the files go.

Having decided which chart to use, you need to create the appropriate repository entry in your Flux repo, so Flux knows where to pull the chart from. Here is the `GitRepository` entry we use for the Mastodon repo chart (there is an example of a [Bitnami Helm repo file](https://geek-cookbook.funkypenguin.co.nz/kubernetes/external-dns/#helmrepository) in the Cookbook):

```yaml title="/clusters/{my-cluster}/gitrepositories/gitrepository-mastodon.yaml"
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: mastodon
  namespace: flux-system
spec:
  interval: 1h0s
  ref:
    branch: main
  url: https://github.com/mastodon/chart
```
## Platform Namespace

Next, we will need to create a [Kubernetes namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) for all the resources for our Mastodon instance. Namespaces create a logical boundary within your cluster, grouping related resources together:

```yaml title="/clusters/{my-cluster}/namespaces/namespace-mastodon.yaml"
apiVersion: v1
kind: Namespace
metadata:
  name: mastodon
```

## Flux Kustomization

Now, we need to provide the instructions to Flux to connect to **our** repository (that reference was created when we [bootstrapped Flux](/building/fluxhelm#bootstrap-flux)), apply our custom deployment configuration, install it into the namespace, and run some health checks to make sure it worked. This uses a Kubernetes object called a [Kustomization](https://kubectl.docs.kubernetes.io/references/kubectl/kustomize/):

```yaml title="/clusters/{my-cluster}/kustomizations/kustomization-mastodon.yaml"
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: mastodon
  namespace: flux-system
spec:
  interval: 15m
  path: ./mastodon
  prune: true # remove any elements later removed from the above path
  timeout: 2m # if not set, this defaults to interval duration, which is 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: server
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: mastodon-web
      namespace: mastodon
    - apiVersion: apps/v1
      kind: Deployment
      name: mastodon-streaming
      namespace: mastodon
#    - apiVersion: apps/v1
#      kind: Deployment
#      name: mastodon-sidekiq
#      namespace: mastodon
```

The `spec.path` entry refers to a `./mastodon` directory at the root of **our** repository. This tells Flux to look in there for the configuration information for the build (we'll create that in just a moment).

**NOTE:** The Cookbook example includes a `healthChecks` entry for `mastodon-sidekiq`. This doesn't work with the Mastodon Helm chart as there is no listener in the deployed pod, so we commented it out.

## Custom Configuration

This is where the magic really happens. We have told Flux to look in the `./mastodon` directory in our GitHub respository that we [bootstrapped Flux](/building/fluxhelm#bootstrap-flux) with earlier. Now, we populate that directory with the special sauce that makes the whole build work.

### ConfigMaps

A [Kubernetes ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) is a set of name-value pairs that acts as a sort of configuration file for the services in your running pods. Because local storage in Kubernetes pods is ephemeral (it gets detroyed and re-created when the pod redeploys), we need a permanent way to store configuration information outside the pod's local file system.

The ConfigMap takes the place of the flat file called `.env.production` in the Mastodon directory of a server-based system that would be lost when the pod restarts in a Kubernetes-based system. The Mastodon Helm chart creates this flat file in each pod from the contents of the ConfigMap.

A Helm chart comes with a `values.yaml` file that is the analog of a default configuration file. We want to set those default values to the ones we want, so we use our own ConfigMap to "override" the default values. Just like we edit a default configuration file to change the values to what we want, our ConfigMap is basically a copy of the default one, with the appropriate values changed. Here is the ConfigMap for our Mastoson instance (comments have been removed for brevity and clarity):

```yaml title="/mastodon/configmap-mastodon-helm-chart-value-overrides.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: mastodon-helm-chart-value-overrides
  namespace: mastodon
data:
  values.yaml: |-  
    image:
      repository: tootsuite/mastodon
      tag: "v4.0.2"
      pullPolicy: IfNotPresent

    mastodon:
      createAdmin:
        enabled: true
        username: [redacted]
        email: [redacted]
      cron:
        removeMedia:
          enabled: true
          schedule: "0 0 * * 0"
      locale: en
      local_domain: govsocial.org
      web_domain: mastodon.govsocial.org
      singleUserMode: false
      authorizedFetch: false
      persistence:
        assets:
          accessMode: ReadWriteMany
          resources:
            requests:
              storage: 10Gi
        system:
          accessMode: ReadWriteMany
          resources:
            requests:
              storage: 100Gi
      s3:
        enabled: true
        force_single_request: true
        access_key: "[redacted]"
        access_secret: "[redacted]"
        existingSecret: ""
        bucket: "mastodongov"
        endpoint: "https://storage.googleapis.com"
        hostname: "storage.googleapis.com"
        region: "us"
        alias_host: ""
      secrets:
        secret_key_base: "[redacted]"
        otp_secret: "[redacted]"
        vapid:
          private_key: "[redacted]"
          public_key: "[redacted]"
        existingSecret: ""
      sidekiq:
        podSecurityContext: {}
        securityContext: {}
        resources: {}
        affinity: {}
        workers:
        - name: all-queues
          concurrency: 25
          replicas: 1
          resources: {}
          affinity: {}
          queues:
            - default,8
            - push,6
            - ingress,4
            - mailers,2
            - pull
            - scheduler
      smtp:
        auth_method: plain
        ca_file: /etc/ssl/certs/ca-certificates.crt
        delivery_method: smtp
        domain: notifications.govsocial.org
        enable_starttls: 'never'
        enable_starttls_auto: false
        from_address: mastodon@notifications.govsocial.org
        openssl_verify_mode: none
        port: 465
        reply_to:
        server: email-smtp.us-east-2.amazonaws.com
        ssl: false
        tls: true
        login: [redacted]
        password: [redacted]
        existingSecret:
      streaming:
        port: 4000
        workers: 1
        base_url: null
        replicas: 1
        affinity: {}
        podSecurityContext: {}
        securityContext: {}
        resources: {}
      web:
        port: 3000
        replicas: 1
        affinity: {}
        podSecurityContext: {}
        securityContext: {}
        resources: {}

      metrics:
        statsd:
          address: ""

    ingress:
      enabled: false
      annotations:
      ingressClassName:
      hosts:
        - host: mastodongov.org
          paths:
            - path: '/'
      tls:
        - secretName: mastodon-tls
          hosts:
            - mastodon.local

    elasticsearch:
      enabled: false
      image:
        tag: 7

    postgresql:
      enabled: false
      postgresqlHostname: [redacted]
      postgresqlPort: 5432
      auth:
        database: mastodongov
        username: mastodon
        password: "[redacted]"
        postgresPassword: "[redacted]"

    redis:
      enabled: true
      hostname: ""
      port: 6379
      password: ""

    service:
      type: ClusterIP
      port: 80

    externalAuth:
      oidc:
        enabled: false
      saml:
        enabled: false
      oauth_global:
        omniauth_only: false
      cas:
        enabled: false
      pam:
        enabled: false
      ldap:
        enabled: false

    podSecurityContext:
      runAsUser: 991
      runAsGroup: 991
      fsGroup: 991

    securityContext: {}

    serviceAccount:
      create: true
      annotations: {}
      name: ""

    podAnnotations: {}

    jobAnnotations: {}

    resources: {}

    nodeSelector: {}

    tolerations: []

    affinity: {}
```

The quickest way to create this file is to create this much by hand:

```yaml title="/mastodon/configmap-mastodon-helm-chart-value-overrides.yaml"
apiVersion: v1
kind: ConfigMap
metadata:
  name: mastodon-helm-chart-value-overrides
  namespace: mastodon
data:
  values.yaml: |-  
```

and then paste in the rest of it underneath from the `values.yaml` from the Helm chart repo you used, being careful to indent everything you paste in by 4 spaces ([YAML](https://yaml.org/spec/1.2.2/) is picky about indentation).

Now, let's walk through the specifics of how we configured Mastodon to deploy with the various services (AWS SES, Cloud Storage, and Cloud SQL) that we prepared earlier.

#### AWS SES

Here is the section of our ConfigMap that relates to the AWS SES service we [prepared earlier](/building/email#setting-up-aws-ses):

```yaml
      smtp:
        auth_method: plain
        ca_file: /etc/ssl/certs/ca-certificates.crt
        delivery_method: smtp
        domain: notifications.govsocial.org
        enable_starttls: 'never'
        enable_starttls_auto: false
        from_address: mastodon@notifications.govsocial.org
        openssl_verify_mode: none
        port: 465
        reply_to:
        server: email-smtp.us-east-2.amazonaws.com
        ssl: false
        tls: true
        login: [redacted]
        password: [redacted]
        # -- you can also specify the name of an existing Secret
        # with the keys login and password
        existingSecret:
```

This took a surprising amount of trial and error to get working. If you have problems, and you have managed to get the rest of your Mastodon instance running, the Sidekiq Retries queue (in the `Administration` settings in Mastodon) is your friend. It will provide you with useful error messages to help you troubleshoot the problem.

The `domain:` setting will be the custom domain that you set up and verified with SES, and the `from_address:` can be any email address from that domain. You should paste the values for `login:` and `password:` from the SMTP Credential you configured in the SES console, remembering that `login:` is the randomly generated account name for the credential, not the name of the account principal itself.

The tricky part was getting the connection to work. We could not get `STARTTLS` on port `587` to work at all, and only got [implicit TLS](https://mailtrap.io/blog/starttls-ssl-tls/) working by setting `enable_starttls:` to `'never'`, `enable_starttls_auto:` to `false`, `openssl_verify_mode:` to `none`, `ssl:` to `false`, and `tls:` to `true`. 

#### Cloud Storage

Here is the section of our ConfigMap that relates to the S3-compatible storage we [prepared earlier](/building/storage/):

```yaml
      s3:
        enabled: true
        force_single_request: true
        access_key: "[redacted]"
        access_secret: "[redacted]"
        # -- you can also specify the name of an existing Secret
        # with keys AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
        existingSecret: ""
        bucket: "mastodongov"
        endpoint: "https://storage.googleapis.com"
        hostname: "storage.googleapis.com"
        region: "us"
        # -- If you have a caching proxy, enter its base URL here.
        alias_host: ""
```

You'll paste in the `access_key` and `access_secret` values from the [HMAC key](https://cloud.google.com/storage/docs/authentication/managing-hmackeys) that you created for your Cloud Storage bucket, and set `bucket` to the name you chose for it.

The only catch we stumbled on when getting this to work was the addition of `force_single_request: true`, as Google Cloud Storage cannot handle chunked requests.

#### PostgreSQL

Here is the section of our ConfigMap that relates to the PostgreSQL database we [prepared earlier](/building/postgres/):

```yaml
    postgresql:
      # -- disable if you want to use an existing db; in which case the values below
      # must match those of that external postgres instance
      enabled: false
      postgresqlHostname: [redacted]
      postgresqlPort: 5432
      auth:
        database: mastodongov
        username: mastodon
        # you must set a password; the password generated by the postgresql chart will
        # be rotated on each upgrade:
        # https://github.com/bitnami/charts/tree/master/bitnami/postgresql#upgrade
        password: "[redacted]"
        # Set the password for the "postgres" admin user
        # set this to the same value as above if you've previously installed
        # this chart and you're having problems getting mastodon to connect to the DB
        postgresPassword: "[redacted]"
        # you can also specify the name of an existing Secret
        # with a key of password set to the password you want
        # existingSecret: ""
```

The first thing to note is that `enabled` is set to `false`. This seems counter-intuitive - after all, we need a PostgreSQL database for the thing to work. In this case, the `enabled` setting really tells the Mastodon deployment whether or not to create a new database locally in the cluster or not. In our deployment, we did not want that - we wanted to use the Cloud SQL database that we already created.

The `postgresqlHostname` setting will be the internal IP address of your PostgreSQL instance.

Remember that we created a separate database for each platform we are running, so we changed the `database` and `username` to match what we created. Because we are also using a different user for the platform database, we needed to set both the `password` (which is the password of the account in the `username` setting) and the `postgresPassword` (which is the password of the default `postgres` account) to the correct values. Mastodon uses each for different database tasks, so it needs both passwords in this configuration.

When you get to the actual deployment and your pods spin up, you will notice a [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) called `mastodon-db-migrate` spin up as well. This job is creating the correct database schema for your instance. Your other mastodon pods may not be available until that job completes.

## HelmRelease

Now that we have our instance-specific ConfigMap in place, we can create the HelmRelease that will pull the Helm chart from the repository we chose for it and apply our ConfigMap to it. Here is the HelmRelease file we use:

```yaml title="/mastodon/helmrelease-mastodon.yaml"
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mastodon
  namespace: mastodon
spec:
  chart:
    spec:
      chart: ./
      sourceRef:
        kind: GitRepository
        name: mastodon
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: mastodon
  valuesFrom:
  - kind: ConfigMap
    name: mastodon-helm-chart-value-overrides
    valuesKey: values.yaml 
```

With all this in place, here is the sequence of events:

- The `/clusters/{my-cluster}/kustomizations/kustomization-mastodon.yaml` Kustomization tells Flux on that cluster to monitor the `./mastodon` path **our** repository (`name: flux-system`) for changes every `interval: 15m`
- When a change is detected, it updates the `name: mastodon-helm-chart-value-overrides` ConfigMap from **our** repo on the cluster, and...
- pulls the `name: mastodon` HelmRelease, which points at the **platform** repository (`name: mastodon`) containing the Helm chart that builds the pods and services deploys the chart in our cluster, with their local `.env.production` files built from the updated ConfigMap.
- The `healthChecks` in the Kustomization are run to make sure everything deployed successfully.

**NOTE:** The **platform** repository is itself monitored for changes every `interval: 15m`, and changes in that will also trigger a Flux reconcilliation. If you want to avoid unexpected upgrades, you can specify [a valid `image.tag`](https://hub.docker.com/r/tootsuite/mastodon/tags) in your ConfigMap. This is particularly important now that v4.1 is imminent, and the published Helm Chart could change without warning.

## Install Mastodon

You can either wait for Flux to detect your changes, or you can speed up the process by running the following from your CLI machine:

```bash
~$ flux reconcile source git flux-system
```

You can see the Mastodon pods by running the following from your CLI machine (or looking in your GKE console):

```bash
~$ kubectl get pods -n mastodon

mastodon-redis-master-0                       1/1     Running
mastodon-redis-replicas-0                     1/1     Running
mastodon-redis-replicas-1                     1/1     Running
mastodon-redis-replicas-2                     1/1     Running
mastodon-sidekiq-all-queues-7d8b8c596-456kx   1/1     Running
mastodon-sidekiq-all-queues-7d8b8c596-97rfv   1/1     Running
mastodon-sidekiq-all-queues-7d8b8c596-98d2x   1/1     Running
mastodon-streaming-5bdc55888b-95fbn           1/1     Running
mastodon-streaming-5bdc55888b-cwnsv           1/1     Running
mastodon-streaming-5bdc55888b-czh28           1/1     Running
mastodon-web-56cc95dd99-k7mfg                 1/1     Running
mastodon-web-56cc95dd99-n524q                 1/1     Running
mastodon-web-56cc95dd99-vmkxf                 1/1     Running
```

That's it! You're done building your Mastodon instance on GKE! Now, we need to make sure people can access it.