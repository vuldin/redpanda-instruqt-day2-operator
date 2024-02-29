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
- title: Shell
  type: terminal
  hostname: server
difficulty: ""
timelimit: 600
---
Upgrading Redpanda with the operator is a simple process. We will update the Redpanda resource to reference the latest version, and then the operator handles the rest.

## Update operator

Modify the Redpanda resource to use a new Redpanda version. First create a patch file:

```bash,run
cat <<EOF > patch.yaml
spec:
  clusterSpec:
    image:
      tag: "v23.3.5"
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

A rolling restart has begun. Each broker is put into maintenance mode, moving any partition leadership to other brokers. Then Redpanda is upgraded to the chosen version and the process is restarted. Then the broker is taken out of maintenance mode and the process continues to the next broker until all brokers have been upgraded. Verify the new version has eventually been applied to each broker:

```bash,run
rpk redpanda admin brokers list
```

Eventually the output will be the following:

```bash,nocopy
NODE-ID  NUM-CORES  MEMBERSHIP-STATUS  IS-ALIVE  BROKER-VERSION
4        1          active             true      v23.3.5 - 2c0d4bcfb15f814433713fc7be2586b37c4fda06
5        1          active             true      v23.3.5 - 2c0d4bcfb15f814433713fc7be2586b37c4fda06
6        1          active             true      v23.3.5 - 2c0d4bcfb15f814433713fc7be2586b37c4fda06
```

It may take time for the Redpanda resource to complete this rolling upgrade. You can verify the status by running this command:

```bash,run
kubectl get redpanda redpanda -n redpanda -o yaml  -o jsonpath={.status} | jq
```

Eventually you will get this output similar to the following:

```bash,nocopy
{
  "conditions": [
    {
      "lastTransitionTime": "2024-03-11T23:36:47Z",
      "message": "Redpanda reconciliation succeeded",
      "reason": "RedpandaClusterDeployed",
      "status": "True",
      "type": "Ready"
    }
  ],
  "helmRelease": "redpanda",
  "helmReleaseReady": true,
  "helmRepository": "redpanda-repository",
  "helmRepositoryReady": true,
  "observedGeneration": 4
}
```

Look for `"status": "True"` and `"reason": "RedpandaClusterDeployed"` in the above output.

This challenge is now complete. Click 'Next' in the bottom right to continue to the next assignment.
