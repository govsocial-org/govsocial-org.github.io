---
#template: home.html
title: Building a FediBlockHole Cron Job
---

# Building a FediBlockHole Cron Job

FediBlockHole is a Python tool that expects to be installed on a machine and run from the command line. We could have installed it on our CLI VM and run it from there with a crontab entry to automate it, but there are a couple of disadvantages with this approach. Firstly, it would require our VM to be more stable than a [spot VM](https://cloud.google.com/compute/docs/instances/spot), increasing our hosting costs. Secondly, the supported VM image in GCP comes with Python 3.9.x, and FediBlockHole requires at least version 3.10.

Rather than embarking on that expedition, we decided to containerize FediBlockHole, and deploy a Kubernetes cron job that instantiates the container and runs the script.

!!! Note
    When we did this, we forked rather than cloning the [FediBlockHole repo](https://github.com/eigenmagic/fediblockhole) into [our own](https://github.com/cunningpike/fediblockhole). We wanted to submit a [pull request](https://github.com/eigenmagic/fediblockhole/pull/38) for ingrating our work back into the original project, and to keep our repo up to date with it. The following documentation is from that perspective. All repo paths are relative to the root of our forked repo.

## Containerizing FediBlockHole

To prepare for containerizing FediBlockHole, we installed Docker on our CLI machine from its package manager.

!!! Note
    In a GCP Compute VM, the package is called `docker.io`

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

## Creating the FediBlockHole Helm Chart

Next, we wanted to leverage the Flux/Helm system [we built earlier](/building/fluxhelm/) to deploy a Kubernetes cron job in our cluster.

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

### Templates

These files are really just metadata for the Helm deployment - the real work is done in the [templates](https://helm.sh/docs/chart_template_guide/getting_started/).

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

You can see the values from the [Values file](#values-file) (prefixed with `.Values.`) throughout the template. An example of where a value from the Values file overrides the default from the Chart is highlighted in the file above.

#### Helpers File

 There are several values in the template (prefixed with `fediblockhole.`) that are not defined in the Values file. These are defined in a [Helpers file](https://helm.sh/docs/chart_template_guide/named_templates/), like this:

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

A slightly different syntax for defining defaults and overrides is highlighted above, showing how the local path and filename for the FediBlockHole configuration has a default value of `/etc/default/fediblockhole.conf.toml` used if another value is not set in the [Values file](#values-file).

#### Configuration File

As pointed out at the start, FediBlockHole was intended to be deployed on a machine with a persistant storage system, and read a [TOML-formatted](https://toml.io/en/v1.0.0) configuration file from a local filepath. This poses a problem when containerizing it, as we need to be able to easily make changes to the configuration file and redeploy the cron job with the updated values into the pod's local file system where the script expects to find it.



