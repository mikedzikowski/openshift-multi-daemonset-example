# CrowdStrike Falcon Node Sensor — OpenShift Deployment Guide

## Overview

This manifest deploys the CrowdStrike Falcon Node Sensor across all nodes in an
OpenShift cluster using the Falcon Operator. It creates two separate DaemonSets
to ensure complete node coverage:

| DaemonSet | Namespace | Target Nodes |
|---|---|---|
| `falcon-node-sensor-worker` | `falcon-system` | Worker nodes |
| `falcon-node-sensor-master` | `falcon-master-nodes` | Master / Control Plane nodes |

### What Gets Deployed

| Resource | Description |
|---|---|
| 3 Namespaces | `falcon-operator`, `falcon-system`, `falcon-master-nodes` |
| 2 Service Accounts | One per sensor namespace |
| 1 SCC | Privileged access required for kernel-level sensor |
| 1 ClusterRole + 2 ClusterRoleBindings | RBAC for operator |
| 2 FalconNodeSensor CRs | One per node type |
| 1 OperatorGroup + 1 Subscription | OLM operator installation |

---

## Prerequisites

### Access Requirements
- OpenShift cluster-admin privileges
- `oc` CLI installed and logged in
- Access to OperatorHub / Red Hat Certified Operators catalog

### CrowdStrike Requirements
- Falcon Customer ID (CID)
- Falcon sensor image hosted in an accessible container registry
- Proxy host and port (if your cluster requires a proxy to reach CrowdStrike cloud)

### Verify Cluster Access
```bash
oc whoami
oc cluster-info
oc auth can-i '*' '*' --all-namespaces
```

---

## Configuration

Before deploying, substitute the following placeholders in
`falcon-node-sensor-cid.yaml`:

| Placeholder | Description | Example |
|---|---|---|
| `<YOUR_FALCON_CID>` | CrowdStrike Falcon Customer ID | `1234567890ABCDEF` |
| `<YOUR_PROXY_HOST>` | Proxy hostname | `proxy.example.com` |
| `<YOUR_PROXY_PORT>` | Proxy port | `8080` |
| `<YOUR_REGISTRY>/falcon-sensor:<VERSION>` | Full sensor image path | `myregistry.example.com/falcon-sensor:7.34.0` |
| `<YOUR_CLUSTER_NAME>` | Cluster name for sensor grouping tag | `my-ocp-cluster` |

### Substitute Placeholders
```bash
sed -i 's/<YOUR_FALCON_CID>/your_cid/g' falcon-node-sensor-cid.yaml
sed -i 's/<YOUR_PROXY_HOST>/proxy.example.com/g' falcon-node-sensor-cid.yaml
sed -i 's/<YOUR_PROXY_PORT>/8080/g' falcon-node-sensor-cid.yaml
sed -i 's|<YOUR_REGISTRY>/falcon-sensor:<VERSION>|myregistry.example.com/falcon-sensor:7.34.0|g' falcon-node-sensor-cid.yaml
sed -i 's/<YOUR_CLUSTER_NAME>/my-ocp-cluster/g' falcon-node-sensor-cid.yaml
```

### Verify No Placeholders Remain
```bash
grep -n "YOUR_" falcon-node-sensor-cid.yaml
```
> Output should be empty before proceeding.

> If your cluster does not require a proxy, remove the `apd`, `aph`, and `app`
> fields from both FalconNodeSensor CRs. If OpenShift cluster-wide proxy is
> configured via OLM, the operator will pick it up automatically.

---

## Deployment Steps

### Step 1 — Apply the Manifest
```bash
oc apply -f falcon-node-sensor-cid.yaml
```

### Step 2 — Approve the InstallPlan
The operator subscription uses `Manual` approval to prevent unintended upgrades.
Approve the InstallPlan to allow the operator to install:
```bash
INSTALL_PLAN=$(oc get installplan -n falcon-operator \
  -o jsonpath='{.items[0].metadata.name}')

oc patch installplan $INSTALL_PLAN \
  -n falcon-operator \
  --type merge \
  --patch '{"spec":{"approved":true}}'
```

### Step 3 — Apply SCC Bindings
```bash
oc adm policy add-scc-to-user falcon-sensor-scc \
  system:serviceaccount:falcon-system:falcon-operator-node-sensor

oc adm policy add-scc-to-user falcon-sensor-scc \
  system:serviceaccount:falcon-master-nodes:falcon-operator-node-sensor
```

### Step 4 — Wait for Operator to be Running
```bash
watch oc get pods -n falcon-operator
```
> Proceed only when the operator pod shows `Running`.

### Step 5 — Patch Master DaemonSet NodeSelector
The Falcon Operator overrides the `nodeSelector` field and sets it to
`kubernetes.io/os: linux` which matches all nodes. The patch below restricts
the master DaemonSet to master nodes only. Run this after the master DaemonSet
is created:

```bash
oc patch daemonset falcon-node-sensor-master \
  -n falcon-master-nodes \
  --type=merge \
  -p '{"spec":{"template":{"spec":{"nodeSelector":
  {"kubernetes.io/os":"linux","node-role.kubernetes.io/master":""}}}}}'
```

### Step 6 — Verify Deployment
```bash
# Check both DaemonSets
oc get daemonset -n falcon-system
oc get daemonset -n falcon-master-nodes

# Check pods on correct nodes
oc get pods -n falcon-system -o wide && echo "---" && \
oc get pods -n falcon-master-nodes -o wide

# Confirm no node has duplicate sensor pods
oc get pods -A -o wide | grep falcon-sensor

# Check FalconNodeSensor CR status
oc get falconnodesensor -A
```

### Expected Output
```
# falcon-system (worker nodes)
NAME                        DESIRED   CURRENT   READY   AVAILABLE
falcon-node-sensor-worker   3         3         3       3

# falcon-master-nodes (master nodes)
NAME                        DESIRED   CURRENT   READY   AVAILABLE
falcon-node-sensor-master   3         3         3       3
```
```
Expected Output
DaemonSets
NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
falcon-node-sensor-worker   3         3         3       3            3           kubernetes.io/os=linux   118m
---
NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                                          AGE
falcon-node-sensor-master   3         3         3       3            3           kubernetes.io/os=linux,node-role.kubernetes.io/master=  7m39s
Pods
NAME                              READY   STATUS    RESTARTS   AGE    IP           NODE
falcon-node-sensor-worker-6xzf5   1/1     Running   0          115m   10.2.4.132   worker-eastus1-f6kbc
falcon-node-sensor-worker-s4qlz   1/1     Running   0          115m   10.2.4.134   worker-eastus2-867fq
falcon-node-sensor-worker-vspkd   1/1     Running   0          115m   10.2.4.135   worker-eastus3-rf7qq
---
NAME                              READY   STATUS    RESTARTS   AGE    IP          NODE
falcon-node-sensor-master-n27gn   1/1     Running   0          95s    10.2.4.10   master-2
falcon-node-sensor-master-qpcms   1/1     Running   0          2m35s  10.2.4.8    master-0
falcon-node-sensor-master-rftgh   1/1     Running   0          118s   10.2.4.9    master-1
---
```

## Troubleshooting

### Operator Not Starting
```bash
oc describe subscription falcon-operator -n falcon-operator
oc get events -n falcon-operator --sort-by='.lastTimestamp'
```

### Pods Not Scheduling on Correct Nodes
```bash
oc get events -n falcon-system --sort-by='.lastTimestamp'
oc get events -n falcon-master-nodes --sort-by='.lastTimestamp'
```

### Pods CrashLoopBackOff
```bash
oc logs <pod-name> -n falcon-master-nodes
oc logs <pod-name> -n falcon-master-nodes -c init-falconstore
oc describe pod <pod-name> -n falcon-master-nodes | tail -30
```

### SCC Issues
```bash
oc adm policy who-can use scc falcon-sensor-scc
```

### Operator Logs
```bash
oc logs -n falcon-operator \
  $(oc get pods -n falcon-operator -o jsonpath='{.items[0].metadata.name}') \
  --tail=100
```

### Master DaemonSet Reverts NodeSelector
The operator may revert the nodeSelector patch during reconciliation.
If master pods reappear on worker nodes, re-run the patch from Step 5.

---

## Uninstall

```bash
# Remove FalconNodeSensor CRs (operator will clean up DaemonSets)
oc delete falconnodesensors --all

# Remove remaining resources
oc delete -f falcon-node-sensor-cid.yaml

# Remove SCC bindings
oc adm policy remove-scc-from-user falcon-sensor-scc \
  system:serviceaccount:falcon-system:falcon-operator-node-sensor
oc adm policy remove-scc-from-user falcon-sensor-scc \
  system:serviceaccount:falcon-master-nodes:falcon-operator-node-sensor
```

> If uninstallation pods crashloop, manually remove `/opt/CrowdStrike` from
> affected nodes to prevent stale Agent IDs on reinstall.

---

## Architecture

```
OpenShift Cluster
├── falcon-operator (namespace)
│   └── falcon-operator pod          ← manages both CRs
├── falcon-system (namespace)
│   ├── FalconNodeSensor CR          ← worker CR
│   └── DaemonSet                    ← 1 pod per worker node
└── falcon-master-nodes (namespace)
    ├── FalconNodeSensor CR          ← master CR
    └── DaemonSet                    ← 1 pod per master node
```

---

## Resources

- [Falcon Operator GitHub](https://github.com/crowdstrike/falcon-operator)
- [Falcon Operator Documentation](https://github.com/crowdstrike/falcon-operator/blob/main/docs/node/README.md)
- [CrowdStrike Support Portal](https://supportportal.crowdstrike.com)
