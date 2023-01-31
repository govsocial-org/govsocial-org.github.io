---
#template: home.html
title: PostgreSQL
---

# PostgreSQL

As with S3-compatible storage, most Fediverse platforms require a PostgreSQL database instance where much of the instance data is stored. Despite the importance of the performance of the database to that of the overall instance, you can save a lot of up-front cost by starting small and cloning to a larger machine as you grow.

We use a Google CloudSQL PostgreSQL instance, selecting the smallest possible machine (`db-f1-micro`) and set the storage to `auto-grow`.

You will need to ensure that you enable the [Private IP](https://cloud.google.com/sql/docs/postgres/connect-instance-private-ip#create-instance) interface for the instance so the GKE cluster can reach it. We did not enable the [CloudSQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy) - we may eventually do that at some point.

We chose to create a separate database and user for each platform, rather than piling everything into the default `postgres` one.