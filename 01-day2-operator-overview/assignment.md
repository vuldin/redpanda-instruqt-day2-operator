---
slug: day2-operator-overview
id: vbe5vp2h1g8s
type: challenge
title: Overview
teaser: This section provides an overview of the initial state of the cluster and
  how this configuration was done, along with an overview of tools we will use throughout
  the lab.
notes:
- type: text
  contents: |-
    This track uses a multi-node Kubernetes cluster provided by k3s. There are 8 VMs total (providing 7 worker nodes and one control plane).

    You will walk through the configuration for Kubernetes, the Operator, Redpanda, networking, and the various tools to interact with this environment.

    Please wait while the VMs are deployed and the Redpanda cluster is started and configured (this takes ~8 minutes without hot start environments).

    ![skate.png](../assets/skate.png)
tabs:
- id: endesrqhzs26
  title: Shell
  type: terminal
  hostname: server
- id: s3lvuzo38s82
  title: Prometheus
  type: service
  hostname: server
  path: /
  port: 9090
- id: jtzoxibj06ju
  title: Grafana
  type: service
  hostname: server
  path: /
  port: 3000
- id: dirqfiicjt00
  title: Producer
  type: terminal
  hostname: server
  cmd: ./produce.sh
difficulty: ""
timelimit: 600
enhanced_loading: null
---
There are two ways to deploy Redpanda in Kubernetes: the operator and the helm chart. This track focused on operator-based deployments. One thing to keep in mind is that the operator actually uses the helm chart under the covers to perform some basic management tasks for Redpanda (so the two deployment methods are more similar than you may think). For detailed information about running Redpanda in Kubernetes, see the [Redpanda documentation](https://docs.redpanda.com/docs/deploy/deployment-option/self-hosted/kubernetes).

This assignment (or challenge if you are familiar with Instruqt) is here to explain how this environment was configured and deployed. There are no required commands; instead you will be given commands to run that will verify the cluster and peripheral environment is in working order.

Expand each section to view details, then click 'Next' at the bottom right once you have gone through all the content.

Tools
===============

The following tools are already installed in your environment:
- `helm`: helps you define, install, and upgrade applications running on Kubernetes. Helm is used to deploy the the Redpanda operator.
- `kubectl`: lets you deploy applications, inspect and manage cluster resources, and view logs in Kubernetes cluster.
- `rpk`: lets you manage your entire Redpanda cluster without the need to run a separate script for each function, as with Apache Kafka. The `rpk` commands handle everything from configuring nodes and kernel tuning to acting as a client to produce and consume data.
- `yq`: a lightweight and portable command-line YAML processor written in Go

Kubernetes
===============

## Node count

There are 8 nodes in the Kubernetes cluster (1 control plane and 7 worker nodes). Since the Redpanda cluster has three brokers, we are only using half of the worker nodes initially. List all nodes in the Kubernetes cluster:

```bash,run
kubectl get nodes
```

Output:
```bash,nocopy
NAME      STATUS   ROLES                  AGE     VERSION
server    Ready    control-plane,master   6m1s    v1.29.0+k3s1
worker4   Ready    <none>                 5m19s   v1.29.0+k3s1
worker5   Ready    <none>                 5m22s   v1.29.0+k3s1
worker1   Ready    <none>                 5m27s   v1.28.6+k3s2
worker6   Ready    <none>                 5m22s   v1.29.0+k3s1
worker2   Ready    <none>                 5m18s   v1.28.6+k3s2
worker3   Ready    <none>                 5m18s   v1.28.6+k3s2
worker7   Ready    <none>                 4m23s   v1.29.0+k3s1
```

## MetalLB

[MetalLB](https://metallb.universe.tf/) is deployed within the cluster to handle assigning IP addresses to LoadBalancer services:

```bash,run
kubectl get all -n metallb-system
```

Output:

```bash,nocopy
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-786f9df989-dvx4r   1/1     Running   0          5m10s
pod/speaker-8q92p                 1/1     Running   0          5m10s
pod/speaker-zxj67                 1/1     Running   0          5m10s
pod/speaker-w7frl                 1/1     Running   0          5m10s
pod/speaker-5f2gl                 1/1     Running   0          5m10s
pod/speaker-l9qb9                 1/1     Running   0          5m10s
pod/speaker-ghrsz                 1/1     Running   0          5m10s
pod/speaker-tkrqr                 1/1     Running   0          5m10s

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.43.27.112   <none>        443/TCP   5m11s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   7         7         7       7            7           kubernetes.io/os=linux   5m10s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           5m10s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-786f9df989   1         1         1       5m10s
```

## Filesystem

While XFS is recommended, ext4 filesystem type also works. These Kubernetes nodes are formatted by default with ext4, so this environment goes with ext4.

Verify the filesystem type:

```bash,run
df -Th | grep "^/dev"
```

Output:

```bash,nocopy
/dev/root      ext4      20G  5.0G   15G  27% /
/dev/sda15     vfat     105M  6.1M   99M   6% /boot/efi
/dev/sdb       ext4     488M   54M  398M  12% /opt/instruqt/bootstrap
```

> Note: Most of our performance tests have been done on XFS up to this point, and we are in the process of expanding our performance tests to include ext4. Initial tests show similar performance across both filesystem types.

## Operator

The operator is responsible for deploying and configuring Redpanda. In production, the operator configuration will likely be checked into git and referenced within a CI/CD pipeline that provides infrastructure as code (IaC).

Check the operator status:

```bash,run
kubectl get redpanda -n redpanda
```

Output:

```bash,nocopy
NAME       READY   STATUS
redpanda   True    Redpanda reconciliation succeeded
```

## Helm

Two different helm charts are used when deploying Redpanda with the operator:

1. The operator chart deploys the operator (this is also called the `redpanda-controller` chart)
2. The operator uses the Redpanda chart internally to handle some tasks related to managing Redpanda. See [this link](https://docs.redpanda.com/current/deploy/deployment-option/self-hosted/kubernetes/k-deployment-overview/#helm-and-redpanda-operator) for more details on how the operator makes use of the helm chart.

It is important to pin the versions of both charts (along with Redpanda itself) in order to ensure compatibility between these components over time. Below are the locations where each of these components are pinned in code:

| Component name | link |
| - | - |
| Redpanda | [link](https://gist.github.com/vuldin/31ab8a3fb0a1cd7f871fd846991fb6d0#file-operator-config-yaml-L10) |
| Operator chart | [link](https://github.com/vuldin/redpanda-instruqt-day2-operator/blob/main/track_scripts/setup-server#L72) |
| Redpanda chart | [link](https://gist.github.com/vuldin/31ab8a3fb0a1cd7f871fd846991fb6d0#file-operator-config-yaml-L7) |

To verify the chart versions used within this environment:

```bash,run
helm list -n redpanda
```

Output:

```bash,nocopy
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
redpanda                redpanda        1               2024-05-30 00:05:11.259705923 +0000 UTC deployed        redpanda-5.7.36 v23.3.10
redpanda-controller     redpanda        1               2024-05-29 23:59:15.630077974 +0000 UTC deployed        operator-0.4.24 v2.1.20-24.1.2
```

The version of the two charts are shown in the "CHART" column.

You can also verify the state of the internal helm release that the operator uses to interface with the Redpanda deployment:

```bash,run
kubectl get helmrelease -n redpanda
```

Output should be similar to the following:

```bash,nocopy
NAME       AGE   READY   STATUS
redpanda   24m   True    Helm install succeeded for release redpanda/redpanda.v1 with chart redpanda@5.7.36
```

Security
===============

## TLS

This deployment uses self-signed certificates that stored in the secret `tls-external`:

```bash,run
kubectl describe secret tls-external -n redpanda
```

## SASL

SASL is enabled, and a single superuser has been created with the credentials "username/password". These credentials are stored in the secret `redpanda-superusers`:

```bash,run
kubectl describe secret redpanda-superusers -n redpanda
```

> Note: Both TLS and SASL are enabled per broker in `/etc/redpanda/redpanda.yaml`:
>
> ```bash,run
> kubectl exec pod/redpanda-0 -c redpanda -n redpanda -- cat /etc/redpanda/redpanda.yaml
> ```

External access
===============

## LoadBalancer

The operator can be configured to create two different services for external access: NodePort and LoadBalancer. This deployment makes use of the LoadBalancer option, which deploys a LoadBalancer per broker. More details on this configuration are in [our docs](https://docs.redpanda.com/current/manage/kubernetes/networking/external/k-loadbalancer/?tab=tabs-1-helm-operator).

Get details on the LoadBalancer services:

```bash,run
kubectl get svc -n redpanda -o wide | grep LoadBalancer
```

Output:

```bash,nocopy
lb-redpanda-0              LoadBalancer   10.43.55.130    172.18.0.13   31644:32751/TCP,31092:31084/TCP,30082:32561/TCP,30081:30432/TCP   52m   app.kubernetes.io/component=redpanda-statefulset,app.kubernetes.io/instance=redpanda,app.kubernetes.io/name=redpanda,statefulset.kubernetes.io/pod-name=redpanda-0
lb-redpanda-2              LoadBalancer   10.43.221.179   172.18.0.12   31644:32084/TCP,31092:32429/TCP,30082:30505/TCP,30081:32015/TCP   52m   app.kubernetes.io/component=redpanda-statefulset,app.kubernetes.io/instance=redpanda,app.kubernetes.io/name=redpanda,statefulset.kubernetes.io/pod-name=redpanda-2
lb-redpanda-1              LoadBalancer   10.43.224.143   172.18.0.11   31644:31178/TCP,31092:32505/TCP,30082:31033/TCP,30081:32334/TCP   52m   app.kubernetes.io/component=redpanda-statefulset,app.kubernetes.io/instance=redpanda,app.kubernetes.io/name=redpanda,statefulset.kubernetes.io/pod-name=redpanda-1
```

## DNS resolution

While you can configure Redpanda to advertise IP addresses to clients, it is recommended to use hostnames. This environment handles DNS resolution of these external hostnames by updating `/etc/hosts`:

```bash,run
tail -3 /etc/hosts
```

Output:

```bash,nocopy
172.18.0.13 redpanda-0.testdomain.local
172.18.0.11 redpanda-1.testdomain.local
172.18.0.12 redpanda-2.testdomain.local
```

> Note: In a production environment, DNS resolution should be handled by a service such as [route53](https://aws.amazon.com/route53/).

Performance
===============

## Tuning nodes

There are two tuning steps to take on the worker nodes Redpanda is assigned to:

1. *autotuner* - An autotuner script has been ran on each Kubernetes node which applies various changes to the operating system kernel. These tuning steps range from increasing the number of simultaneous I/O requests (to maximize throughput) to enabling Transparent Huge Pages (THP) (which improves memory page caching). This autotuner should be ran at each startup, and one way to do this is by using a privileged DaemonSet to run the script on each worker node where Redpanda will run. More details on each tuning step can be found [here](https://docs.redpanda.com/current/reference/rpk/rpk-redpanda/rpk-redpanda-tune-list/#tuners), and our docs for running the autotuner script on Kubernetes nodes can be found [here](https://docs.redpanda.com/current/deploy/deployment-option/self-hosted/kubernetes/k-tune-workers/).
2. *iotune* - Redpanda can be configured to take advantage of available I/O capabilities. Redpanda will try to determine this when running on well-known cloud instance types, but this can also be determined by running `rpk iotune` (which has been performed in this environment). More details on iotune can be found [here](https://docs.redpanda.com/current/reference/rpk/rpk-iotune/).

> Note: The autotuner step should be configured to run at each start of the Kubernetes node, but the iotune step only needs to be done once.

> Note: We are improving the process for tuning Kubernetes nodes, stay tuned for more details.

Redpanda
===============

> Note: This environment was deployed and configured by running a set of scripts, which can be found in [this repo](https://github.com/vuldin/redpanda-instruqt-day2-operator).

The Redpanda cluster in this environment has three brokers. You can verify the broker count, connectivity, as well as view some other basic information related to the cluster with the following command:

```bash,run
rpk cluster info
```

Output:

```bash,nocopy
CLUSTER
----------
redpanda.d05c8102-5ed0-4730-8ab5-29fcc57bc38d

BROKERS
----------
ID    HOST                         PORT
0*    redpanda-0.testdomain.local  31092
1     redpanda-1.testdomain.local  31092
2     redpanda-2.testdomain.local  31092
```

This Redpanda cluster is deployed via the operator and configured out-of-the box in a number of ways that make it possible to run the above command successfully, which is basically a Kafka client making an external connection to Redpanda within the Kubernetes cluster.

## rpk

rpk is the CLI for Redpanda that lets you configure, manage, and tune Redpanda clusters. It also let you manage topics, groups, and access control lists (ACLs). More details on rpk and its capabilities are in [our docs](https://docs.redpanda.com/current/reference/rpk/).

rpk is installed by default wherever Redpanda is installed (ie. on each broker in the cluster). While you could connect to the brokers to use rpk, the approach taken in these instructions is to use a local install of rpk that connects into the remote cluster via the hostnames previously made resolvable to the deployed LoadBalancers. You can verify this configuration by running the following command:

```bash,run
rpk profile print
```

Output:

```bash,nocopy
name: operator
kafka_api:
    brokers:
        - redpanda-0.testdomain.local:31092
        - redpanda-1.testdomain.local:31092
        - redpanda-2.testdomain.local:31092
    tls:
        ca_file: ca.crt
    sasl:
        user: username
        password: password
        mechanism: SCRAM-SHA-256
admin_api:
    addresses:
        - redpanda-0.testdomain.local:31644
        - redpanda-1.testdomain.local:31644
        - redpanda-2.testdomain.local:31644
    tls:
        ca_file: ca.crt
```

The configuration for connecting to this cluster is stored in the "operator" profile shown above. You can create multiple profiles within rpk, and switch between them to connect to various clusters. More details on rpk profiles [here](https://docs.redpanda.com/current/reference/rpk/rpk-profile/rpk-profile/).

Monitoring
===============

## Prometheus

Redpanda provides prometheus metrics through endpoints on the admin API. These instructions focus on the recommended `/public_metrics` endpoint. More details on how to access this metric and definitions for each metric available are [here](https://docs.redpanda.com/current/reference/public-metrics-reference/). You can see the output from this endpoint with curl:

```bash,run
curl redpanda-0.testdomain.local:31644/public_metrics
```

Prometheus must be configured to scrape this endpoint. The helm chart is a good tool for handling both the deployment and configuration in Kubernetes. The most important Prometheus configuration found in `values.yaml` is the following section:

```bash,nocopy
serverFiles:
  prometheus.yml:
    global:
      scrape_interval: 10s
      evaluation_interval: 10s
    scrape_configs:
    - job_name: redpanda
      static_configs:
      - targets:
        - redpanda-0.redpanda.redpanda.svc.cluster.local:9644
        - redpanda-1.redpanda.redpanda.svc.cluster.local:9644
        - redpanda-2.redpanda.redpanda.svc.cluster.local:9644
      metrics_path: /public_metrics
```

The above section configures Prometheus to scrape from the internal admin port on each of the Redpanda brokers. A full `values.yaml` file can be found [here](https://gist.github.com/vuldin/9e0ed0724eff56dd194e500f92a2dc77#file-values-prometheus-yaml-L206-L212).

## Grafana

Grafana has the Prometheus server configured as a data source to drive dashboards that we provide, which are automatically loaded at deployment time. The critical section for connecting Grafana to Prometheus is:

```bash,nocopy
datasources:
 datasources.yaml:
   apiVersion: 1
   datasources:
   - name: Prometheus
     type: prometheus
     url: http://prometheus-server.prometheus.svc.cluster.local
     access: proxy
     isDefault: true
```

We provide Grafana dashboards in our [observability repo](https://github.com/redpanda-data/observability/tree/main/grafana-dashboards). These are an excellent starting point for monitoring Redpanda, and can help even if you plan to use some other visualization tool. Each chart has a query or expression that makes use of one or more metrics, which provides an example that can be ported over to other tools if needed. Also, dashboards can be loaded into Grafana at deploy time by adding [the proper configuration](https://gist.github.com/vuldin/e579c4ffca46db95d6209391d943862f#file-values-grafana-yaml-L17-L25) to the `values.yaml` file.


You have reached the end of the cluster overview assignment. Please click the 'Next' button below to continue.
