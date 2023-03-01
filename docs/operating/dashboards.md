---
#template: home.html
title: Installation
---

# Operational Dashboards

A large part of providing a stable Fediverse platform is comprehensive operational monitoring. In common with many Kubernetes-based instances, we picked [Prometheus](https://prometheus.io/) to scrape operational metrics, and [Grafana](https://grafana.com/) to display them in a series of dashboards. We also wanted to publish these dashboards to meet the principle of ["Default to Open"](https://playbook.cio.gov/#play13) from the [US Digital Service](https://www.usds.gov/) [Playbook](https://playbook.cio.gov/).

As explained below, the Grafana `publicDashboards` [feature](https://grafana.com/docs/grafana/latest/dashboards/dashboard-public/) is in alpha at the moment, and does not support dashboards that contain template variables (which is most of them!). Until that becomes possible, we are committing to providing monthly screenshots of our operational metrics.

## Installing Prometheus and Grafana

The [GKE Marketplace](https://cloud.google.com/marketplace) provides two products, "Prometheus & Grafana", which (despite its name) provides a Grafana-ready implementation of Prometheus 2.41 only, and "Grafana", which provides Grafana 7.4.

### Prometheus

The Marketplace install of Prometheus is reasonably current, so we deployed that. No configuration changes were needed for our implementation.

!!! Note
    If you plan to deploy an Ingress in front of Prometheus (we did not, using [Port Forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) instead), the provided values of `timeoutSeconds: 30` for both the `livenessProbe` and `readinessProbe` will need to be reduced to below 30 in the YAML for the StatefulSet to avoid the GKE strict implementation of `TimeoutSec should be less than checkIntervalSec`. The default value of `checkIntervalSec` in GKE is 30 seconds, and `timeoutSeconds` must be less than that[^1].

One of the advantages of the GKE Marketplace Prometheus implementation is that it comes with a Google Cloud Monitoring datasource already configured. This datasource comes with some excellent pre-defined dashboards for Cloud SQL, Cloud Storage, and Load Balancers.

Most of the exporters in the GKE-provided Prometheus worked just fine as installed, but we encountered a wrinkle with the [cAdvisor](https://github.com/google/cadvisor) exporter. We also needed to add a [Statsd exporter](https://github.com/prometheus/statsd_exporter) in order for our Mastodon dashboard to work.

#### GKE cAdvisor

When we installed the GKE Marketplace Prometheus application, the GKE cAdvisor didn't emit any data as initially configured. After a *lot* of trouble-shooting, we eventually found this little nugget in the `prometheus-1-prometheus-config` ConfigMap:

```yaml hl_lines="10-18"
apiVersion: v1
data:
  ...
  prometheus.yaml: |
    ...
    "scrape_configs":
    ...
    - "job_name": "gke-cadvisor"
      ...
      "metric_relabel_configs":
      - "action": "drop"
        "regex": "^$"
        "source_labels":
        - "namespace"
      - "action": "drop"
        "regex": "^$"
        "source_labels":
        - "pod_name"
```

The effect of this was to effectively silence the exporter, meaning that, while the exporter itself appeared healthy, it wasn't sending any data. Removing the entire `metric_relabel_configs` block (highlighted above) solved that problem[^2].

#### Statsd Exporter

At least one Fediverse platform, Mastodon, has a [built-in StatsD exporter](https://docs.joinmastodon.org/admin/config/#statsd). What we needed was a way to convert the metrics from the StatsD format to Prometheus format. The [Prometheus Community](https://github.com/prometheus-community) GitHub repo publishes a [Helm chart for their statsd-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-statsd-exporter), which does exactly that. This we installed in [the usual way](/building/fediblockhole/#deploying-fediblockhole).

### Grafana

 The Marketplace version of Grafana is too old to support the dashboards we wanted to use. Instead, we installed Grafana 9.3 from the [Bitnami chart](https://github.com/bitnami/charts/tree/main/bitnami/grafana). The method we used is identical to how we [installed FediBlockHole](/building/fediblockhole/#deploying-fediblockhole), so we won't repeat it here. The only configuration changes we made were:

```yaml title="/grafana/configmap-grafana-helm-chart-value-overrides.yaml"
data:
  values.yaml: |- 
     grafana:
       updateStrategy:
        type: Recreate
       extraEnvVars:
        - name: GF_FEATURE_TOGGLES_ENABLE
          value: publicDashboards
```

The change to `updateStrategy.type` is to remove the multi-mount contention on the Persistent Volume claim that a `RollingUpdate` type encounters with the `standard-rwo` StorageClass in a Kubernetes Deployment.

!!! Warning
    This change means that the Grafana service will be briefly unavailable during pod updates. We determined we could live with it. YMMV.

The `extraEnvVars` setting enables the `publicDashboards` [feature](https://grafana.com/docs/grafana/latest/dashboards/dashboard-public/), which is currently in alpha release. We intended to publish our Grafana dashboards as part of this site, but all the dashboards we use use template variables, which are not currently supported. Until they are, we will publish monthly screenshots instead.

We also created a DNS entry and [HTTPS Ingress](https://cloud.google.com/kubernetes-engine/docs/tutorials/http-balancer) for our Grafana install, so we could access it from the Internet.

When you get Grafana running, you will be presented with a login screen. Because we implemented in GKE from a Helm chart which creates the `admin` user and generates a secret for it, we had no idea what it was, so we extracted it from the secret with this command:

```bash
~$ kubectl get secret --namespace grafana grafana-admin -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode ; echo
```

Once that's done, you should be able to login to your Grafana instance and start dashboarding!

[^1]: It's kinda bizarre that Google didn't fix this in their Marketplace offering...
[^2]: Another bizarre configuration choice in the Marketplace install...