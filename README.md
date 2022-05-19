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
> https://prometheus.io/docs/alerting/latest/configuration/
> https://www.pagerduty.com/docs/guides/prometheus-integration-guide/
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


- From the OpenShift web console, navigate to **`Monitoring → Alerting`**
- Click the **`TestAlert`** link to get details
- Click Silence Alert to silence the alert.
- Navigate to **`Monitoring → Alerting`** in the web console to verify that the TestAlert alert is no longer firing.

```
$ sed -i 's/warning/critical/' /tmp/alertmanager.yaml

$ oc set data secret/alertmanager-main -n openshift-monitoring --from-file /tmp/alertmanager.yaml

$ oc project openshift-monitoring
Now using project "openshift-monitoring" on server "https://api.ocp4.example.com:6443".
[student@workstation monitor]$ oc get all
NAME                                               READY   STATUS    RESTARTS   AGE
pod/alertmanager-main-0                            5/5     Running   0          183d
pod/alertmanager-main-1                            5/5     Running   0          183d
pod/alertmanager-main-2                            5/5     Running   0          183d
pod/cluster-monitoring-operator-78cd56b95d-sjk86   2/2     Running   0          183d
pod/grafana-f9c9ccb65-zz965                        2/2     Running   0          183d
pod/kube-state-metrics-5db8c78f5f-mpcbg            3/3     Running   0          183d
pod/node-exporter-25v9w                            2/2     Running   0          183d
pod/node-exporter-2cvgl                            2/2     Running   0          183d
pod/node-exporter-87jlh                            2/2     Running   0          183d
pod/node-exporter-hh8zm                            2/2     Running   0          183d
pod/node-exporter-l98xb                            2/2     Running   0          183d
pod/node-exporter-lp52j                            2/2     Running   0          183d
pod/openshift-state-metrics-7cf4dc694b-kjx8f       3/3     Running   0          183d
pod/prometheus-adapter-566c44f955-6thkr            1/1     Running   0          105m
pod/prometheus-adapter-566c44f955-vlwv5            1/1     Running   0          105m
pod/prometheus-k8s-0                               6/6     Running   0          183d
pod/prometheus-k8s-1                               6/6     Running   0          183d
pod/prometheus-operator-7ccb887947-sp4cr           2/2     Running   0          183d
pod/thanos-querier-5997fbd4bd-f6m9n                5/5     Running   0          183d
pod/thanos-querier-5997fbd4bd-g2ckq                5/5     Running   0          183d

NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main             ClusterIP   172.30.188.235   <none>        9094/TCP,9092/TCP            183d
service/alertmanager-operated         ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   183d
service/cluster-monitoring-operator   ClusterIP   None             <none>        8443/TCP                     183d
service/grafana                       ClusterIP   172.30.98.162    <none>        3000/TCP                     183d
service/kube-state-metrics            ClusterIP   None             <none>        8443/TCP,9443/TCP            183d
service/node-exporter                 ClusterIP   None             <none>        9100/TCP                     183d
service/openshift-state-metrics       ClusterIP   None             <none>        8443/TCP,9443/TCP            183d
service/prometheus-adapter            ClusterIP   172.30.209.97    <none>        443/TCP                      183d
service/prometheus-k8s                ClusterIP   172.30.68.1      <none>        9091/TCP,9092/TCP            183d
service/prometheus-operated           ClusterIP   None             <none>        9090/TCP,10901/TCP           183d
service/prometheus-operator           ClusterIP   None             <none>        8443/TCP,8080/TCP            183d
service/thanos-querier                ClusterIP   172.30.191.9     <none>        9091/TCP,9092/TCP,9093/TCP   183d

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   6         6         6       6            6           kubernetes.io/os=linux   183d

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cluster-monitoring-operator   1/1     1            1           183d
deployment.apps/grafana                       1/1     1            1           183d
deployment.apps/kube-state-metrics            1/1     1            1           183d
deployment.apps/openshift-state-metrics       1/1     1            1           183d
deployment.apps/prometheus-adapter            2/2     2            2           183d
deployment.apps/prometheus-operator           1/1     1            1           183d
deployment.apps/thanos-querier                2/2     2            2           183d

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/cluster-monitoring-operator-78cd56b95d   1         1         1       183d
replicaset.apps/grafana-7ff876c957                       0         0         0       183d
replicaset.apps/grafana-f9c9ccb65                        1         1         1       183d
replicaset.apps/kube-state-metrics-5db8c78f5f            1         1         1       183d
replicaset.apps/openshift-state-metrics-7cf4dc694b       1         1         1       183d
replicaset.apps/prometheus-adapter-566c44f955            2         2         2       105m
replicaset.apps/prometheus-adapter-7d4cb9df7             0         0         0       183d
replicaset.apps/prometheus-operator-6b568cc478           0         0         0       183d
replicaset.apps/prometheus-operator-7ccb887947           1         1         1       183d
replicaset.apps/thanos-querier-5997fbd4bd                2         2         2       183d
replicaset.apps/thanos-querier-6d4dbcb564                0         0         0       183d

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   3/3     183d
statefulset.apps/prometheus-k8s      2/2     183d

NAME                                         HOST/PORT                                                      PATH   SERVICES            PORT    TERMINATION          WILDCARD
route.route.openshift.io/alertmanager-main   alertmanager-main-openshift-monitoring.apps.ocp4.example.com          alertmanager-main   web     reencrypt/Redirect   None
route.route.openshift.io/grafana             grafana-openshift-monitoring.apps.ocp4.example.com                    grafana             https   reencrypt/Redirect   None
route.route.openshift.io/prometheus-k8s      prometheus-k8s-openshift-monitoring.apps.ocp4.example.com             prometheus-k8s      web     reencrypt/Redirect   None
route.route.openshift.io/thanos-querier      thanos-querier-openshift-monitoring.apps.ocp4.example.com             thanos-querier      web     reencrypt/Redirect   None
[student@workstation monitor]$ 

```

> https://role.rhu.redhat.com/rol-rhu/app/courses/do380-4.6/pages/ch09s03

## Using the Cluster Monitoring Stack

Navigate to **`Monitoring → Dashboards`**

Select the `Kubernetes/Compute Resources/Cluster` dashboard.

Scroll to the **`CPU Quota`** section, and then click the **`CPU Usage`** column header
