---
#template: home.html
title: Building a k8s FediBlockHole Cron Job
---

# Building a k8s FediBlockHole Cron Job

FediBlockHole is a Python tool that is designed to be installed on a machine and run from the command line. We could have installed it on our CLI VM and run it from there with a crontab entry to automate it, but there are a couple of disadvantages with this approach. Firstly, it would require our VM to be more stable than a [spot VM](https://cloud.google.com/compute/docs/instances/spot), increasing our hosting costs. Secondly, the supported VM image in GCP comes with Python 3.9.x, and FediBlockHole requires at least version 3.10.

Rather than embarking on that expedition, we decided to containerize FediBlockHole, and deploy a Kubernetes cron job that instantiates the container and runs the script.

!!! Note
    When we did this, we **forked** rather than **cloning** the [FediBlockHole repo](https://github.com/eigenmagic/fediblockhole) into [our own repo](https://github.com/cunningpike/fediblockhole). We wanted to submit a [pull request](https://github.com/eigenmagic/fediblockhole/pull/38) for ingrating our work back into the original project, and to keep our repo up to date with it. The following documentation is from that perspective. All repo paths are relative to the root of our forked repo.

## Containerizing FediBlockHole

To prepare for containerizing FediBlockHole, we installed Docker on our CLI machine from its package manager:

```bash
~$ sudo apt-get install docker.io
```

!!! Note
    In a GCP Compute VM, the package is called `docker.io`. YMMV.

Next, we prepared a Dockerfile for the Docker container:

```dockerfile title="/container/Dockerfile"
# Use the official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:slim

# Copy local code to the container image.
ENV APP_HOME /app
WORKDIR $APP_HOME

# Install production dependencies.
RUN pip install fediblockhole

USER 1001
# Run the script on container startup.
ENTRYPOINT ["fediblock-sync"]
```

We also made sure there was a Python-flavored Docker ignore file:

```yaml title="/container/.dockerignore"
Dockerfile
#README.md
*.pyc
*.pyo
*.pyd
__pycache__
```

Then, we built our Docker image:

```bash
~$ sudo docker build https://github.com/[user|organization]/fediblockhole.git#main:container --no-cache
```
``` {.bash .no-copy}
Successfully built {container-id}
```

!!! Note
    The `--no-cache` option forces a refresh from the repo rather than using the local cache. This is important for updates.

Using the `{container-id}` from `docker build`, we annotated the container with the version of FediBlockHole it was built with (currently `0.4.2`), and `latest`:

```bash
~$ sudo docker image tag {container-id} fediblockhole:latest
~$ sudo docker image tag {container-id} fediblockhole:0.4.2
~$ sudo docker image tag fediblockhole:latest ghcr.io/[user|organization]/fediblockhole:latest
~$ sudo docker image tag fediblockhole:0.4.2 ghcr.io/[user|organization]/fediblockhole:0.4.2
```

We signed into our GitHub repository with this command:

```bash
~$ sudo docker login ghcr.io -u {user_name}
```

Once signed in, we pushed the image to our `Packages` in GitHub:

```bash
~$ sudo docker push -a ghcr.io/[user|organization]/fediblockhole
```

The completed package can be found [here](https://github.com/cunningpike/fediblockhole/pkgs/container/fediblockhole).

## Creating the Helm Chart

Next, we wanted to leverage the Flux/Helm system [we built earlier](../fluxhelm/) to deploy a Kubernetes cron job in our cluster.

### Chart File

The first thing we needed was a [Helm chart](https://helm.sh/docs/topics/charts/):

```yaml title="/chart/Chart.yaml"
apiVersion: v2
name: fediblockhole
description: FediBlockHole is a tool for keeping a Mastodon instance blocklist synchronised with remote lists.

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
version: 1.0.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
appVersion: 0.4.2
```

### Helm Ignore File

We also created a Helm ignore file:

```yaml title="/chart/.helmignore"
# A helm chart's templates and default values can be packaged into a .tgz file.
# When doing that, not everything should be bundled into the .tgz file. This
# file describes what to not bundle.
#
# Manually added by us
# --------------------
#

# Boilerplate .helmignore from `helm create mastodon`
# ---------------------------------------------------
#
# Patterns to ignore when building packages.
# This supports shell glob matching, relative path matching, and
# negation (prefixed with !). Only one pattern per line.
.DS_Store
# Common VCS dirs
.git/
.gitignore
.bzr/
.bzrignore
.hg/
.hgignore
.svn/
# Common backup files
*.swp
*.bak
*.tmp
*.orig
*~
# Various IDEs
.project
.idea/
*.tmproj
.vscode/
```

### Values File

Every Helm chart needs a [Values file](https://helm.sh/docs/chart_template_guide/values_files/), which passes values into the [chart's templates](#templates) for the build. This is the Values file for this chart:

```yaml title="/chart/values.yaml"
image:
  repository: ghcr.io/cunningpike/fediblockhole
  # https://github.com/cunningpike/fediblockhole/pkgs/container/fediblockhole/versions
  #
  # alternatively, use `latest` for the latest release or `edge` for the image
  # built from the most recent commit
  #
  # tag: latest
  tag: ""
  # use `Always` when using `latest` tag
  pullPolicy: IfNotPresent

fediblockhole:
  # location of the configuration file. Default is /etc/default/fediblockhole.conf.toml
  conf_file:
    path: ""
    filename: ""
  cron:
    # -- run `fediblock-sync` every hour
    sync:
      # @ignored
      enabled: false
      # @ignored
      schedule: "0 * * * *"

# if you manually change the UID/GID environment variables, ensure these values
# match:
podSecurityContext:
  runAsUser: 991
  runAsGroup: 991
  fsGroup: 991

# @ignored
securityContext: {}

# -- Kubernetes manages pods for jobs and pods for deployments differently, so you might
# need to apply different annotations to the two different sets of pods. The annotations
# set with podAnnotations will be added to all deployment-managed pods.
podAnnotations: {}

# -- The annotations set with jobAnnotations will be added to all job pods.
jobAnnotations: {}

# -- Default resources for all Deployments and jobs unless overwritten
resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #   cpu: 100m
  #   memory: 128Mi
  # requests:
  #   cpu: 100m
  #   memory: 128Mi

# @ignored
nodeSelector: {}

# @ignored
tolerations: []

# -- Affinity for all pods unless overwritten
affinity: {}
```

The `repository:` value points to the location of the [GitHub Package](https://github.com/cunningpike/fediblockhole/pkgs/container/fediblockhole) we pushed our Docker image to. The `tag:` value can be used to override the `appVersion:` value in the [chart](#chart-file).

!!! Note
    We specified a non-root user and group in our Values file to make sure that the containerized script does not run as the root user.

### Templates

All the preceding files are really just metadata for the Helm deployment - the real work is done in the [templates](https://helm.sh/docs/chart_template_guide/getting_started/).

#### Cron Job

The template for the Kubernetes cron job we needed looks like this:

```golang title="/chart/templates/cronjob-fediblock-sync.yaml" hl_lines="23"
{% raw %}
{{ if .Values.fediblockhole.cron.sync.enabled -}}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "fediblockhole.fullname" . }}-sync
  labels:
    {{- include "fediblockhole.labels" . | nindent 4 }}
spec:
  schedule: {{ .Values.fediblockhole.cron.sync.schedule }}
  jobTemplate:
    spec:
      template:
        metadata:
          name: {{ include "fediblockhole.fullname" . }}-sync
          {{- with .Values.jobAnnotations }}
          annotations:
            {{- toYaml . | nindent 12 }}
          {{- end }}
        spec:
          restartPolicy: OnFailure
          containers:
            - name: {{ include "fediblockhole.fullname" . }}-sync
              image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
              imagePullPolicy: {{ .Values.image.pullPolicy }}
              command:
                - fediblock-sync
                - -c
                - "{{- include "fediblockhole.conf_file_path" . -}}{{- include "fediblockhole.conf_file_filename" . -}}"
              volumeMounts:
                - name: config
                  mountPath: {{ include "fediblockhole.conf_file_path" . | quote }}
          volumes:
            - name: config
              configMap:
                name: {{ include "fediblockhole.fullname" . }}-conf-toml
                items:
                - key: {{ include "fediblockhole.conf_file_filename" . | quote }}
                  path: {{ include "fediblockhole.conf_file_filename" . | quote }}
{{- end }}
{% endraw %}
```

You can see the values from the [Values file](#values-file) (prefixed with `.Values.`) in the template. An example of where a value from the Values file overrides the default from the Chart is highlighted in the file above.

#### Helpers File

 There are several values in the template (prefixed with `fediblockhole.`) that are not defined in either the Values file or the Chart. These are defined in a [Helpers file](https://helm.sh/docs/chart_template_guide/named_templates/), like this:

```golang title="/chart/templates/_helpers.tpl" hl_lines="66 69"
{% raw %}
{{/* vim: set filetype=mustache: */}}
{{/*
Expand the name of the chart.
*/}}
{{- define "fediblockhole.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
If release name contains chart name it will be used as a full name.
*/}}
{{- define "fediblockhole.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "fediblockhole.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "fediblockhole.labels" -}}
helm.sh/chart: {{ include "fediblockhole.chart" . }}
{{ include "fediblockhole.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "fediblockhole.selectorLabels" -}}
app.kubernetes.io/name: {{ include "fediblockhole.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Rolling pod annotations
*/}}
{{- define "fediblockhole.rollingPodAnnotations" -}}
rollme: {{ .Release.Revision | quote }}
checksum/config-configmap: {{ include ( print $.Template.BasePath "/configmap-conf-toml.yaml" ) . | sha256sum | quote }}
{{- end }}

{{/*
Create the default conf file path and filename
*/}}
{{- define "fediblockhole.conf_file_path" -}}
{{- default "/etc/default/" .Values.fediblockhole.conf_file.path }}
{{- end }}
{{- define "fediblockhole.conf_file_filename" -}}
{{- default "fediblockhole.conf.toml" .Values.fediblockhole.conf_file.filename }}
{{- end }}
{% endraw %}
```

You can see how values from both the [Chart file](#chart-file) and the [Values file](#values-file) are combined to define other values that are then used in the [cron job template](#cron-job).

A slightly different syntax for defining defaults and overrides is highlighted above, showing how the local path and filename for the FediBlockHole configuration has a default value of `/etc/default/fediblockhole.conf.toml` from the [Chart file](#chart-file) if a value is not set in the [Values file](#values-file).

#### Configuration File

As pointed out at the start, FediBlockHole was intended to be deployed on a machine with a persistant storage system, and read a [TOML-formatted](https://toml.io/en/v1.0.0) configuration file from a local filepath. This poses a challenge when containerizing it, as we need to be able to easily make changes to the configuration file and redeploy the cron job with the updated values into the pod's local file system where the script expects to find it. We also want to make sure that the container is rebuilt with the new configuration file when the changes are made to it.

There are a few tricks in the various files that we used to accomplish this. First, we created a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) template, which globs a copy of the default configuration file stored in the root of the chart into a ConfigMap entry:

```golang title="/chart/templates/configmap-conf-toml.yaml"
{% raw %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "fediblockhole.fullname" . }}-conf-toml
  labels:
    {{- include "fediblockhole.labels" . | nindent 4 }}
data:
  {{ (.Files.Glob "fediblockhole.conf.toml").AsConfig | nindent 4 }}
{% endraw %}
```

Next, we included in the container spec a local filesystem mount that mounts the configuration file into the expected location in the container's local filesystem:

```golang title="/chart/templates/cronjob-fediblock-sync.yaml"
{% raw %}
...
spec:
  ...
  containers:
    ...
      volumeMounts:
        - name: config
          mountPath: {{ include "fediblockhole.conf_file_path" . | quote }}
  volumes:
    - name: config
      configMap:
        name: {{ include "fediblockhole.fullname" . }}-conf-toml
        items:
        - key: {{ include "fediblockhole.conf_file_filename" . | quote }}
          path: {{ include "fediblockhole.conf_file_filename" . | quote }}
{% endraw %}
```

Finally, we set the container to [restart when a change to the ConfigMap is detected](https://renehernandez.io/notes/rolling-pods-config-changes/) by [annotating](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) it with the checksum of the ConfigMap:

```golang title="/chart/templates/_helpers.tpl"
{% raw %}
...
{{/*
Rolling pod annotations
*/}}
{{- define "fediblockhole.rollingPodAnnotations" -}}
rollme: {{ .Release.Revision | quote }}
checksum/config-configmap: {{ include ( print $.Template.BasePath "/configmap-conf-toml.yaml" ) . | sha256sum | quote }}
{{- end }}
...
{% endraw %}
```

All this ensures that a new container image is generated with the latest configuration file in the correct location in its local filesystem, each time a change is made to the ConfigMap.

!!! Note
    Remember that references to the ConfigMap in the chart are relative to the repository from which the Helm Release will actually be run. This will be **our** Flux/Helm repository that we [prepared earlier](../fluxhelm/), allowing us to specify our **own** ConfigMap containing **our** version of the configuration file.

### OAuth Token

In order to push blocks to your Mastodon instance, you will need a [Mastodon OAuth token](/operating/mastodon/#mastodon-oauth-token) with `admin:read` and `admin:write` scopes. You will paste the value from the `Your access token` field in the Mastodon `Development` screen into this block of the [FediBlockHole configuration file](#configuration-file) as highlighted here:

```toml hl_lines="3"
# List of instances to write blocklist to
blocklist_instance_destinations = [
  { domain = '{my-web_domain}', token = '[redacted]', max_followed_severity = 'silence'},
]
```

You won't need to set the `Redirect URI` to anything other than the default - it's just the token that FediBlockHole needs.

!!! Danger "Danger, Will Robinson"
    **Protect your FediBlockHole configuration file carefully** once this token is placed in it, and rotate the token regularly. If compromised, it is all anyone will need to have full admin access to your instance via the API.

### Deploying FediBlockHole

Deploying our FediBlockHole cron job into our cluster followed an identical pattern to how we [deployed Mastodon](../mastodon/). We prepared our deployment by creating a GitHub repository source and namespace for our cron job:

```yaml title="/clusters/{my-cluster}/gitrepositories/gitrepository-fediblockhole.yaml"
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: fediblockhole
  namespace: flux-system
spec:
  interval: 5m
  ref:
    branch: main
  url: https://github.com/cunningpike/fediblockhole/
```

```yaml title="/clusters/{my-cluster}/namespaces/namespace-fediblockhole.yaml"
apiVersion: v1
kind: Namespace
metadata:
  name: fediblockhole
```

Next, we created the Flux kustomization:

```yaml title="/clusters/{my-cluster}/kustomizations/kustomization-fediblockhole.yaml"
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: fediblockhole
  namespace: flux-system
spec:
  interval: 5m
  path: ./fediblockhole
  prune: true # remove any elements later removed from the above path
  timeout: 2m # if not set, this defaults to interval duration, which is 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: server
```

Then, we created our own version of the ConfigMap that configures the FediBlockHole cron job itself:

??? Example "/fediblockhole/configmap-fediblockhole-helm-chart-value-overrides.yaml"
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: fediblockhole-helm-chart-value-overrides
      namespace: fediblockhole
    data:
      values.yaml: |-  
        image:
          repository: ghcr.io/cunningpike/fediblockhole
          # https://github.com/cunningpike/fediblockhole/pkgs/container/fediblockhole/versions
          #
          # alternatively, use `latest` for the latest release or `edge` for the image
          # built from the most recent commit
          #
          # tag: latest
          tag: ""
          # use `Always` when using `latest` tag
          pullPolicy: IfNotPresent

        fediblockhole:
          # location of the configuration file. Default is /etc/default/fediblockhole.conf.toml
          conf_file:
            path: ""
            filename: ""
          cron:
            # -- run `fediblock-sync` every hour
            sync:
              # @ignored
              enabled: true
              # @ignored
              schedule: "0 */2 * * *"

        # if you manually change the UID/GID environment variables, ensure these values
        # match:
        podSecurityContext:
          runAsUser: 991
          runAsGroup: 991
          fsGroup: 991

        # @ignored
        securityContext: {}

        # -- Kubernetes manages pods for jobs and pods for deployments differently, so you might
        # need to apply different annotations to the two different sets of pods. The annotations
        # set with podAnnotations will be added to all deployment-managed pods.
        podAnnotations: {}

        # -- The annotations set with jobAnnotations will be added to all job pods.
        jobAnnotations: {}

        # -- Default resources for all Deployments and jobs unless overwritten
        resources: {}
          # We usually recommend not to specify default resources and to leave this as a conscious
          # choice for the user. This also increases chances charts run on environments with little
          # resources, such as Minikube. If you do want to specify resources, uncomment the following
          # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
          # limits:
          #   cpu: 100m
          #   memory: 128Mi
          # requests:
          #   cpu: 100m
          #   memory: 128Mi

        # @ignored
        nodeSelector: {}

        # @ignored
        tolerations: []

        # -- Affinity for all pods unless overwritten
        affinity: {}
    ```

The only changes you need to make in this file are:

- Set `fediblockhole.cron.sync.enabled:` to `true`.
- Set the desired crontab value in `fediblockhole.cron.sync.schedule:`. We started with `"0 */2 * * *"` initially, as shown, but have since reduced that to `"0 * * * *"` as updates are much faster after the initial sync.

After that, we set up our version of the FediBlockHole configuration file:

??? Example "/fediblockhole/configmap-fediblockhole-conf-toml.yaml"
    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: fediblockhole-conf-toml
      namespace: fediblockhole
    data:
      fediblockhole.conf.toml: |-  
        # List of instances to read blocklists from.
        # If the instance makes its blocklist public, no authorization token is needed.
        #   Otherwise, `token` is a Bearer token authorised to read domain_blocks.
        # If `admin` = True, use the more detailed admin API, which requires a token with a 
        #   higher level of authorization.
        # If `import_fields` are provided, only import these fields from the instance.
        #   Overrides the global `import_fields` setting.
        blocklist_instance_sources = [
          # { domain = 'public.blocklist'}, # an instance with a public list of domain_blocks
          # { domain = 'jorts.horse', token = '<a_different_token>' }, # user accessible block list
          # { domain = 'eigenmagic.net', token = '<a_token_with_read_auth>', admin = true }, # admin access required
        ]

        # List of URLs to read csv blocklists from
        # Format tells the parser which format to use when parsing the blocklist
        # max_severity tells the parser to override any severities that are higher than this value
        # import_fields tells the parser to only import that set of fields from a specific source
        blocklist_url_sources = [
          # { url = 'file:///path/to/fediblockhole/samples/demo-blocklist-01.csv', format = 'csv' },
          { url = 'https://codeberg.org/oliphant/blocklists/raw/branch/main/blocklists/_unified_min_blocklist.csv', format = 'csv' },
          { url = 'https://rapidblock.org/blocklist.json', format = 'rapidblock.json' },
        ]

        ## These global allowlists override blocks from blocklists
        # These are the same format and structure as blocklists, but they take precedence
        allowlist_url_sources = [
          # { url = 'https://raw.githubusercontent.com/eigenmagic/fediblockhole/main/samples/demo-allowlist-01.csv', format = 'csv' },
          { url = 'https://codeberg.org/oliphant/blocklists/raw/branch/main/blocklists/__allowlist.csv', format = 'csv' },
        ]

        # List of instances to write blocklist to
        blocklist_instance_destinations = [
          { domain = 'mastodon.govsocial.org', token = '[redacted]', max_followed_severity = 'silence'},
        ]

        ## Store a local copy of the remote blocklists after we fetch them
        #save_intermediate = true

        ## Directory to store the local blocklist copies
        # savedir = '/tmp'

        ## File to save the fully merged blocklist into
        # blocklist_savefile = '/tmp/merged_blocklist.csv'

        ## Don't push blocklist to instances, even if they're defined above
        # no_push_instance = false

        ## Don't fetch blocklists from URLs, even if they're defined above
        # no_fetch_url = false

        ## Don't fetch blocklists from instances, even if they're defined above
        # no_fetch_instance = false

        ## Set the mergeplan to use when dealing with overlaps between blocklists
        # The default 'max' mergeplan will use the harshest severity block found for a domain.
        # The 'min' mergeplan will use the lightest severity block found for a domain.
        # mergeplan = 'max'

        ## Set which fields we import
        ## 'domain' and 'severity' are always imported, these are additional
        ## 
        import_fields = ['public_comment', 'reject_media', 'reject_reports', 'obfuscate']

        ## Set which fields we export
        ## 'domain' and 'severity' are always exported, these are additional
        ## 
        export_fields = ['public_comment']
    ```

More information about how we chose our block list sources can be found [here](/operating/mastodon/#server-moderation).

Finally, we created the Helm Release that deployed the configured cron job into our cluster:

```yaml title="/fediblockhole/helmrelease-fediblockhole.yaml"
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: fediblockhole
  namespace: fediblockhole
spec:
  chart:
    spec:
      chart: ./chart
      sourceRef:
        kind: GitRepository
        name: fediblockhole
        namespace: flux-system
  interval: 15m
  timeout: 5m
  releaseName: fediblockhole
  valuesFrom:
  - kind: ConfigMap
    name: fediblockhole-helm-chart-value-overrides
    valuesKey: values.yaml
```

!!! Note "A Couple of Notes"
    - Rate-limiting (either in our [Ingress](../mastodon/#ingress_1) or in the Mastodon API itself) makes running FediBlockHole on our instance a relatively slow operation, at least for an initial load. YMMV, but we recommend setting your `fediblockhole.cron.sync.schedule:` to a fairly lengthy interval (at least 2 hours) until you get a feel for how long a typical run takes in your environment.
    - FediBlockHole currently does **NOT** delete expired blocks i.e. if an existing block in your instance is no longer in a pulled list, it needs to be removed manually. We are planning to contribute code to the FediBlockProject to implement this feature.
    - v1.0.0 of our Helm chart for FediBlockHole does not include ConfigMaps for the local file import functionality. This will be included in the next release.