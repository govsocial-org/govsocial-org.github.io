---
#template: home.html
title: Building the Mastodon Instance
---

# Building the Mastodon Instance

In building our Mastodon instance, we basically followed [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/mastodon/), but with a few important changes. Some of these related to the specifics of deploying in GCP, and some related to our implementation choices for this instance.

## Helm Chart Repository

The first thing to do is to decide which Helm chart to use. We used the "official" one from [the Mastodon repo](https://github.com/mastodon/chart). As the Cookbook points out, this isn't a Helm repository, but a regular GitHub repository with the Helm chart in it. There is also a [Bitnami Helm chart](https://github.com/bitnami/charts/tree/main/bitnami/mastodon), which did not exist at the time we started (it was created on December 15, 2022), and which we have not tried.

!!! Note
    In all the file examples below, the path is from the root of your repo. Refer to the [Flux repo plan](/building/fluxhelm/#flux-repository-structure) if you're unsure where all the files go.

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

Now, we need to provide the instructions to Flux to connect to **our** repository (that reference was created when we [bootstrapped Flux](/building/fluxhelm/#bootstrap-flux)), apply our custom deployment configuration, install it into the namespace, and run some health checks to make sure it worked. This uses a Kubernetes object called a [Kustomization](https://kubectl.docs.kubernetes.io/references/kubectl/kustomize/):

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

The `spec.path:` entry refers to a `./mastodon` directory at the root of **our** repository. This tells Flux to look in there for the configuration information for the build (we'll create that in just a moment).

!!! Note
    [The Cookbook example](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/mastodon/#kustomization) includes a `healthChecks` entry for `mastodon-sidekiq`. This doesn't work with the Mastodon Helm chart as there is no listener in the deployed pod, so we commented it out.

## Custom Configuration

This is where the magic really happens. We have told Flux to look in the `./mastodon` directory in our GitHub respository that we [bootstrapped Flux](/building/fluxhelm/#bootstrap-flux) with earlier. Now, we populate that directory with the special sauce that makes the whole build work.

### ConfigMap

A [Kubernetes ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) is a set of name-value pairs that acts as a sort of configuration file for the services in your running pods. Because local storage in Kubernetes pods is ephemeral (it gets detroyed and re-created when the pod redeploys), we need a permanent way to store configuration information outside the pod's local file system.

The ConfigMap takes the place of the flat file called `.env.production` in the Mastodon directory of a server-based system that would be lost when the pod restarts in a Kubernetes-based system. The Mastodon Helm chart creates this flat file in each pod from the contents of the ConfigMap.

A Helm chart comes with a `values.yaml` file that is the analog of a default configuration file. We want to set those default values to the ones we want, so we use our own ConfigMap to "override" the default values. Just like we edit a default configuration file to change the values to what we want, our ConfigMap is basically a copy of the default one with the appropriate values changed. Here is the ConfigMap for our Mastoson instance (comments have been removed for brevity and clarity):

??? Example "configmap-mastodon-helm-chart-value-overrides.yaml"
    ```yaml
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

#### General

You will want to set the `local_domain:` (and `web_domain:`, if it's different) values to those you configured when [preparing your DNS domains](/building/domains/).

You will also need to pick the `username:` and `email:` for the Mastodon account that will be the initial admin user for the instance. You can add other users to various roles in the instance once it's running.

!!! Note
    Our install didn't create this user - if this happens to you, there is a workaround that you can do once your instance is running.

#### Secrets

Mastodon uses a set of secrets for client/server authentication, 2FA, and authentication between its various services. Here is the relevant section of the ConfigMap:

```yaml
secrets:
  secret_key_base: "[redacted]"
  otp_secret: "[redacted]"
  vapid:
    private_key: "[redacted]"
    public_key: "[redacted]"
  # -- you can also specify the name of an existing Secret
  # with keys SECRET_KEY_BASE and OTP_SECRET and
  # VAPID_PRIVATE_KEY and VAPID_PUBLIC_KEY
  existingSecret: ""
```

To obtain the `secret_key_base:` and `otp_secret:` values, you will need to install the `rake` package from the package manager on your CLI machine. Create a file named `rakefile` in your working directory with these contents:

```ruby title="rakefile"
desc 'Generate a cryptographically secure secret key (this is typically used to generate a secret for cookie sessions).'
task :secret do
  require 'securerandom'
  puts SecureRandom.hex(64)
end
```

Then, run `rake secret` twice, once for the `secret_key_base`, and once for the `otp_secret`, and paste the values into your ConfigMap.

The `vapid` key is a public/private keypair. We used an [online Vapid key generator](https://www.attheminute.com/us/vapid-key-generator) for these.

#### Ingress

Note that the `ingress.enabled:` value is set to `false`. The chart doesn't contain a spec for the GKE Ingress, and we created ours by hand once our instance was up and running.

#### Elastic Search

We also left `elasticsearch.enabled:` set to `false`. Elastic requires its own cluster, which would have increased our initial hosting costs considerably. We may add this feature (which allows users to perform full-text searches on their timelines) at some point.

Now, let's walk through the specifics of how we configured Mastodon to deploy with the various services (AWS SES, Cloud Storage, and Cloud SQL) that we prepared earlier.

#### SMTP

Here is the section of our ConfigMap that relates to the AWS SES service we [prepared earlier](/building/email/#setting-up-aws-ses):

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

The tricky part was getting the connection to work. We could not get `STARTTLS` on port `587` to work at all, and only got [implicit TLS](https://mailtrap.io/blog/starttls-ssl-tls/) working by setting `enable_starttls: 'never'`, `enable_starttls_auto: false`, `openssl_verify_mode: none`, `ssl: false`, and `tls: true`. 

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

You'll paste in the `access_key:` and `access_secret:` values from the [HMAC key](https://cloud.google.com/storage/docs/authentication/managing-hmackeys) that you created for your Cloud Storage bucket, and set `bucket:` to the name you chose for it.

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

The first thing to note is that `postgres.enabled:` is set to `false`. This seems counter-intuitive - after all, we need a PostgreSQL database for the thing to work. In this case, the `enabled:` setting really tells the Mastodon deployment whether or not to enable a database locally in the cluster or not. In our deployment, we did not want that - we wanted to use the Cloud SQL database that we already created.

The `postgresqlHostname:` setting will be the internal IP address of your PostgreSQL instance.

Remember that we created a separate database for each platform we are running, so we changed the `database:` and `username:` to match what we created. Because we are also using a different user for the platform database, we needed to set both the `password:` (which is the password of the account in the `username:` setting) and the `postgresPassword:` (which is the password of the default `postgres` account) to the correct values. Mastodon uses each for different database tasks, so it needs both passwords in this configuration.

When you get to the actual [deployment](/building/mastodon/#deploy-mastodon) and your pods spin up, you should notice a [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) called `mastodon-db-migrate` spin up as well. This job is creating the correct database schema for your instance. Your other Mastodon pods may not be available until that job completes.

!!! Note
    You may find that the `mastodon-db-migrate` job doesn't run with `postgres.enabled` set to `false` (although our experience was based on incorrectly setting it to `true` in our initial deployment and desperately trying to switch to Cloud SQL after [deploying Mastodon](/building/mastodon/#deploy-mastodon)[^1] ).

YMMV with a correctly configured fresh install, but if that happens to you, here is how we fixed it. The `mastodon-web` and `mastodon-sidekiq` pods will fail to start, but the `mastodon-streaming` pod will because, unlike the other two, it is not dependent on a database connection. The dirty little secret is that all three pods are running the same Mastodon image, so we can use the running `mastodon-streaming` pod to access a running Mastodon service and the `rails` environment we need.

Connect to the running `mastodon-streaming` pod from your CLI machine by entering this:

```bash
~$ kubectl get pods -n mastodon
```
``` {.bash .no-copy}
NAME                                          READY   STATUS
mastodon-streaming-bb578b4bc-gzfr2            1/1     Running
mastodon-streaming-bb578b4bc-nszjf            1/1     Running
mastodon-streaming-bb578b4bc-wsv2l            1/1     Running
```
```bash
~$ kubectl exec -it mastodon-streaming-bb578b4bc-wsv2l -n mastodon /bin/sh
```

It doesn't matter which pod you connect to, if there is more than one, like in the example above. Once you are connected to a pod, run the following:

```sh
$ export OTP_SECRET=[redacted]
$ export SECRET_KEY_BASE=[redacted]
$ RAILS_ENV=production bundle exec rails db:migrate
```

You should see the `mastodon-db-migrate` job appear in your `Workloads` in the Cloud Console, and this will prepare your database. Once the job is finished, your `mastodon-web` and `mastodon-sidekiq` pods should restart successfully.

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

!!! Note
    The **platform** repository is itself monitored for changes every `interval: 15m`, and changes in that will also trigger a Flux reconcilliation. If you want to avoid unexpected upgrades, you can specify [a valid `image.tag`](https://hub.docker.com/r/tootsuite/mastodon/tags) in your ConfigMap. This is particularly important now that v4.1 is imminent, and the published Helm Chart could change without warning.

## Deploy Mastodon

You can either wait for Flux to detect your changes, or you can speed up the process by running the following from your CLI machine:

```bash
flux reconcile source git flux-system
```

You can see the Mastodon pods by running the following from your CLI machine (or looking in your GKE console):

```bash
kubectl get pods -n mastodon
```
``` {.bash .no-copy}
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

That's it! You're done deploying your Mastodon instance on GKE! Now, we need to make sure people can access it.

## Ingress

You will [remember](/building/mastodon/#ingress) that we did not enable the ingress that is included in the Mastodon Helm chart and instead opted to configure the GKE Ingress by hand.

You can do this in the console by going to `Services & Ingress` in the GKE menu in Google Cloud Console. You will need an `External HTTPS Ingress` with two ingress paths to make Mastodon work properly, especially with mobile applications:

- A path for the `mastodon-streaming` service, with the path set to `/api/v1/streaming`
- A path for the `mastodon-web` service, with the path set to `/*`

As part of creating an HTTPS ingress, you will need a TLS certificate. We opted to use a Google-manged certificate. The domain for the certificate needs to be for the `web_domain` of the instance (or `local_domain`, if `web_domain` is not set).

The `spec` for the resulting ingress should look like this:

```yaml
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: mastodon-web
            port:
              number: 3000
        path: /*
        pathType: ImplementationSpecific
      - backend:
          service:
            name: mastodon-streaming
            port:
              number: 4000
        path: /api/v1/streaming
        pathType: ImplementationSpecific
```

The Ingress will be allocated an external IP address that you should add as an `A` record for the instance hostname in your DNS record set. Your TLS certificate will not validate until that DNS record propogates (usually within half an hour or so).

Once it's all up and running, you should be able to connect to your instance from your web browser!

## Load Balancer

For GOVSocial.org, we wanted our user accounts to have the scheme `{username}@govsocial.org`, rather than having the full domain of each platform in them, like `{username}@{platform}.govsocial.org`. This means that our `local_domain` in Mastodon is `govsocial.org`, while our `web_domain` is `mastodon.govsocial.org`.

This poses challenges for federation, as any links to user profiles on other instances will intially connect to `govsocial.org` (the `local_domain`) instead of our `web_domain` and will need to be redirected. This redirection is handled by a [`webfinger` service in Mastodon](https://docs.joinmastodon.org/spec/webfinger/).

The load balancer needs to redirect requests for `govsocial.org/.well-known/webfinger` (which is where other instances think it is based on our username scheme) to `mastodon.govsocial.org/.well-known/webfinger` (where the service is actually listening).

To do this, we deployed an `HTTPS (classic) Load Balancer` in the `Network Services -> Load Balancing` menu in our Google Cloud Console. The setup is very similar to the [Ingress we set up earlier](#ingress_1). In fact, you will see the load balancer created by the Ingress in the list when you go there.

!!! Warning
    Don't mess with it :-) If you're the sort of person who can't help pressing big red buttons that say "DO NOT PRESS", it's okay - anything you change will eventually be reverted by the GKE configuration, but your ingress might be broken until that happens.

Create a new `HTTPS (classic) Load Balancer` (this one supports the hostname and path redirect features we need). Despite the higher cost, we selected the Premium network tier, as it allows for the addition of services like [Cloud Armor](https://cloud.google.com/armor/docs/cloud-armor-overview) and [Cloud CDN](https://cloud.google.com/cdn/docs/overview) if the platform needs it in the future.

For the Frontend configuration, make sure to create a new fixed external IP address and add the corresponding `A` record for `local_domain` in your DNS record set. Because we want to use HTTPS, you will need to create a TLS certificate for your `local_domain`. The certificate won't validate until this DNS record propogates (usually with half an hour or so).

For the Backend configuration, pick the default `kube-system-default-http-backend-80` service (there will be a bunch of letters/numbers before and after it.) This service doesn't have anything listening on it, but it will be used for the mandatory default rule in the rules configuration.

In the `Host and path rules`, create a `Prefix redirect` for your `local_domain`, set the `Host redirect` to your `web_domain`, and the `Paths` to `/.well-known/webfinger`. Select `301 - Moved Permanently` for the response, and make sure that `HTTPS redirect` is enabled.

Save your configuration, and wait for it to become available in the Cloud Console. Once it does, you should be able to connect to your Mastodon instance in your browser, and start [operating it](/operating/mastodon/).

!!! Note
    One of the advantages of having this load balancer in conjunction with our domain scheme is that it means that we can use the default path (`/*`) of GOVSocial.org for documentation and non-instance specific content. We created a similar rule in our load balancer for `/*` that redirects to `docs.govsocial.org`, which is what you are reading now. There is a whole other write-up for [how we created that](/operating/documentation/)!

[^1]: Flux to the rescue again! This was one of the issues (the other was abandoning AutoPilot) that had us delete all the Mastodon workloads and start over. The postgreSQL configuration seems to be particularly "sticky" and, try as we might, we could not get the corrected configuration to take after the initial deployment.