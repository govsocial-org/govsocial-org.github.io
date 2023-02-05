---
#template: home.html
title: Building FediBlockHole
---

# Building FediBlockHole

FediBlockHole is a Python tool that expects to be installed on a machine and run from the command line. We could have installed it on our CLI VM and run it from there with a crontab to automate it, but there are a couple of disadvantages with this approach. Firstly, it would require our VM to be more stable than a [spot VM](https://cloud.google.com/compute/docs/instances/spot), increasing our hosting costs. Secondly, the supported VM image in GCP comes with Python 3.9.x, and FediBlockHole requires at least version 3.10.

Rather than embarking on that expedition, we decided to containerize FediBlockHole, and deploy a Kubernetes cron job that instantiates the container and runs the script.

!!! Note
    When we did this, we forked the [FediBlockHole repo](https://github.com/eigenmagic/fediblockhole) into [our own](https://github.com/cunningpike/fediblockhole), as we wanted to submit a [pull request](https://github.com/eigenmagic/fediblockhole/pull/38) for ingrating our work back into the original project. The following documentation is from that perspective, and all repo paths are relative to the root of the forked repo.

## Containerizing FediBlockHole

To prepare for containerizing FediBlockHole, you need to install Docker on your CLI machine from its package manager.

!!! Note
    In a GCP Compute VM, the package is called `docker.io`

Next, prepare a Dockerfile for your Docker container:

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

You should also make sure there is a Python-flavored Docker ignore file:

```golang title="/container/.dockerignore"
Dockerfile
#README.md
*.pyc
*.pyo
*.pyd
__pycache__
```

Next, build your Docker image:

```bash
~$ sudo docker build https://github.com/[user|organization]/fediblockhole.git#main:container --no-cache
```
``` {.bash .no-copy}
Successfully built {container-id}
```

!!! Note
    The `--no-cache` option forces a refresh from the repo rather than using the local cache. This is important for updates.

Using the `{container-id}` from `docker build`, annotate the container with the version of FediBlockHole it was built with (currently `0.4.2`), and `latest`:

```bash
~$ sudo docker image tag {container-id} fediblockhole:latest
~$ sudo docker image tag {container-id} fediblockhole:0.4.2
~$ sudo docker image tag fediblockhole:latest ghcr.io/[user|organization]/fediblockhole:latest
~$ sudo docker image tag fediblockhole:0.4.2 ghcr.io/[user|organization]/fediblockhole:0.4.2
```
