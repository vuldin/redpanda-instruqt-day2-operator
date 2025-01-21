---
slug: day2-operator-rolling-upgrade
id: 0bnmqvvbvj7y
type: challenge
title: Rolling upgrade
teaser: This section shows how to perform a rolling upgrade, and lists a few things
  to consider before kicking off this upgrade process.
notes:
- type: text
  contents: |-
    The next challenge shows how to quickly upgrade Redpanda with the operator.

    ![skate.png](../assets/skate.png)
tabs:
- id: 9l5gajduyqvj
  title: Shell
  type: terminal
  hostname: server
difficulty: ""
timelimit: 600
enhanced_loading: null
---
There are several components that make up an operator-based Redpanda deployment. Each component is listed below in the order they should be upgraded during this process:

1. CRDs
2. Operator and associated chart
3. Redpanda and associated chart

## Discover new versions

It is recommended to check the release notes for each version (found on the Github releases page linked below) to see if there are any significant changes that could impact your deployment. Also make sure to review the compatibility matrix to see which components have been tested with each other. More details can be found [here](https://docs.redpanda.com/current/upgrade/k-upgrade-operator/).

First let's determine which versions to use during the upgrade for each component. Run the following commands to get the latest versions:

Operator (from [Github releases page](https://github.com/redpanda-data/redpanda-operator/releases)):

```bash,run
curl -s https://api.github.com/repos/redpanda-data/redpanda-operator/releases/latest | jq -cr .tag_name
```

Operator chart (from [Github releases page](https://github.com/redpanda-data/helm-charts/releases)):

```bash,run
curl -s https://api.github.com/repos/redpanda-data/helm-charts/releases | jq -r '.[].name' | grep operator | head -1 | tr "-" "\n" | tail -1
```

Redpanda (from [Github releases page](https://github.com/redpanda-data/helm-charts/releases)):

```bash,run
curl -s https://api.github.com/repos/redpanda-data/redpanda/releases/latest | jq -cr .tag_name
```

Redpanda chart (from [Github releases page](https://github.com/redpanda-data/helm-charts/releases)):

```bash,run
curl -s https://api.github.com/repos/redpanda-data/helm-charts/releases | jq -r '.[].name' | grep redpanda | head -1 | tr "-" "\n" | tail -1
```

The target versions for this upgrade process are the following:

| Component | Version |
|-|-|
| Operator | v2.2.5-24.2.7 |
| Operator chart | 0.4.31 |
| Redpanda | 24.2.7 |
| Redpanda chart | 5.9.9 |

> Note: The version used above are likely different than the latest, but we are pinning to specific version to ensure future users don't run into any unforseen issues. In your environment you will likely want to target upgrading to the latest available version

Since we are currently on Redpanda 23.3.4, we will proceed with the upgrade order above, upgrading each component and finishing with upgrading Redpanda itself. But we will upgrade Redpanda two times: first to the latest 24.1 (24.1.17 at this time) and last to the latest 24.2 (24.2.7 at this time).

## Update CRDs

The Custom Resource Definitions (CRDs) define the resources that make up the operator deployment, and they are updated periodically. These CRDs should be updated prior to any update to the operator version to avoid issues.

```bash,run
kubectl kustomize "https://github.com/redpanda-data/redpanda-operator/operator/config/crd?ref=v2.2.5-24.2.7" | kubectl apply --server-side -f -
```

## Update operator

Now we have the updated CRDs we can upgrade the operator, making use of the proper operator helm chart (listed above):

```bash,run
helm upgrade --install redpanda-controller operator --repo https://charts.redpanda.com -n redpanda --set image.tag=v2.2.5-24.2.7 --version 0.4.31
```

Check the status of this process with the following command:

```bash,run
kubectl -n redpanda rollout status -w deployment/redpanda-controller-operator
```

## Update Redpanda chart and first update of Redpanda

Modify the Redpanda resource to the first Redpanda version as well as the updated chart version. First create a patch file:

```bash,run
cat <<EOF > patch.yaml
spec:
  chartRef:
    chartVersion: 5.9.9
  clusterSpec:
    image:
      tag: "v24.1.7"
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

A rolling restart has begun. The first broker is put into maintenance mode, moving any partition leadership to other brokers. Then the pod is redeployed with the updated Redpanda version. Then the broker is taken out of maintenance mode and the process continues to the next broker until all brokers have been upgraded. Verify the new version has eventually been applied to each broker:

```bash,run
rpk redpanda admin brokers list
```

Eventually the output will be similar the following:

```bash,nocopy
NODE-ID  NUM-CORES  MEMBERSHIP-STATUS  IS-ALIVE  BROKER-VERSION
4        1          active             true      v24.1.7 - 2c0d4bcfb15f814433713fc7be2586b37c4fda06
5        1          active             true      v24.1.7 - 2c0d4bcfb15f814433713fc7be2586b37c4fda06
6        1          active             true      v24.1.7 - 2c0d4bcfb15f814433713fc7be2586b37c4fda06
```

It may take time for the Redpanda resource to complete this rolling upgrade. You can verify the status by running this command:

```bash,run
kubectl get redpanda redpanda -n redpanda -o yaml  -o jsonpath={.status.conditions[0]} | jq
```

Eventually you will get this output similar to the following:

```bash,nocopy
{
  "lastTransitionTime": "2024-03-11T23:36:47Z",
  "message": "Redpanda reconciliation succeeded",
  "reason": "RedpandaClusterDeployed",
  "status": "True",
  "type": "Ready"
}
```

Look for `"status": "True"` and `"reason": "RedpandaClusterDeployed"` in the above output.

You can also verify that the chart upgrade has been applied with the following command:

```bash,run
kubectl describe redpanda -n redpanda | grep "Chart Version"
```

Expected output:

```bash,nocopy
    Chart Version:  5.9.9
```

## Last Redpanda update

Now we can update from Redpanda 24.1.17 to 24.2.7. Follow the same steps as above (only this time we don't need to include the Redpanda chart version since it is already updated):

```bash,run
cat <<EOF > patch.yaml
spec:
  clusterSpec:
    image:
      tag: "v24.2.7"
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

Verify the new version has eventually been applied to each broker:

```bash,run
rpk redpanda admin brokers list
```

Eventually the output will be similar the following:

```bash,nocopy
ID    HOST                                             PORT   RACK  CORES  MEMBERSHIP  IS-ALIVE  VERSION  UUID
0     redpanda-0.redpanda.redpanda.svc.cluster.local.  33145  -     1      active      true      24.2.7   46b73a70-94c0-4e08-873a-b2ff5b4cb69a
1     redpanda-1.redpanda.redpanda.svc.cluster.local.  33145  -     1      active      true      24.2.7   a28c74f7-2395-4c4b-92fd-bd56a16aef9d
2     redpanda-2.redpanda.redpanda.svc.cluster.local.  33145  -     1      active      true      24.2.7   14ef9ec1-4766-40f0-b560-3b9d0cb19167
```

This challenge is now complete. Click 'Next' in the bottom right to continue to the next assignment.
