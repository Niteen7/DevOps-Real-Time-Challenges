# Challenge 3: EKS Cluster Upgrade from Kubernetes 1.29 to 1.30

## Overview

As part of **Challenge 3: Upgrades**, I created a detailed manual for upgrading an **Amazon EKS cluster from Kubernetes version 1.29 to 1.30**.

The purpose of this manual is to document the upgrade process in a structured and repeatable way. It can be used by me and other team members as a reference for future EKS upgrade activities.

The manual focuses on the activities that should be completed **before, during, and after the upgrade**, with separate procedures for the **control plane** and **worker nodes (data plane)**.

---

## 1. Pre-Upgrade Preparation

Before starting the upgrade, it is important to prepare the cluster and understand the changes introduced in the target Kubernetes version.

### 1.1 Take Backups

Before performing any upgrade activity, take the necessary backups of the cluster and its critical resources.

The backup process should cover all important resources and configuration required to restore or recover the cluster if an unexpected issue occurs during the upgrade.

The backup procedure should be documented clearly so that the same process can be followed consistently for future upgrades.

---

### 1.2 Review the Kubernetes and EKS Release Notes

**Always review the release notes before starting an upgrade.**

Each Kubernetes and EKS version introduces new features, changes existing functionality, and may deprecate or remove APIs, features, or behaviors that were available in previous versions.

Before upgrading, we should understand:

* What new features are introduced in the target version.
* What existing features or functionalities have been changed.
* What APIs or features have been deprecated.
* What APIs or features have been removed.
* Whether any of the applications or Kubernetes resources currently deployed in the cluster depend on deprecated functionality.
* Whether any configuration changes are required before the upgrade.
* Whether there are any known compatibility issues.
* Whether there are any changes that could affect cluster performance or application behavior.

For example, a feature that is currently being used in Kubernetes 1.29 may be deprecated or removed in Kubernetes 1.30. In such cases, the required changes should be implemented **before starting the cluster upgrade**.

### Why Release Notes Are Important

Upgrading the cluster without understanding the changes between versions can result in compatibility issues, unexpected behavior, or application/service disruptions.

Therefore, the upgrade should **not be performed in a hurry**.

The release notes should be reviewed carefully, and all relevant changes should be identified and documented before proceeding with the upgrade.

> **Important:** Do not start the upgrade until all relevant release-note changes, deprecated APIs, removed features, and compatibility requirements have been reviewed.

---

## 2. Separate the Upgrade into Control Plane and Data Plane

The upgrade procedure should be divided into two major sections:

1. **Control Plane Upgrade**
2. **Worker Node / Data Plane Upgrade**

Keeping these procedures separate makes the upgrade process easier to understand, execute, troubleshoot, and reuse for future activities.

---

# 3. Control Plane Upgrade

The control plane contains the core Kubernetes components responsible for managing the cluster.

The upgrade documentation should clearly define the required steps and the correct order in which they should be performed.

The control plane section should include the following components and activities:

### 3.1 etcd

Document the required steps related to **etcd**, including:

* How to verify the current etcd state.
* How to take an etcd backup/snapshot where applicable.
* How to validate the backup.
* What checks need to be performed before the upgrade.
* What validation needs to be performed after the upgrade.

### 3.2 kube-apiserver

Document the procedure for upgrading the **Kubernetes API server** from the existing version to the target version.

The procedure should include:

* Pre-upgrade checks.
* Current version verification.
* Upgrade procedure.
* Post-upgrade verification.
* API availability checks.
* Validation of cluster communication.

### 3.3 kube-scheduler

Document the required steps for upgrading and validating the **kube-scheduler**.

The procedure should include:

* Current version verification.
* Upgrade procedure.
* Health/status verification.
* Post-upgrade validation.
* Confirmation that workloads can continue to be scheduled correctly.

### 3.4 Control Plane Upgrade Order

The manual should clearly document the **correct upgrade sequence and dependencies** between the control-plane components.

For every step, document:

* What needs to be checked before starting.
* What command or action needs to be performed.
* What output/result is expected.
* How to verify that the step completed successfully.
* What to do if the step fails.
* What validation needs to be completed before moving to the next step.

This will make the manual easier for other team members to follow without missing important steps.

---

# 4. Worker Node / Data Plane Upgrade

After the control plane has been successfully upgraded and validated, the worker nodes can be upgraded.

The worker-node upgrade should be performed **one node at a time** to minimize the impact on running workloads.

For example, assume the cluster has **5 worker nodes**:

```text
Worker Node 1 → Kubernetes 1.29
Worker Node 2 → Kubernetes 1.29
Worker Node 3 → Kubernetes 1.29
Worker Node 4 → Kubernetes 1.29
Worker Node 5 → Kubernetes 1.29
```

The nodes should be upgraded sequentially rather than upgrading all nodes simultaneously.

---

## 4.1 Identify the Worker Nodes

First, determine:

* How many worker nodes are present.
* Which Kubernetes version each node is running.
* Whether each node is healthy.
* Whether the cluster has sufficient capacity to move workloads from one node to another.

For example:

```text
Total Worker Nodes: 5

Node 1 → 1.29
Node 2 → 1.29
Node 3 → 1.29
Node 4 → 1.29
Node 5 → 1.29
```

Before draining any node, ensure that the remaining nodes have sufficient resources to run the workloads being evicted.

---

## 4.2 Cordon the Node

Select one worker node for the upgrade.

Before draining it, **cordon the node** so that new pods are not scheduled onto it.

A cordoned node becomes **unschedulable**, which prevents the Kubernetes scheduler from placing new pods on that node.

Example:

```bash
kubectl cordon <node-name>
```

Verify the node status:

```bash
kubectl get nodes
```

The selected node should show an unschedulable state.

> **Note:** Cordon and taint are different mechanisms. Cordon marks a node as unschedulable, while taints prevent pods from being scheduled unless they have matching tolerations. Use the mechanism required by the upgrade procedure rather than treating them as interchangeable.

---

## 4.3 Drain the Node

After cordoning the node, drain it to safely evict workloads and move eligible pods to other healthy worker nodes.

Example:

```bash
kubectl drain <node-name> --ignore-daemonsets
```

The exact drain options should be selected according to the workloads and policies running in the cluster.

The goal is to ensure that the node is no longer running workloads that need to be moved before the node upgrade.

After draining, verify the node and workloads:

```bash
kubectl get nodes
kubectl get pods -A -o wide
```

The workloads that can be evicted should have been rescheduled onto other healthy nodes.

---

## 4.4 Upgrade the Worker Node

Once the node has been successfully drained:

1. Verify that the node is ready for maintenance.
2. Perform the required node-level upgrade procedure.
3. Upgrade the **kubelet** to the target Kubernetes version.
4. Upgrade the required Kubernetes packages and dependencies.
5. Apply any required configuration changes.
6. Restart required services where applicable.
7. Verify that the node rejoins the Kubernetes cluster successfully.

The exact package and node upgrade commands should be documented according to the worker-node operating system and the EKS node type being used.

---

## 4.5 Verify the Upgraded Node

After upgrading the node, verify that it has successfully joined the cluster and is healthy.

Check the node:

```bash
kubectl get nodes
```

Verify the Kubernetes version:

```bash
kubectl get nodes -o wide
```

Confirm that:

* The node is in `Ready` state.
* The node is running the expected Kubernetes version.
* kubelet is healthy.
* Required system pods are running.
* The node has no unexpected errors or conditions.
* The node is communicating correctly with the control plane.

---

## 4.6 Make the Node Schedulable Again

Once all upgrade and validation activities are complete, make the node schedulable again.

Example:

```bash
kubectl uncordon <node-name>
```

Verify:

```bash
kubectl get nodes
```

The node should now be available for scheduling workloads.

If a maintenance taint was explicitly added during the process, remove it only after confirming that the node is fully ready.

---

# 5. Repeat the Process for the Remaining Worker Nodes

After successfully upgrading and validating the first worker node, repeat the same procedure for the remaining nodes.

For example:

### Initial State

```text
Node 1 → 1.29
Node 2 → 1.29
Node 3 → 1.29
Node 4 → 1.29
Node 5 → 1.29
```

### After Upgrading Node 5

```text
Node 1 → 1.29
Node 2 → 1.29
Node 3 → 1.29
Node 4 → 1.29
Node 5 → 1.30
```

Then continue with the next node:

```text
Node 1 → 1.29
Node 2 → 1.29
Node 3 → 1.29
Node 4 → 1.30
Node 5 → 1.30
```

Continue this process until all worker nodes have been upgraded and validated.

### Final State

```text
Node 1 → 1.30
Node 2 → 1.30
Node 3 → 1.30
Node 4 → 1.30
Node 5 → 1.30
```

---

# 6. Post-Upgrade Validation

After completing the control-plane and worker-node upgrades, perform a complete cluster validation.

Verify:

* Kubernetes control-plane version.
* Worker-node versions.
* All nodes are in `Ready` state.
* All required system pods are running.
* Application workloads are healthy.
* Pods are distributed correctly across worker nodes.
* Services are accessible.
* Ingress/load-balancer functionality is working.
* No unexpected errors are present in cluster events.
* Monitoring and logging systems are functioning correctly.
* Application performance is normal.
* There are no unexpected warnings related to deprecated or removed APIs.

Example checks:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A
kubectl get events -A
```

---

# 7. Key Upgrade Principles

The following principles should be followed for every EKS upgrade:

1. **Always take backups before starting the upgrade.**
2. **Always review the release notes before upgrading.**
3. **Identify deprecated and removed APIs/features before the upgrade.**
4. **Validate application and workload compatibility with the target Kubernetes version.**
5. **Upgrade the control plane before upgrading the worker nodes.**
6. **Upgrade worker nodes one at a time whenever the architecture and capacity allow it.**
7. **Cordon and drain a worker node before performing maintenance.**
8. **Verify the upgraded node before making it schedulable again.**
9. **Monitor the cluster throughout the upgrade process.**
10. **Perform complete post-upgrade validation after all nodes have been upgraded.**
11. **Document any issues encountered and the corresponding resolution.**
12. **Do not rush the upgrade. Proper preparation and validation are more important than completing the upgrade quickly.**

---

# 8. Conclusion

The purpose of this manual is to provide a structured and reusable procedure for upgrading an **Amazon EKS cluster from Kubernetes 1.29 to 1.30**.

The most important part of an upgrade is not simply changing the Kubernetes version. It is understanding the changes introduced by the target version, identifying compatibility issues in advance, protecting the existing environment through backups, and upgrading the cluster in a controlled and validated manner.

By maintaining separate procedures for the **control plane** and **worker nodes/data plane**, and by documenting the required checks before and after each stage, this manual can serve as a reliable reference for future Kubernetes and EKS upgrade activities.
