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

Scroll to the **`Requests by Namespace`** section and then click the **`Memory Usage`** column header

Select the **`Kubernetes/Compute Resources/Namespace (Workloads)`** dashboard. Select the `monitor-troubleshoot` namespace and leave `deployment` as the workload type.

## Configuring Storage for the Cluster Monitoring Stack

**Determine if your cluster already contains a configuration map for the cluster monitoring operator**
```
$ oc get configmap cluster-monitoring-config   -n openshift-monitoring
Error from server (NotFound): configmaps "cluster-monitoring-config" not found

$ oc get operators.operators.coreos.com -A
No resources found
[student@workstation monitor]$ oc get pod -n openshift-monitoring 
NAME                                           READY   STATUS    RESTARTS   AGE
alertmanager-main-0                            5/5     Running   0          183d
alertmanager-main-1                            5/5     Running   0          183d
alertmanager-main-2                            5/5     Running   0          183d
cluster-monitoring-operator-78cd56b95d-sjk86   2/2     Running   0          183d
grafana-f9c9ccb65-zz965                        2/2     Running   0          183d
kube-state-metrics-5db8c78f5f-mpcbg            3/3     Running   0          183d
node-exporter-25v9w                            2/2     Running   0          183d
node-exporter-2cvgl                            2/2     Running   0          183d
node-exporter-87jlh                            2/2     Running   0          183d
node-exporter-hh8zm                            2/2     Running   0          183d
node-exporter-l98xb                            2/2     Running   0          183d
node-exporter-lp52j                            2/2     Running   0          183d
openshift-state-metrics-7cf4dc694b-kjx8f       3/3     Running   0          183d
prometheus-adapter-566c44f955-6thkr            1/1     Running   0          3h56m
prometheus-adapter-566c44f955-vlwv5            1/1     Running   0          3h56m
prometheus-k8s-0                               6/6     Running   0          183d
prometheus-k8s-1                               6/6     Running   0          183d
prometheus-operator-7ccb887947-sp4cr           2/2     Running   0          183d
thanos-querier-5997fbd4bd-f6m9n                5/5     Running   0          183d
thanos-querier-5997fbd4bd-g2ckq                5/5     Running   0          183d
[student@workstation monitor]$ 

$ oc exec -it prometheus-k8s-0 -c prometheus -n openshift-monitoring -- ls -l /prometheus
total 20
drwxr-sr-x. 3 nobody nobody    68 May 19 05:00 01G3DAZQ9V9BVRB6645EJVYD8G
drwxr-sr-x. 3 nobody nobody    68 May 19 07:00 01G3DHVEHT1SEBWPH45XCK29K3
drwxr-sr-x. 2 nobody nobody    34 May 19 07:00 chunks_head
-rw-r--r--. 1 nobody nobody 20001 May 19 07:35 queries.active
drwxr-sr-x. 3 nobody nobody    81 May 19 07:00 wal

$ oc exec -it prometheus-k8s-0 -c prometheus -n openshift-monitoring -- df -h /prometheus
Filesystem                            Size  Used Avail Use% Mounted on
/dev/mapper/coreos-luks-root-nocrypt   40G  9.5G   31G  24% /prometheus

```
> NFS Storage Class
```
[student@workstation monitor]$ oc get all -n nfs-client-provisioner
NAME                                          READY   STATUS    RESTARTS   AGE
pod/nfs-client-provisioner-7cd58d5569-p72n7   1/1     Running   0          183d

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           183d

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-client-provisioner-7cd58d5569   1         1         1       183d
[student@workstation monitor]$ oc get deployment.apps/nfs-client-provisioner -n nfs-client-provisioner -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-11-16T21:12:14Z"
  generation: 1
  labels:
    app: nfs-client-provisioner
  managedFields:
  - apiVersion: apps/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:labels:
          .: {}
          f:app: {}
      f:spec:
        f:progressDeadlineSeconds: {}
        f:replicas: {}
        f:revisionHistoryLimit: {}
        f:selector:
          f:matchLabels:
            .: {}
            f:app: {}
        f:strategy:
          f:type: {}
        f:template:
          f:metadata:
            f:labels:
              .: {}
              f:app: {}
          f:spec:
            f:containers:
              k:{"name":"nfs-client-provisioner"}:
                .: {}
                f:env:
                  .: {}
                  k:{"name":"NFS_PATH"}:
                    .: {}
                    f:name: {}
                    f:value: {}
                  k:{"name":"NFS_SERVER"}:
                    .: {}
                    f:name: {}
                    f:value: {}
                  k:{"name":"PROVISIONER_NAME"}:
                    .: {}
                    f:name: {}
                    f:value: {}
                f:image: {}
                f:imagePullPolicy: {}
                f:name: {}
                f:resources: {}
                f:terminationMessagePath: {}
                f:terminationMessagePolicy: {}
                f:volumeMounts:
                  .: {}
                  k:{"mountPath":"/persistentvolumes"}:
                    .: {}
                    f:mountPath: {}
                    f:name: {}
            f:dnsPolicy: {}
            f:restartPolicy: {}
            f:schedulerName: {}
            f:securityContext: {}
            f:serviceAccount: {}
            f:serviceAccountName: {}
            f:terminationGracePeriodSeconds: {}
            f:volumes:
              .: {}
              k:{"name":"nfs-client-root"}:
                .: {}
                f:name: {}
                f:nfs:
                  .: {}
                  f:path: {}
                  f:server: {}
    manager: OpenAPI-Generator
    operation: Update
    time: "2021-11-16T21:12:14Z"
  - apiVersion: apps/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:deployment.kubernetes.io/revision: {}
      f:status:
        f:availableReplicas: {}
        f:conditions:
          .: {}
          k:{"type":"Available"}:
            .: {}
            f:lastTransitionTime: {}
            f:lastUpdateTime: {}
            f:message: {}
            f:reason: {}
            f:status: {}
            f:type: {}
          k:{"type":"Progressing"}:
            .: {}
            f:lastTransitionTime: {}
            f:lastUpdateTime: {}
            f:message: {}
            f:reason: {}
            f:status: {}
            f:type: {}
        f:observedGeneration: {}
        f:readyReplicas: {}
        f:replicas: {}
        f:updatedReplicas: {}
    manager: kube-controller-manager
    operation: Update
    time: "2022-05-19T03:36:02Z"
  name: nfs-client-provisioner
  namespace: nfs-client-provisioner
  resourceVersion: "63213"
  selfLink: /apis/apps/v1/namespaces/nfs-client-provisioner/deployments/nfs-client-provisioner
  uid: 14d07c0c-9f1d-4e98-9f5e-ba7a9dedb688
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nfs-client-provisioner
    spec:
      containers:
      - env:
        - name: PROVISIONER_NAME
          value: k8s-sigs.io/nfs-subdir-external-provisioner
        - name: NFS_SERVER
          value: 192.168.50.254
        - name: NFS_PATH
          value: /exports-ocp4
        image: quay.io/redhattraining/nfs-subdir-external-provisioner:v4.0.2
        imagePullPolicy: IfNotPresent
        name: nfs-client-provisioner
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /persistentvolumes
          name: nfs-client-root
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: nfs-client-provisioner
      serviceAccountName: nfs-client-provisioner
      terminationGracePeriodSeconds: 30
      volumes:
      - name: nfs-client-root
        nfs:
          path: /exports-ocp4
          server: 192.168.50.254
status:
  availableReplicas: 1
  conditions:
  - lastTransitionTime: "2021-11-16T21:12:14Z"
    lastUpdateTime: "2021-11-16T21:12:21Z"
    message: ReplicaSet "nfs-client-provisioner-7cd58d5569" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2022-05-19T03:36:02Z"
    lastUpdateTime: "2022-05-19T03:36:02Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 1
  readyReplicas: 1
  replicas: 1
  updatedReplicas: 1

$ oc get sc nfs-storage -o yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  creationTimestamp: "2021-11-16T21:12:19Z"
  managedFields:
  - apiVersion: storage.k8s.io/v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:storageclass.kubernetes.io/is-default-class: {}
      f:parameters:
        .: {}
        f:archiveOnDelete: {}
      f:provisioner: {}
      f:reclaimPolicy: {}
      f:volumeBindingMode: {}
    manager: OpenAPI-Generator
    operation: Update
    time: "2021-11-16T21:12:19Z"
  name: nfs-storage
  resourceVersion: "22881"
  selfLink: /apis/storage.k8s.io/v1/storageclasses/nfs-storage
  uid: cbafc01e-26b0-4abe-abae-ff86d32f35f7
parameters:
  archiveOnDelete: "false"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate

```
### Configure the cluster monitoring operator so that both Prometheus and Alertmanager use persistent storage

```
$ vim persistent-storage.yml
prometheusK8s:
  retention: 15d
  volumeClaimTemplate:
    spec:
      storageClassName: nfs-storage
      resources:
        requests:
          storage: 40Gi
alertmanagerMain:
  volumeClaimTemplate:
    spec:
      storageClassName: nfs-storage
      resources:
        requests:
          storage: 20Gi

$ oc create configmap cluster-monitoring-config -n openshift-monitoring --from-file config.yaml=persistent-storage.yml

$ oc get persistentvolumeclaim -l app=prometheus -n openshift-monitoring
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-8a1cfbc3-906a-4a83-a4b1-c0c14cc155d4   40Gi       RWO            nfs-storage    36s
prometheus-k8s-db-prometheus-k8s-1   Bound    pvc-9ad0ebed-ae60-493e-b2cc-698a30696aac   40Gi       RWO            nfs-storage    36s

$ oc exec -it prometheus-k8s-0 -c prometheus -n openshift-monitoring -- df -h /prometheus
Filesystem                                                                                                                                   Size  Used Avail Use% Mounted on
192.168.50.254:/exports-ocp4/openshift-monitoring-prometheus-k8s-db-prometheus-k8s-0-pvc-8a1cfbc3-906a-4a83-a4b1-c0c14cc155d4/prometheus-db   40G  721M   40G   2% /prometheus

$ oc get persistentvolumeclaims  -l app=alertmanager -n openshift-monitoring
NAME                                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
alertmanager-main-db-alertmanager-main-0   Bound    pvc-be6a3fd3-0fb1-4b4f-89e0-026e0852da83   20Gi       RWO            nfs-storage    2m11s
alertmanager-main-db-alertmanager-main-1   Bound    pvc-cd78d642-b137-4c81-af34-3b6c5c914b75   20Gi       RWO            nfs-storage    2m11s
alertmanager-main-db-alertmanager-main-2   Bound    pvc-fb217117-f23a-47dc-9be7-8af0153cbb31   20Gi       RWO            nfs-storage    2m11s

$ oc exec -it alertmanager-main-0 -c alertmanager -n openshift-monitoring -- df -h /alertmanager
Filesystem                                                                                                                                           Size  Used Avail Use% Mounted on
192.168.50.254:/exports-ocp4/openshift-monitoring-alertmanager-main-db-alertmanager-main-0-pvc-be6a3fd3-0fb1-4b4f-89e0-026e0852da83/alertmanager-db   40G  728M   40G   2% /alertmanager


```
