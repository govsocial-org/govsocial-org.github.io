---
#template: home.html
title: Building GOVSocial
---

# Building GOVSocial

When we first began thinking of hosting Fediverse platforms for public service use, one of our first considerations was whether we would self-host or host in the cloud, and whether we wanted to deploy on servers or in a containerized environment.

We wanted to start small, and scale quickly as needed without service interruption. We started the project in November 2022, just as the first wave of refugees from commercial platforms started, and were keenly aware of the struggles of existing instances to rapidly scale to meet demand and to control the escalating hosting costs of a large influx of users.

This need for scalable reliability and control over our hosting costs led us to pick a Kubernetes deployment hosted in a major cloud service provider. This met the [US Digital Service](https://www.usds.gov/) [Playbook](https://playbook.cio.gov/) principles of ["Choose a modern technology stack"](https://playbook.cio.gov/#play8) and ["Deploy in a flexible hosting environment"](https://playbook.cio.gov/#play9).

We picked Google primarily because they are committed to [open standards](https://opensource.google/) and have a proven track record of support for the [public sector](https://cloud.google.com/blog/topics/public-sector).

We set out to prove that we could implement our entire platform using GCP services. As you will see, we managed that with the sole exception of transactional email, as Google does not currently provide such a service, at least at any sort of scale.

The project is indebted to two excellent guides on hosting our first instance, [Mastodon](https://mastodon.govsocial.org/about/), on Kubernetes and on Google Cloud Platform (GCP):

- [The Funky Penguin's Geek Cookbook](https://geek-cookbook.funkypenguin.co.nz/recipes/kubernetes/mastodon/)
- [Justin Ribeiro's guide to installing Mastodon on GCP](https://justinribeiro.com/chronicle/2019/09/27/setting-up-mastodon-on-google-cloud-platform/)

We made quite a few necessary modifications to both of these, and learned a ton along the way. We have documented our journey here to assist others interested in a similar implementation.