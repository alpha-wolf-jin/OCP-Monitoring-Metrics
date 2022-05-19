# OCP-Monitoring-Metrics

**Git**
```
echo "# OCP-Monitoring-Metrics" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/alpha-wolf-jin/OCP-Monitoring-Metrics.git
git config --global credential.helper 'cache --timeout 7200'
git push -u origin main

git add . ; git commit -a -m "update README" ; git push -u origin main
```

## Listing the Cluster Monitoring Stack Components

**Cluster Monitoring Operator**
The cluster monitoring operator (CMO) is the central component of the monitoring stack. It controls the deployed monitoring components and ensures that they are always in sync with the latest version of the CMO.

**Prometheus Operator**
The Prometheus operator deploys and configures both Prometheus and Alertmanager. The operator also manages the generation and configuration of configuration targets (service monitors and pod monitors).

**Prometheus**
Prometheus is the monitoring server.

**Prometheus Adapter**
The Prometheus adapter exposes cluster resources that are used for Horizontal Pod Autoscaling (HPA).

**Alertmanager**
Alertmanager handles alerts sent by the Prometheus server.

**Kube state metrics**
kube-state-metrics is a converter agent that exports Kubernetes objects to metrics that Prometheus can parse.

**OpenShift state metrics**
openshift-state-metrics is based on the kube-state-metrics and adds monitoring for OpenShift-specific resources (such as image registry metrics).

**Node exporter**
node-exporter exports low-level metrics for worker nodes.

**Thanos Querier**
Thanos Querier is a single, multitenant interface that enables aggregating and deduplicating cluster and user workload metrics.

**Grafana**
Grafana is a platform for analyzing and visualizing metrics. The Grafana dashboards provided with the monitoring stack are read-only.

## Managing OpenShift Alerts

--    Configure Alertmanager to send email alerts.
--    Review default alerts.
--    Silence a firing alert.
