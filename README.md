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

-    Configure Alertmanager to send email alerts.
-    Review default alerts.
-    Silence a firing alert.

**Extract the existing alertmanager-main secret from the openshift-monitoring namespace to the /tmp directory**
```
$ oc extract secret/alertmanager-main --to /tmp/ -n openshift-monitoring --confirm
/tmp/alertmanager.yaml

$ cat /tmp/alertmanager.yaml 
"global":
  "resolve_timeout": "5m"
"receivers":
- "name": "null"
"route":
  "group_by":
  - "namespace"
  "group_interval": "5m"
  "group_wait": "30s"
  "receiver": "null"
  "repeat_interval": "12h"
  "routes":
  - "match":
      "alertname": "Watchdog"
    "receiver": "null"

```
**The default alertmanager-main secret contains many unnecessary quote**

```
$ sed -i 's/"//g' /tmp/alertmanager.yaml
```

**Modify the /tmp/alertmanager.yaml file**

```
$ vim /tmp/alertmanager.yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 192.168.50.254:25
  smtp_from: alerts@ocp4.example.com
  smtp_require_tls: false
receivers:
- name: default
- name: email-notification
  email_configs:
    - to: ocp-admins@example.com
route:
  group_by:
  - namespace
  group_interval: 5m
  group_wait: 30s
  receiver: default
  repeat_interval: 1m
  routes:
  - match:
      alertname: Watchdog
    receiver: default
  - match:
      severity: warning
    receiver: email-notification

```

**Update the existing alertmanager-main secret in the openshift-monitoring namespace**

```
$ oc set data secret/alertmanager-main -n openshift-monitoring --from-file /tmp/alertmanager.yaml

$ oc logs -f -c alertmanager alertmanager-main-0 -n openshift-monitoring
...
level=info ts=2022-05-19T04:50:21.490Z caller=coordinator.go:131 component=configuration msg="Completed loading of configuration file" file=/etc/alertmanager/config/alertmanager.yaml

```

**Verification**
```
$ ssh lab@utility.lab.example.com
[lab@utility ~]$ mutt

q:Quit  d:Del  u:Undel  s:Save  m:Mail  r:Reply  g:Group  ?:Help
   1  D  May 19 alerts@ocp4.exa ( 203) [FIRING:2]  (openshift-monitoring/k8s war
   2 N   May 19 alerts@ocp4.exa ( 219)
   3 N   May 19 alerts@ocp4.exa ( 167) [FIRING:1]  (TestAlert openshift-monitori

```


- From the OpenShift web console, navigate to `**Monitoring → Alerting**`
