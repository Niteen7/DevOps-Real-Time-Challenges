# Kubernetes Real-Time DevOps Challenges

This repository contains practical, real-world Kubernetes troubleshooting and operational scenarios commonly encountered by DevOps/SRE teams.

The purpose of this repository is to document real-time challenges, troubleshooting approaches, upgrade procedures, operational best practices, and lessons learned so that they can be used as a reference for future activities.

---

## Topics Covered

### 1. Resource Sharing & Isolation

Managing multiple teams and applications within Kubernetes requires proper resource isolation and capacity management.

Topics covered:

* Managing multiple teams/applications using Kubernetes namespaces.
* Using `ResourceQuota` to control aggregate CPU and memory usage per namespace.
* Using `LimitRange` to define default, minimum, and maximum resource requests and limits.
* Understanding the difference between:

  * Resource Requests
  * Resource Limits
  * ResourceQuota
  * LimitRange
  * Node Capacity
* Understanding how resource requests affect Pod scheduling.
* Understanding how memory limits can result in `OOMKilled`.
* Reducing the blast radius through namespace-level and container-level resource isolation.

---

### 2. OOMKilled & CrashLoopBackOff

This challenge focuses on troubleshooting Kubernetes workloads that experience `OOMKilled` and `CrashLoopBackOff` conditions.

A typical production troubleshooting flow:

```text
Pod
 |
 v
CrashLoopBackOff
 |
 v
kubectl describe pod
 |
 v
Check Events / Last State
 |
 v
OOMKilled?
 |
 v
Check Logs + Resource Usage
 |
 v
Investigate Application/JVM
 |
 v
Root Cause Analysis
 |
 v
Fix + Deploy
 |
 v
Monitor
```

#### Useful Commands

```bash
kubectl get pods
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl top pod <pod>
kubectl top nodes
kubectl get resourcequota
kubectl describe resourcequota
kubectl get limitrange
kubectl describe limitrange
```

#### Important Considerations

* `OOMKilled` indicates that a container was terminated because it exceeded its memory limit or the node experienced memory pressure; it does not automatically mean there is a memory leak.
* `CrashLoopBackOff` is a restart/backoff condition and is not necessarily the root cause.
* Application logs and Kubernetes events should be reviewed before making changes.
* Increasing a memory limit is not always the correct solution.
* Resource usage should be analyzed before increasing CPU or memory allocations.
* Application-level issues should be investigated before simply restarting or scaling the workload.

---

### 3. Java Application Troubleshooting

Java applications require additional troubleshooting techniques because JVM-level behavior can directly affect Kubernetes workloads.

#### Thread Dump

A **Thread Dump** can be used to investigate:

* Blocked threads
* Deadlocks
* Thread contention
* High CPU usage
* Thread pool issues

#### Heap Dump

A **Heap Dump** can be used to investigate:

* JVM memory usage
* Object retention
* Potential memory leaks
* Unexpected memory growth

#### JVM and GC Metrics

JVM and Garbage Collection metrics can help identify:

* Increasing heap usage
* Frequent Garbage Collection
* Long GC pauses
* Memory-growth patterns
* Potential memory leaks

Container memory limits must also be considered when configuring JVM memory settings.

#### Common Diagnostic Commands

```bash
jstack <PID>
kill -3 <PID>
jmap -dump:format=b,file=heap.hprof <PID>
jcmd <PID> GC.heap_dump heap.hprof
```

---

### 4. EKS Cluster Upgrade — Kubernetes 1.29 to 1.30

This challenge focuses on upgrading an **Amazon EKS cluster from Kubernetes version 1.29 to 1.30**.

The upgrade procedure is documented as a reusable operational manual that can be followed by DevOps/SRE team members during future Kubernetes upgrades.

The objective is not simply to change the Kubernetes version, but to perform the upgrade in a **controlled, safe, and validated manner**.

The upgrade process is divided into:

1. Pre-upgrade preparation
2. Release-note and compatibility review
3. Control-plane upgrade
4. Worker-node/data-plane upgrade
5. Post-upgrade validation

---

## 4.1 Pre-Upgrade Preparation

Before starting the upgrade, the existing cluster must be reviewed and prepared.

### Backup

Take the necessary backups before performing any upgrade activity.

Backups should cover critical cluster resources and configuration required for recovery in case an unexpected issue occurs during the upgrade.

The backup procedure should be documented and validated so that the same process can be followed consistently during future upgrades.

---

## 4.2 Review Release Notes

**Release notes must be reviewed before starting the upgrade.**

Every Kubernetes/EKS release can introduce:

* New features
* Feature changes
* Deprecated APIs
* Removed APIs
* Configuration changes
* Behavioral changes
* Compatibility requirements
* Performance-related changes
* Changes that may affect existing applications or workloads

Before upgrading from Kubernetes 1.29 to 1.30, verify whether any functionality currently being used in the cluster has been deprecated or removed in the target version.

### Why Release Notes Matter

A Kubernetes upgrade should never be performed without understanding the changes introduced by the target version.

For example, an API or feature that works in Kubernetes 1.29 may be deprecated or removed in Kubernetes 1.30.

If such dependencies are not identified before the upgrade, applications or cluster components may experience compatibility issues.

Therefore:

> **Do not rush the upgrade. Read and understand the release notes before making any changes.**

The following should be documented before proceeding:

* Deprecated APIs/features
* Removed APIs/features
* Required configuration changes
* Application compatibility requirements
* Kubernetes component changes
* Known issues
* Required pre-upgrade actions
* Required post-upgrade validation

---

## 4.3 Control Plane Upgrade

The control plane upgrade should be documented separately from the worker-node upgrade.

The upgrade procedure should clearly define the correct order, dependencies, validation steps, and rollback/recovery considerations.

### Control Plane Components

The documentation should cover:

1. `etcd`
2. `kube-apiserver`
3. `kube-scheduler`
4. Other control-plane components applicable to the EKS environment

### etcd

The procedure should include:

* Checking the current etcd state.
* Taking an etcd backup/snapshot where applicable.
* Validating the backup.
* Performing required pre-upgrade checks.
* Performing post-upgrade validation.

> **Note:** In Amazon EKS, AWS manages the EKS control plane and its underlying control-plane components. Therefore, the exact upgrade procedure differs from a self-managed Kubernetes cluster. Control-plane components such as `etcd`, `kube-apiserver`, and `kube-scheduler` should not be manually upgraded inside an AWS-managed EKS control plane.

### kube-apiserver

For an EKS cluster, the Kubernetes control plane version is upgraded through the EKS upgrade mechanism.

Validation should include:

* Current Kubernetes version.
* Target Kubernetes version.
* API server availability.
* Cluster connectivity.
* API compatibility.
* Post-upgrade cluster health.

### kube-scheduler

The scheduler is part of the EKS-managed control plane.

After the control-plane upgrade, validate that:

* The control plane is healthy.
* Pods can be scheduled successfully.
* New workloads can be deployed.
* Existing workloads continue operating normally.

---

## 4.4 Worker Node / Data Plane Upgrade

After the EKS control plane has been successfully upgraded and validated, worker nodes can be upgraded.

Worker nodes should generally be upgraded **one at a time** where the cluster capacity and architecture allow it.

For example, assume the cluster has five worker nodes:

```text
Node 1 → Kubernetes 1.29
Node 2 → Kubernetes 1.29
Node 3 → Kubernetes 1.29
Node 4 → Kubernetes 1.29
Node 5 → Kubernetes 1.29
```

The nodes should be upgraded sequentially rather than taking all worker nodes offline at the same time.

---

## 4.5 Identify Worker Nodes

Before upgrading a worker node, determine:

* Total number of worker nodes.
* Kubernetes version of each node.
* Node health/status.
* Available CPU and memory capacity.
* Workloads currently running on the node.
* Whether the remaining nodes have enough capacity to run rescheduled workloads.

Example:

```bash
kubectl get nodes
```

Example cluster state:

```text
NAME      STATUS   VERSION
node-01   Ready    v1.29
node-02   Ready    v1.29
node-03   Ready    v1.29
node-04   Ready    v1.29
node-05   Ready    v1.29
```

---

## 4.6 Cordon the Worker Node

Before draining the node, mark it as **unschedulable**.

This prevents new Pods from being scheduled onto the node.

Example:

```bash
kubectl cordon <node-name>
```

Verify the node:

```bash
kubectl get nodes
```

> **Cordon and taint are different mechanisms.** Cordon marks a node as unschedulable, while a taint prevents Pods from being scheduled unless they have a matching toleration. Use the mechanism required by the specific maintenance procedure.

---

## 4.7 Drain the Worker Node

After cordoning the node, drain it to safely evict workloads that can be moved to other healthy nodes.

Example:

```bash
kubectl drain <node-name> --ignore-daemonsets
```

The exact drain options should be selected according to the workloads, PodDisruptionBudgets, DaemonSets, and other policies configured in the cluster.

Verify the workloads after draining:

```bash
kubectl get pods -A -o wide
```

The goal is to ensure that workloads that can be evicted have been rescheduled onto other healthy worker nodes.

---

## 4.8 Upgrade the Worker Node

Once the node has been successfully drained:

1. Verify that the node is ready for maintenance.
2. Perform the appropriate worker-node upgrade procedure.
3. Upgrade `kubelet` to the target Kubernetes version.
4. Upgrade required Kubernetes packages and dependencies.
5. Apply required configuration changes.
6. Restart required services where applicable.
7. Verify that the node successfully rejoins the cluster.

The exact commands depend on the EKS node architecture being used, such as:

* Managed Node Groups
* Self-managed nodes
* EKS Auto Mode
* Bottlerocket
* Amazon Linux
* Other supported node configurations

The node-specific upgrade procedure should therefore be documented separately for the environment.

---

## 4.9 Validate the Upgraded Worker Node

After upgrading the node, verify that it has successfully joined the cluster.

```bash
kubectl get nodes
```

Check detailed information:

```bash
kubectl get nodes -o wide
```

Confirm that:

* The node is in `Ready` state.
* The node is running the expected Kubernetes version.
* `kubelet` is healthy.
* Required system Pods are running.
* There are no unexpected node conditions.
* The node can communicate with the control plane.
* Workloads can be scheduled successfully.

---

## 4.10 Make the Node Schedulable

After completing the upgrade and validation, make the node schedulable again:

```bash
kubectl uncordon <node-name>
```

Verify:

```bash
kubectl get nodes
```

If a maintenance taint was explicitly added, remove it only after confirming that the node is fully healthy and ready to accept workloads.

---

## 4.11 Repeat for Remaining Worker Nodes

After successfully upgrading and validating the first worker node, repeat the same procedure for the remaining nodes.

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

Continue the same process for the remaining nodes until all worker nodes have been upgraded.

### Final State

```text
Node 1 → 1.30
Node 2 → 1.30
Node 3 → 1.30
Node 4 → 1.30
Node 5 → 1.30
```

---

## 4.12 Post-Upgrade Validation

After completing both the control-plane and worker-node upgrades, perform a complete cluster validation.

Verify:

* Control-plane version.
* Worker-node versions.
* All nodes are in `Ready` state.
* Required system Pods are running.
* Application workloads are healthy.
* Pods are distributed correctly.
* Services are accessible.
* Ingress/load-balancer functionality is working.
* Monitoring and logging are functioning correctly.
* No unexpected Kubernetes events are present.
* Applications are performing normally.
* No unexpected warnings related to deprecated or removed APIs are present.

### Useful Validation Commands

```bash
kubectl get nodes
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A
kubectl get events -A
```

---

# Key Takeaways

The following principles should be followed during Kubernetes and EKS upgrade activities:

* **Always take backups before starting an upgrade.**
* **Always review the release notes before upgrading.**
* Identify deprecated and removed APIs/features before the upgrade.
* Validate application compatibility with the target Kubernetes version.
* Upgrade the EKS control plane before upgrading compatible worker nodes.
* Upgrade worker nodes in a controlled and incremental manner.
* Cordon and drain worker nodes before performing node maintenance.
* Validate every upgraded node before returning it to service.
* Monitor cluster health throughout the upgrade.
* Perform complete post-upgrade validation.
* Document issues encountered during the upgrade and their resolutions.
* Perform Root Cause Analysis (RCA) for unexpected issues.
* Do not rush an upgrade. Proper preparation and validation are more important than completing the upgrade quickly.

---

# Production Troubleshooting Principles

The challenges documented in this repository follow a few important DevOps/SRE principles.

### 1. Understand Before Changing

Before making changes to a production system, understand:

* What is currently happening.
* What changed recently.
* What the expected behavior should be.
* What the potential impact of the change is.
* How the change can be validated.
* How the change can be reversed if necessary.

### 2. Check Evidence Before Taking Action

Use Kubernetes events, logs, metrics, resource usage, application diagnostics, and system information to identify the actual problem.

Avoid making changes based only on assumptions.

### 3. Perform Root Cause Analysis

Do not simply restart a Pod, increase resources, or add worker nodes without understanding why the problem occurred.

The objective should always be to identify the **root cause**, implement the appropriate fix, and monitor the system afterward.

### 4. Minimize Blast Radius

Changes should be performed in a controlled manner.

Examples include:

* Namespace-level resource isolation.
* Pod resource requests and limits.
* Resource quotas.
* Gradual worker-node upgrades.
* Cordon and drain before node maintenance.
* Validation after each major change.

---

# DevOps and Development Responsibilities

Troubleshooting production issues is often a collaborative effort between DevOps/SRE and development teams.

### DevOps/SRE Responsibilities

DevOps/SRE teams typically provide:

* Infrastructure.
* Kubernetes cluster management.
* Resource configuration.
* Monitoring and observability.
* Logs and metrics.
* Cluster-level troubleshooting.
* Deployment/platform support.
* Upgrade procedures.
* Operational documentation.

### Development Responsibilities

Development teams typically investigate and resolve:

* Application-level issues.
* Application bugs.
* Memory leaks.
* Threading issues.
* JVM configuration problems.
* Application performance problems.
* Code-level failures.

Effective troubleshooting requires collaboration between both teams.

---

# Repository Goal

The goal of this repository is to document **real-world Kubernetes production challenges, troubleshooting approaches, EKS operational procedures, upgrade processes, and DevOps best practices**.

The documentation is intended to serve as a practical reference for:

* DevOps Engineers
* SREs
* Kubernetes Engineers
* Cloud Engineers
* Developers working with Kubernetes
* Team members responsible for production operations

The objective is to continuously add real-time scenarios, document the investigation and resolution process, and build a reusable knowledge base for future Kubernetes and DevOps activities.
