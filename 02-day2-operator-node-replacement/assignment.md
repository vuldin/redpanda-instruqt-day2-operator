---
slug: day2-operator-node-replacement
id: rj0u3ktsyutz
type: challenge
title: Replacing nodes and decommissioning Redpanda
teaser: This section goes through the process for replacing Kubernetes nodes under
  a running Redpanda cluster.
notes:
- type: text
  contents: |-
    Now we continue to the first cluster operation: replacing Kubernetes nodes (and decommissioning Redpanda).

    ![skate.png](../assets/skate.png)
tabs:
- id: 156pxx5na22q
  title: Shell
  type: terminal
  hostname: server
difficulty: ""
timelimit: 600
enhanced_loading: null
---
This scenario focuses on replacing Kubernetes nodes under a running Redpanda cluster. This process involves configuring taints/tolerations on the replacement nodes, updating the operator and StatefulSet, setting up an additional broker to ensure replica factor is maintained throughout the process, and then deleting/decommissioning the original pods while managing the PersistentVolumes.

Changing the broker count should involve a number of considerations:

- Availability: do you have enough brokers to span across all racks and/or availability zones?
- Cost: infrastructure costs will be impacted by a change in broker count
- Data retention: storage capacity and possible retention values are determined in large part by the local disk capacity across all brokers
- Durability: you should have more brokers than your lowest partition replication factor
- Partition count: this value is determined primarily by the CPU core count of the overall cluster

See [our docs](https://docs.redpanda.com/current/manage/kubernetes/decommission-brokers/) for more details on the above considerations.

One of the most common reasons for replacing Kubernetes nodes is upgrading Kubernetes, which is done by replacing the nodes Redpanda pods run on with new ones that have an updated Kubernetes version. This environment has two sets of nodes in the cluster:

```bash,run
kubectl get nodes
```

Output:

```bash,nocopy
NAME      STATUS   ROLES                  AGE     VERSION
worker6   Ready    <none>                 5m33s   v1.29.0+k3s1
worker1   Ready    <none>                 8m17s   v1.28.6+k3s2
worker2   Ready    <none>                 7m43s   v1.28.6+k3s2
worker3   Ready    <none>                 7m9s    v1.28.6+k3s2
server    Ready    control-plane,master   10m     v1.29.0+k3s1
worker4   Ready    <none>                 6m36s   v1.29.0+k3s1
worker5   Ready    <none>                 6m4s    v1.29.0+k3s1
worker7   Ready    <none>                 4m37s   v1.29.0+k3s1
```

Worker nodes 1-3 make up the first group, which has Kubernetes `v1.28.6+k3s2`. The remaining worker nodes 4-7 make up a group which has `v1.29.0+k3s1`. Also both groups have different taints and labels as shown in the following table:

| Group | K8s version | taint | label |
|-|-|-|-|
| 1 (worker nodes 1-3) | v1.28.6+k3s2 | `redpanda-pool1=true:NoSchedule` | `nodetype=redpanda-pool1` |
| 2 (worker nodes 4-7) | v1.29.0+k3s1 | `redpanda-pool2=true:NoSchedule` | `nodetype=redpanda-pool2` |

When Redpanda was first deployed it contained a toleration and nodeSelector for the first group of nodes. The second group of nodes represent new nodes that were added to the cluster, possibly as new nodepool. The StatefulSet will need to be updated so that it has a new toleration and nodeSelector definition that matches this new set of nodes.

> Why do we have four additional Kubernetes nodes to replace the three original nodes?
>
> Because the recommended topic replica factor is 3. This means each partition within a topic must have three replicas or the cluster will go into unhealthy status. We will avoid making the cluster unhealthy by adding a temporary broker prior to decommissioning the three original brokers. Adding this temporary broker is unneeded if you have more brokers than your highest topic replica factor.

## Update operator

Modify the Redpanda resource with new tolerations, nodeSelector, and updateStrategy. One way to do this is by first creating a patch file:

```bash,run
cat <<EOF > patch.yaml
spec:
  clusterSpec:
    tolerations:
    - effect: NoSchedule
      key: redpanda-pool2
      operator: Equal
      value: "true"
    nodeSelector:
      nodetype: redpanda-pool2
    statefulset:
      replicas: 4
      updateStrategy:
        type: OnDelete
EOF
```

Then applying the patch:

```bash,run
kubectl patch redpanda redpanda -n redpanda --type merge --patch-file patch.yaml
```

Output:

```bash,nocopy
redpanda.cluster.redpanda.com/redpanda patched
```

At this point a new broker is being spun up on one of the replacement worker nodes with the updated Kubernetes version:

```bash,run
kubectl get pods -n redpanda -o wide
```

Once this broker has started there will be 4 brokers in the cluster:

```bash,run
rpk cluster info
```

Output:

```bash,nocopy
CLUSTER
-------
redpanda.893fd3b3-7eaa-44b1-a725-64464f106cd5

BROKERS
-------
ID    HOST                         PORT
0*    redpanda-0.testdomain.local  31092
1     redpanda-1.testdomain.local  31092
2     redpanda-2.testdomain.local  31092
3     redpanda-3.testdomain.local  31092
```

Node 3 is the temporary broker mentioned earlier, and will be removed from the cluster once the other three brokers are recreated. Check the cluster health:

```bash,run
rpk cluster health
```

Output:

```bash,nocopy
CLUSTER HEALTH OVERVIEW
-----------------------
Healthy:                          true
Unhealthy reasons:                []
Controller ID:                    0
All nodes:                        [0 1 2 3]
Nodes down:                       []
Leaderless partitions (0):        []
Under-replicated partitions (0):  []
```

> Note: You may need to run `rpk cluster health` a few times to see output similar to above.

## Update rpk profile

The temporary broker needs to be added to the rpk profile in order to allow rpk to communicate with this broker. First create a file containing an updated version of the existing profile:

```bash,run
rpk profile print | yq '.kafka_api.brokers += [ "redpanda-3.testdomain.local:31092" ],.admin_api.addresses += [ "redpanda-3.testdomain.local:31644" ]' > tmp-operator
```

Then use this file to create a new temporary profile (for use only while this temporary broker exists):

```bash,run
rpk profile create tmp-operator --from-profile tmp-operator
```

And since we are using `/etc/hosts` to provide DNS in this environment, we must update this file for the new broker:

```bash,run
echo `kubectl get svc/lb-redpanda-3 -n redpanda -o yaml | yq '.status.loadBalancer.ingress[].ip'` redpanda-3.testdomain.local | sudo tee -a /etc/hosts
```

## Replace original brokers

Replacing the original brokers consists of these steps taken on each broker:

1. decommission a broker
2. delete the PersistentVolumeClaim
3. delete the Pod

The nodes to be replaced are 0, 1, and 2.

> Note: In the output shown above, node 0 is the controller leader. We'll focus on decommissioning nodes 1 and 2 first so that the controller leader switches to another broker at most one time. This isn't required, but is common practice.

Decommission node 2:

```bash,run
rpk redpanda admin brokers decommission 2
```

Then check status:

```bash,run
rpk redpanda admin brokers decommission-status 2
```

Output will eventually say:

```bash,nocopy
Node 2 is decommissioned successfully.
```

> Note: The command above may return something similar to the following output:
>
> ```bash,nocopy
> DECOMMISSION PROGRESS
> =====================
> NAMESPACE-TOPIC  PARTITION  MOVING-TO  COMPLETION-%  PARTITION-SIZE
> kafka/log1       1          6          0             150093
> kafka/log3       2          6          0             151280
> kafka/log4       0          5          0             7937
> ```
>
> This would mean the decommission process is still progressing. Re-run the commission status command until you see the expected output above.

Now we can delete the PersistentVolumeClaim and then the pod associated with this broker:

```bash,run
kubectl delete pvc datadir-redpanda-2 -n redpanda --wait=false
```

Output:

```bash,nocopy
persistentvolumeclaim "datadir-redpanda-2" deleted
```

The `--wait=false` says to wait before deleting the associated PersistentVolume until the associated pod is deleted. This is done to ensure the Redpanda broker can shut down cleanly.

Now delete the pod `redpanda-2`:

```bash,run
kubectl delete pod redpanda-2 -n redpanda
```

The output will eventually say:

```bash,nocopy
pod "redpanda-2" deleted
```

> Note: Deleting a pod can take around 45 seconds depending on your environment.

Deleting the PVC and pod for redpanda-2 will allow the StatefulSet to create a new pod as redpanda-2 located on one of the replacement nodes. Check that this new pod is on one of the new Kubernetes nodes:

```bash,run
kubectl get pods -n redpanda -o wide | grep redpanda-2
```

The output should contain one of the replacement worker nodes (workers 4-7) in the 7th column:

```bash,nocopy
redpanda-2                                      2/2     Running     0          5m25s   10.42.6.6    worker6   <none>           <none>
```

Check that there are 4 brokers in the cluster:

```bash,run
rpk cluster info
```

Output:

```bash,nocopy
CLUSTER
-------
redpanda.11f95c5e-19cc-43fb-a846-b5dd39d20a9f

BROKERS
-------
ID    HOST                         PORT
0*    redpanda-0.testdomain.local  31092
1     redpanda-1.testdomain.local  31092
4     redpanda-2.testdomain.local  31092
3     redpanda-3.testdomain.local  31092
```

At this point we have updated the operator and StatefulSet to increase the broker count to 4 and ensure new brokers get assigned to the replacement Kubernetes nodes. Doing this has automatically brought up a temporary 4th broker (redpanda-3, node 3). We have also decommissioned the first of the original brokers (redpanda-2, node 2) and deleted the related cluster resources. The StatefulSet then created a new broker (redpanda-2, node 4) but with the same hostname as the previously decommissioned broker (redpanda-2, node 2). This is important since clients may only know of the original hostnames. If we only deleted the pods without first decommissioning the associated broker, then the StatefulSet would have created redpanda-4, node 4.

We will repeat the same steps for the remaining two original brokers. Continue with node 1:

```bash,run
rpk redpanda admin brokers decommission 1
```

```bash,run
rpk redpanda admin brokers decommission-status 1
```

```bash,run
kubectl delete pvc datadir-redpanda-1 -n redpanda --wait=false
```

```bash,run
kubectl delete pod redpanda-1 -n redpanda
```

Verify cluster health before continuing with the next broker:

```bash,run
rpk cluster health
```

Continue with node 0:

```bash,run
rpk redpanda admin brokers decommission 0
```

```bash,run
rpk redpanda admin brokers decommission-status 0
```

```bash,run
kubectl delete pvc datadir-redpanda-0 -n redpanda --wait=false
```

```bash,run
kubectl delete pod redpanda-0 -n redpanda
```

Now verify cluster health:

```bash,run
rpk cluster health
```

Output:

```bash,nocopy
CLUSTER HEALTH OVERVIEW
-----------------------
Healthy:                          true
Unhealthy reasons:                []
Controller ID:                    3
All nodes:                        [3 4 5 6]
Nodes down:                       []
Leaderless partitions (0):        []
Under-replicated partitions (0):  []
```

The cluster will have IDs 3-6 assigned to redpanda-0 through redpanda-3:

```bash,run
rpk cluster info
```

Output:

```bash,nocopy
CLUSTER
-------
redpanda.11f95c5e-19cc-43fb-a846-b5dd39d20a9f

BROKERS
-------
ID    HOST                         PORT
6     redpanda-0.testdomain.local  31092
5     redpanda-1.testdomain.local  31092
4     redpanda-2.testdomain.local  31092
3*    redpanda-3.testdomain.local  31092
```

Remove the temporary broker and update the Redpanda resource:

```bash,run
rpk redpanda admin brokers decommission 3
```

```bash,run
rpk redpanda admin brokers decommission-status 3
```

```bash,run
cat <<EOF > patch.yaml
spec:
  clusterSpec:
    statefulset:
      replicas: 3
      updateStrategy:
        type: RollingUpdate
EOF
```

```bash,run
kubectl patch redpanda redpanda -n redpanda --type merge --patch-file patch.yaml
```

Switch away from the temporary rpk profile, back to the operator profile:

```bash,run
rpk profile use operator
```

Verify cluster health:

```bash,run
rpk cluster health
```

The output will eventually be:

```bash,nocopy
CLUSTER HEALTH OVERVIEW
---------------------------------
Healthy:                          true
Unhealthy reasons:                []
Controller ID:                    4
All nodes:                        [4 5 6]
Nodes down:                       []
Leaderless partitions (0):        []
Under-replicated partitions (0):  []
```

The first group of nodes with the older Kubernetes version can now be removed from the cluster (if they were dedicated to Redpanda). Click 'Next' in the bottom right to continue to the next assignment.

