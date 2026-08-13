# Kubernetes Real-Time DevOps Challenges

> Real-world Kubernetes scenarios covering resource sharing, `ResourceQuota`, `LimitRange`, CPU/memory requests and limits, `OOMKilled`, `CrashLoopBackOff`, and Java application troubleshooting.

---

# Challenge 1: Resource Sharing

## Scenario 1: Resource Quota

Suppose we have a Kubernetes cluster with multiple namespaces. We use separate namespaces for:

* Resource isolation
* Team-level separation
* Environment separation
* Application isolation

For example, we may have separate namespaces for:

* Development
* QA
* Staging
* Production
* Individual application teams

Each team is allocated a specific amount of CPU and memory resources. They should not be allowed to consume resources beyond their allocated quota without proper approval.

### Real-Time Problem

Suppose we have the following cluster:

```text
Cluster Capacity:
    CPU    = 100 cores
    Memory = 100 GiB
```

And we have multiple namespaces:

```text
Cluster
│
├── Namespace A
│   ├── Application A
│   └── Application B
│
├── Namespace B
│   ├── Application C
│   └── Application D
│
├── Namespace C
│   ├── Application E
│   └── Application F
│
└── Namespace D
    ├── Application G
    └── Application H
```

Suppose Namespace A starts consuming a significant amount of cluster resources.

As a result, other namespaces may not have enough available CPU or memory to schedule their Pods.

For example:

```text
Namespace A
    |
    +-- CPU: 15 cores
    +-- Memory: 15 GiB

Namespace B
    |
    +-- CPU: 15 cores
    +-- Memory: 15 GiB

Namespace C
    |
    +-- CPU: 15 cores
    +-- Memory: 15 GiB
```

If the cluster does not have sufficient **allocatable resources** to satisfy the resource requests of newly created Pods, those Pods may remain in the:

```text
Pending
```

state because the Kubernetes scheduler cannot find a suitable node.

In some situations, existing workloads can also experience failures due to memory pressure. Containers may be terminated with an:

```text
OOMKilled
```

reason.

If a container repeatedly terminates and Kubernetes continues restarting it, the Pod may eventually show:

```text
CrashLoopBackOff
```

---

# DevOps Responsibilities

As a DevOps engineer, our responsibilities include:

1. Understanding the resource requirements of each namespace.
2. Configuring appropriate `ResourceQuota` objects.
3. Configuring CPU and memory requests and limits.
4. Monitoring actual resource consumption.
5. Ensuring sufficient cluster capacity.
6. Working with the development team when applications require additional resources.
7. Identifying resource bottlenecks.
8. Providing troubleshooting information to the development team.

---

# Step 1: Configure ResourceQuota

For each namespace, we can configure a `ResourceQuota` to control the aggregate amount of CPU and memory that workloads in that namespace can request or consume.

Example:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payment-quota
  namespace: payment
spec:
  hard:
    requests.cpu: "15"
    requests.memory: 15Gi
    limits.cpu: "15"
    limits.memory: 15Gi
```

The important point is:

> `ResourceQuota` is namespace-scoped.

It controls the aggregate resource usage of workloads within a particular namespace.

---

# How Do We Decide the Resource Values?

Before configuring the quota, we should discuss the application's requirements with the development team.

The development team should ideally provide resource requirements based on:

* Performance testing
* Load testing
* Historical production metrics
* Application profiling
* Expected traffic
* Peak traffic
* CPU utilization
* Memory utilization
* Application behavior under load

For example, the development team may provide the following information:

> Based on performance and load testing, the application requires approximately 15 CPU cores and 15 GiB of memory under the expected peak workload.

We should use these measurements as a starting point rather than selecting arbitrary values.

However, we should also validate these values against actual production metrics.

A benchmark value is not necessarily a permanent requirement.

---

# Important: ResourceQuota Does Not Create Resources

One common misunderstanding is that increasing `ResourceQuota` automatically provides additional CPU and memory.

It does not.

For example:

```text
Cluster Capacity
----------------
100 CPU
100 GiB Memory
```

If we configure:

```text
Namespace A
-----------
Quota:
15 CPU
15 GiB Memory
```

the quota only establishes the maximum allowed resource usage for that namespace.

It does not reserve 15 physical CPU cores and 15 GiB of RAM exclusively for that namespace.

---

# What Happens If the Cluster Does Not Have Enough Capacity?

Suppose the cluster has:

```text
100 CPU cores
100 GiB memory
```

But the workloads' resource requests require more capacity than the cluster can provide.

Increasing the namespace quota alone will not solve the problem.

We need to determine whether:

* Existing nodes have enough allocatable resources.
* Pods have unnecessarily high resource requests.
* Actual workload usage has increased.
* Cluster autoscaling is configured.
* Additional worker nodes are required.
* The workload is over-provisioned.
* Some workloads can be optimized.

If the cluster genuinely needs additional capacity, we can:

* Scale the node pool.
* Add worker nodes.
* Configure or tune Cluster Autoscaler.
* Review resource requests.
* Optimize workloads.

Therefore, the correct approach is:

```text
ResourceQuota
      +
Resource Requests/Limits
      +
Monitoring
      +
Appropriate Cluster Capacity
      +
Autoscaling
```

---

# Scenario 1 Summary

The first level of isolation is:

```text
Cluster
   |
   +-- Namespace A
   +-- Namespace B
   +-- Namespace C
   +-- Namespace D
```

`ResourceQuota` prevents a namespace from requesting or consuming resources beyond its configured quota.

However, namespace-level quota alone does not control individual containers.

That leads us to Scenario 2.

---

# Scenario 2: Resource Limits and Pod-Level Isolation

Now suppose we have allocated:

```text
CPU    = 15 cores
Memory = 15 GiB
```

to a particular namespace.

However, that namespace contains five microservices:

```text
Payment Namespace
│
├── Payment API
├── Order Service
├── Notification Service
├── Reporting Service
└── Fraud Service
```

Suppose one of these microservices has a memory leak.

If appropriate container-level resource limits are not configured, that application could consume a significant amount of the namespace's available memory.

This could negatively affect other workloads running in the same namespace.

---

# Blast Radius

Without proper namespace isolation:

```text
Problematic Workload
        |
        v
Potentially affects the wider cluster
```

With namespace-level isolation:

```text
Problematic Workload
        |
        v
Primarily affects its namespace
```

However, namespace isolation alone is not sufficient.

We also need to control resource consumption at the container level.

---

# ResourceQuota vs LimitRange

This distinction is extremely important in Kubernetes.

## ResourceQuota

`ResourceQuota` controls the **aggregate resource consumption of a namespace**.

For example:

```text
Namespace: payment

Total Allowed:
----------------
CPU    = 15 cores
Memory = 15 GiB
```

Multiple Pods inside that namespace collectively consume resources within the configured quota.

---

## LimitRange

`LimitRange` defines default and/or minimum/maximum resource requests and limits for containers within a namespace.

Example:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: application-limits
  namespace: payment
spec:
  limits:
    - type: Container
      default:
        cpu: "1"
        memory: "1Gi"
      defaultRequest:
        cpu: "500m"
        memory: "512Mi"
      max:
        cpu: "4"
        memory: "4Gi"
```

This can help prevent individual containers from consuming resources beyond an established boundary.

---

# ResourceQuota vs LimitRange vs Requests vs Limits

| Kubernetes Feature | Scope     | Purpose                                                 |
| ------------------ | --------- | ------------------------------------------------------- |
| `ResourceQuota`    | Namespace | Controls aggregate resource usage                       |
| `LimitRange`       | Namespace | Defines/defaults container resource requests and limits |
| Resource Request   | Container | Used by scheduler for placement decisions               |
| Resource Limit     | Container | Restricts maximum resource consumption                  |
| Node Capacity      | Node      | Determines available CPU and memory on a node           |

---

# Different Microservices Have Different Resource Requirements

Not every microservice requires the same amount of CPU and memory.

For example:

| Microservice         | CPU Request | Memory Request | CPU Limit | Memory Limit |
| -------------------- | ----------: | -------------: | --------: | -----------: |
| Payment API          |        500m |          512Mi |         2 |          2Gi |
| Order Service        |           1 |            1Gi |         4 |          4Gi |
| Notification Service |        200m |          256Mi |         1 |          1Gi |
| Reporting Service    |           2 |            2Gi |         4 |          4Gi |

These values should ideally be determined using:

* Performance testing
* Load testing
* Historical metrics
* Application profiling
* Production monitoring
* Expected traffic patterns
* Peak traffic requirements

The DevOps team can then configure appropriate requests and limits.

---

# Resource Requests

A resource request represents the amount of CPU or memory that the container expects to need.

For example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The Kubernetes scheduler uses resource requests when determining whether a Pod can be scheduled onto a node.

For example:

```text
Node
-------------------------
Allocatable CPU: 8 cores
Allocatable Memory: 16 GiB

Existing Requests:
CPU: 6 cores
Memory: 12 GiB

Available for Scheduling:
CPU: 2 cores
Memory: 4 GiB
```

If a new Pod requests:

```text
CPU: 3 cores
Memory: 5 GiB
```

the scheduler cannot place the Pod on this node because the node does not have enough allocatable requested resources.

The Pod may remain:

```text
Pending
```

---

# Resource Limits

A resource limit specifies the maximum amount of CPU or memory that a container can consume.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

For memory, exceeding the configured limit can result in the container being terminated with:

```text
OOMKilled
```

CPU behaves differently.

If a container reaches its CPU limit, it is generally throttled rather than killed for exceeding the CPU limit.

---

# What If One Microservice Still Has a Memory Leak?

Now suppose we have configured everything correctly:

* ResourceQuota
* LimitRange
* CPU requests
* Memory requests
* CPU limits
* Memory limits
* Namespace isolation
* Monitoring

But one application still has a memory leak.

For example:

```text
Cluster
│
├── Namespace: Payment
│   │
│   ├── Payment API
│   ├── Order Service
│   ├── Notification Service
│   ├── Reporting Service
│   └── Fraud Service  <-- Memory Leak
│
└── Other Namespaces
```

The problematic application may continuously consume memory until it reaches its container memory limit.

At that point, Kubernetes/Linux may terminate the container with:

```text
OOMKilled
```

If the container repeatedly terminates, the Pod may enter:

```text
CrashLoopBackOff
```

---

# Important Point About OOMKilled

We should not automatically assume:

> `OOMKilled` means there is definitely a memory leak.

There are several possible causes:

* Memory leak
* Memory limit is too low
* Legitimately high-memory workload
* Incorrect JVM heap configuration
* High native/off-heap memory usage
* Excessive object allocation
* Large requests being processed simultaneously
* Cache growth
* Inefficient application behavior
* Unexpected traffic increase

Therefore, we need evidence before concluding that a memory leak exists.

---

# DevOps and Development Responsibilities

At this point, Kubernetes resource controls are doing their job.

They prevent the problematic container from consuming unlimited memory.

However, the application itself still has a problem.

This is where the DevOps and Development teams need to work together.

---

## DevOps Responsibilities

The DevOps team should provide:

* Infrastructure configuration
* Kubernetes resource configuration
* Resource isolation
* Monitoring
* Logging
* Troubleshooting
* Deployment support
* Resource utilization metrics
* Pod events
* Container termination information

Useful information includes:

* Pod events
* Container termination reason
* Exit code
* CPU metrics
* Memory metrics
* Container logs
* Previous container logs
* Restart count
* Memory utilization graphs
* Deployment version
* Resource requests
* Resource limits
* JVM metrics
* Thread dumps, when appropriate
* Heap dumps, when available

---

## Development Responsibilities

The Development team should:

* Analyze application behavior.
* Identify application-level problems.
* Perform Root Cause Analysis (RCA).
* Investigate memory leaks.
* Analyze heap dumps.
* Analyze thread dumps when relevant.
* Fix application code.
* Test the fix.
* Create and track a Jira ticket.
* Provide a new application version.

---

# DevOps and Development Collaboration

The process can be represented as:

```text
DevOps
  |
  +-- Infrastructure
  +-- Kubernetes configuration
  +-- Resource management
  +-- Monitoring
  +-- Troubleshooting evidence
  +-- Deployment
          |
          v
Development
  |
  +-- Application analysis
  +-- Root Cause Analysis
  +-- Code/Application fix
  +-- Testing
          |
          v
     New Version
          |
          v
       Deployment
          |
          v
       Monitoring
```

---

# Challenge 2: OOMKilled Issue with a Pod

Suppose we have a Java application running inside a Kubernetes Pod.

We execute:

```bash
kubectl get pods
```

We may see:

```text
NAME                         READY   STATUS             RESTARTS
payment-api-7d8f9c7b8-x2abc  0/1     CrashLoopBackOff   10
```

At this point, `CrashLoopBackOff` tells us that the container is repeatedly failing and Kubernetes is applying an increasing restart backoff.

It does **not by itself tell us the root cause**.

We need to investigate further.

---

# Step 1: Describe the Pod

Run:

```bash
kubectl describe pod <pod_name>
```

We should check:

* `Events`
* `Last State`
* Container status
* Termination reason
* Exit code
* Restart count

We may find:

```text
Reason: OOMKilled
Exit Code: 137
```

This tells us that the container was terminated due to an out-of-memory condition.

---

# Step 2: Check Previous Container Logs

If the container has restarted, use:

```bash
kubectl logs <pod_name> --previous
```

This is useful because it allows us to inspect logs from the previous container instance.

For example:

```bash
kubectl logs payment-api-7d8f9c7b8-x2abc --previous
```

---

# Step 3: Check the Pod Configuration

Run:

```bash
kubectl get pod <pod_name> -o yaml
```

Look for:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"

  limits:
    cpu: "2"
    memory: "2Gi"
```

We need to understand:

* What is the CPU request?
* What is the memory request?
* What is the CPU limit?
* What is the memory limit?
* What is the actual memory consumption?
* Is the memory limit too low?
* Is the application consuming memory unexpectedly?

---

# Step 4: Check Pod Events

Run:

```bash
kubectl describe pod <pod_name>
```

Look at the Events section.

For example:

```text
Events:
  Type     Reason     Message
  ----     ------     -------
  Normal   Started    Started container payment-api
  Warning  BackOff    Back-off restarting failed container
```

The container status may show:

```text
Last State:
  Terminated:
    Reason: OOMKilled
    Exit Code: 137
```

---

# Step 5: Check Resource Usage

If Metrics Server is available, we can check current resource consumption:

```bash
kubectl top pod <pod_name>
```

Example:

```text
NAME                         CPU(cores)   MEMORY(bytes)
payment-api-7d8f9c7b8-x2abc  1500m        1.9Gi
```

We can also check node-level usage:

```bash
kubectl top nodes
```

Example:

```text
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-01    3500m        44%    12Gi            75%
node-02    6000m        75%    14Gi            87%
```

This helps us understand whether the problem is isolated to the application or whether the cluster itself is under resource pressure.

---

# How Do We Troubleshoot OOMKilled?

Suppose the application is a Java microservice.

The first thing we should **not** assume is:

> `OOMKilled` means there is definitely a memory leak.

There can be several reasons for an `OOMKilled` event:

1. The application genuinely has a memory leak.
2. The configured memory limit is too low.
3. The application has a legitimate high-memory workload.
4. JVM heap configuration is inappropriate for the container.
5. Native/off-heap memory consumption is high.
6. Excessive object creation is occurring.
7. Large requests are being processed simultaneously.
8. A cache is consuming too much memory.
9. Traffic has unexpectedly increased.
10. A third-party library has abnormal memory behavior.

Therefore, we need evidence before concluding that there is a memory leak.

---

# Java Application Troubleshooting

Suppose the Pod is running a Java application.

There are two important diagnostic mechanisms:

1. Thread Dump
2. Heap Dump

They provide different types of information.

---

# Thread Dump

A thread dump is a snapshot of the application's threads at a particular point in time.

It helps us investigate:

* Blocked threads
* Deadlocks
* High CPU situations
* Waiting threads
* Thread contention
* Thread states
* Application execution flow

Common commands include:

```bash
jstack <PID>
```

or:

```bash
kill -3 <PID>
```

`kill -3` sends `SIGQUIT` to the Java process, and the JVM typically prints a thread dump to its standard output.

---

# Finding the Java Process ID

Inside the container, we may use:

```bash
ps -ef
```

or:

```bash
jps
```

if the required JDK tools are available.

For example:

```text
PID    Process
1      java -jar payment-api.jar
```

Then:

```bash
jstack 1
```

or:

```bash
kill -3 1
```

---

# Using kubectl exec

If the necessary tools are available inside the container, we can enter the Pod:

```bash
kubectl exec -it <pod_name> -- sh
```

Then:

```bash
ps -ef
```

and:

```bash
kill -3 <PID>
```

However, production environments require caution.

Before executing diagnostic commands in production, we should follow the organization's operational procedures and make sure the diagnostic action will not negatively affect the application.

---

# Heap Dump

A heap dump is different from a thread dump.

A heap dump captures information about objects allocated in the JVM heap.

It can help developers investigate:

* Excessive object creation
* Objects consuming large amounts of memory
* Unexpected object retention
* Memory leaks
* Cache-related memory growth
* Dominant objects
* Object references

Common commands include:

```bash
jmap -dump:format=b,file=heap.hprof <PID>
```

or:

```bash
jcmd <PID> GC.heap_dump heap.hprof
```

`jcmd` is generally preferred for many modern JVM diagnostic operations.

---

# Thread Dump vs Heap Dump

| Diagnostic                | Purpose     | Useful For                                          |
| ------------------------- | ----------- | --------------------------------------------------- |
| `jstack <PID>`            | Thread dump | Deadlocks, blocked threads, high CPU, thread issues |
| `kill -3 <PID>`           | Thread dump | Capturing JVM thread information                    |
| `jmap -dump...`           | Heap dump   | Memory/object analysis                              |
| `jcmd <PID> GC.heap_dump` | Heap dump   | Modern JVM heap analysis                            |

The key difference is:

```text
Thread Dump
    |
    +-- What are the threads doing?
    +-- Are threads blocked?
    +-- Is there a deadlock?
    +-- Why is CPU high?
    +-- What are the thread states?

Heap Dump
    |
    +-- What objects are consuming memory?
    +-- What objects are retained?
    +-- Is there a possible memory leak?
    +-- Which objects dominate the heap?
```

---

# Important Kubernetes Consideration

There is an important limitation when troubleshooting `OOMKilled`.

If the **container itself is killed by the Linux cgroup OOM mechanism**, the JVM process may be terminated immediately.

In that situation, we may not have an opportunity to generate a heap dump after the `OOMKilled` event.

Therefore, for production Java applications, it is often better to plan memory diagnostics proactively.

Possible approaches include:

* JVM heap-dump-on-OOM configuration
* Java Flight Recorder (JFR)
* JVM metrics
* Prometheus
* Grafana
* Application Performance Monitoring (APM)
* Continuous profiling
* Garbage Collection (GC) logs
* Application-level memory metrics

This gives the development team evidence before or during the memory-growth problem rather than trying to collect everything after the container has already been killed.

---

# JVM Memory Considerations

For Java applications running inside containers, we should also consider the difference between:

```text
Container Memory
       |
       +-- JVM Heap
       |
       +-- Metaspace
       |
       +-- Thread Stacks
       |
       +-- Direct Buffers
       |
       +-- Native Memory
       |
       +-- Other JVM/Process Memory
```

Therefore, setting:

```text
Container memory limit = 2 GiB
```

does not necessarily mean:

```text
JVM heap = 2 GiB
```

The JVM also requires memory outside the Java heap.

Therefore, JVM memory configuration should be designed with the container memory limit in mind.

---

# Sharing the Findings with the Development Team

Once we collect the available diagnostic information, we should share it with the development team.

For example:

```text
Pod: payment-api-7d8f9c7b8-x2abc
Namespace: payment
Container: payment-api

Application Version:
v2.4.1

Restart Count:
10

Termination Reason:
OOMKilled

Exit Code:
137

Memory Limit:
2Gi

Observed Memory Usage:
~2Gi
```

We can also provide:

* Relevant logs
* Previous container logs
* Pod events
* Memory utilization graphs
* JVM metrics
* GC information
* Thread dumps
* Heap dumps, if available
* Deployment configuration
* Application version
* Timeline of when the issue started
* Traffic/load information
* Resource request and limit configuration

---

# Development Team Analysis

The development team can then analyze the application behavior and determine whether the root cause is:

* Memory leak
* Incorrect JVM configuration
* Excessive object allocation
* Large in-memory data processing
* Cache growth
* Unexpected traffic pattern
* Application bug
* Third-party library issue
* Insufficient container memory limit
* Native memory consumption
* Incorrect application configuration

The development team can create a Jira ticket and perform the Root Cause Analysis (RCA).

The RCA should ideally answer:

```text
What happened?
       |
       v
Why did it happen?
       |
       v
Which component caused it?
       |
       v
Why was it not detected earlier?
       |
       v
What is the permanent fix?
       |
       v
How will we prevent it from happening again?
```

---

# Application Fix and Deployment

Once the development team identifies and fixes the issue, they should:

1. Implement the application fix.
2. Perform unit testing.
3. Perform integration testing.
4. Perform load/performance testing where appropriate.
5. Build a new application version.
6. Publish the new image.
7. Provide the new version through the CI/CD pipeline.

For example:

```text
Old Version
v2.4.1
    |
    v
Memory Issue Identified
    |
    v
Code Fix
    |
    v
Testing
    |
    v
New Version
v2.4.2
```

The DevOps team can then deploy the new version through the organization's CI/CD process.

---

# Post-Deployment Monitoring

After deploying the new version, we should monitor:

* Pod restarts
* Memory utilization
* CPU utilization
* JVM heap usage
* Garbage Collection
* Request rate
* Response time
* Error rate
* OOMKilled events
* Application logs
* Node resource usage

For example:

```text
Before Fix
----------

Memory Usage
   |
2Gi|                 ███
   |              █████
   |           ███████
   |        █████████
   |     ███████████
   +----------------------> Time


After Fix
---------

Memory Usage
   |
2Gi|
   |
   |       ███ ███ ███
   |      █████████████
   |     ██████████████
   +----------------------> Time
```

The goal is to verify that memory usage is stable after the fix.

---

# Real-Time Troubleshooting Flow

The complete troubleshooting approach can be summarized as:

```text
                    Application Pod
                          |
                          v
                 kubectl get pods
                          |
                          v
                 CrashLoopBackOff?
                          |
                          v
              kubectl describe pod
                          |
                          v
              Check Events + Status
                          |
                          v
                     OOMKilled?
                    /          \
                  No            Yes
                  |              |
                  v              v
          Investigate       Check Resource
          other causes      Requests/Limits
                                 |
                                 v
                         Check Memory Usage
                                 |
                                 v
                         Check Application
                              Logs
                                 |
                                 v
                      kubectl logs --previous
                                 |
                                 v
                       Check JVM Metrics
                                 |
                                 v
                         Check GC Activity
                                 |
                                 v
                    Thread Dump / Heap Dump
                    when technically possible
                                 |
                                 v
                      Share Evidence with
                       Development Team
                                 |
                                 v
                     Development Performs
                              RCA
                                 |
                                 v
                        Fix Application
                                 |
                                 v
                     Build New Version
                                 |
                                 v
                      Deploy Through CI/CD
                                 |
                                 v
                       Monitor Application
```

---

# Overall Resource Isolation Model

The overall Kubernetes resource isolation model can be visualized as:

```text
                         Kubernetes Cluster
                                |
          +---------------------+---------------------+
          |                     |                     |
          v                     v                     v
    Namespace A           Namespace B           Namespace C
          |                     |                     |
     ResourceQuota         ResourceQuota         ResourceQuota
          |                     |                     |
     LimitRange             LimitRange             LimitRange
          |                     |                     |
          v                     v                     v
       Pods                  Pods                  Pods
          |                     |                     |
          v                     v                     v
     Containers             Containers             Containers
          |                     |                     |
          v                     v                     v
    CPU / Memory           CPU / Memory           CPU / Memory
     Requests/Limits        Requests/Limits        Requests/Limits
```

---

# Kubernetes Resource Management Hierarchy

A simplified hierarchy is:

```text
Cluster
   |
   +-- Nodes
        |
        +-- Namespaces
              |
              +-- ResourceQuota
              |
              +-- LimitRange
              |
              +-- Pods
                    |
                    +-- Containers
                          |
                          +-- Requests
                          |
                          +-- Limits
```

---

# ResourceQuota

`ResourceQuota` controls the aggregate resource consumption of a namespace.

Example:

```text
Namespace: payment

Total Quota
------------------
CPU Requests:    15
Memory Requests: 15Gi

CPU Limits:      15
Memory Limits:   15Gi
```

If the namespace reaches its configured quota, new workloads that would exceed the quota may be rejected by the Kubernetes API.

---

# LimitRange

`LimitRange` defines resource constraints and defaults for individual containers within a namespace.

Example:

```text
Namespace: payment

Container Defaults
------------------
CPU Request:       500m
Memory Request:    512Mi

CPU Limit:         1
Memory Limit:      1Gi

Maximum:
CPU:               4
Memory:            4Gi
```

---

# Resource Requests

Resource requests are primarily used by the Kubernetes scheduler.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

The scheduler uses these requested resources when deciding where to place a Pod.

---

# Resource Limits

Resource limits establish the maximum resource consumption for a container.

Example:

```yaml
resources:
  limits:
    cpu: "2"
    memory: "2Gi"
```

Important behavior:

```text
CPU Limit
    |
    +-- Container may be throttled

Memory Limit
    |
    +-- Container may be terminated with OOMKilled
```

---

# Node Capacity

Node capacity determines whether the cluster has enough allocatable resources to schedule workloads.

For example:

```text
Worker Node
-------------------------
CPU Capacity:       8 cores
Memory Capacity:    32 GiB

Allocatable:
CPU:                7.5 cores
Memory:             30 GiB
```

Kubernetes schedules workloads based on the available allocatable resources and the Pods' resource requests.

---

# Important Difference: Quota vs Capacity

These concepts should not be confused.

```text
ResourceQuota
     |
     v
Controls how much a namespace is allowed to request/use


Node Capacity
     |
     v
Determines how much resource the cluster actually has available
```

For example:

```text
Namespace Quota:
20 CPU

Cluster Available:
10 CPU
```

The namespace may have permission for 20 CPU, but the cluster currently has only 10 CPU available.

The quota does not create the missing 10 CPU.

---

# Important Difference: OOMKilled vs CrashLoopBackOff

These two statuses are also different.

## OOMKilled

`OOMKilled` is a container termination reason.

Example:

```text
Last State:
  Terminated:
    Reason: OOMKilled
```

It indicates that the container was terminated because of an out-of-memory condition.

---

## CrashLoopBackOff

`CrashLoopBackOff` indicates that Kubernetes is repeatedly restarting a failing container and applying an increasing delay between restart attempts.

It is not the root cause.

For example:

```text
Application crashes
       |
       v
Container exits
       |
       v
Kubernetes restarts container
       |
       v
Container crashes again
       |
       v
Repeated failures
       |
       v
CrashLoopBackOff
```

The actual reason for the crash could be:

* `OOMKilled`
* Application exception
* Configuration error
* Missing environment variable
* Failed dependency
* Failed health check
* Permission issue
* Invalid command
* Application startup failure

Therefore, we should investigate the actual container termination reason.

---

# Important Technical Corrections and Best Practices

## 1. ResourceQuota Is Namespace-Scoped

Incorrect:

```text
ResourceQuota = Cluster Level
```

Correct:

```text
ResourceQuota = Namespace Scoped
```

A cluster can have multiple namespaces, and each namespace can have its own quota.

---

## 2. LimitRange Is Also Namespace-Scoped

`LimitRange` is configured for a namespace and applies resource defaults and constraints to containers within that namespace.

---

## 3. Requests Are Used for Scheduling

CPU and memory requests are important to the Kubernetes scheduler.

Example:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
```

The scheduler uses these values when determining whether a Pod can fit on a node.

---

## 4. Limits Restrict Container Consumption

Example:

```yaml
resources:
  limits:
    cpu: "2"
    memory: "2Gi"
```

The container can consume resources up to the configured limits, subject to Kubernetes and Linux resource-management behavior.

---

## 5. OOMKilled Does Not Always Mean Memory Leak

This is one of the most important troubleshooting points.

Do not immediately conclude:

```text
OOMKilled = Memory Leak
```

Instead:

```text
OOMKilled
   |
   +-- Check memory limit
   |
   +-- Check actual usage
   |
   +-- Check application behavior
   |
   +-- Check JVM configuration
   |
   +-- Check traffic
   |
   +-- Check JVM metrics
   |
   +-- Check heap
   |
   +-- Check application logs
   |
   +-- Investigate memory leak
```

---

## 6. CrashLoopBackOff Is Not the Root Cause

`CrashLoopBackOff` tells us that the container is repeatedly failing.

We need to investigate:

```bash
kubectl describe pod <pod_name>
```

and:

```bash
kubectl logs <pod_name> --previous
```

to determine the actual reason.

---

## 7. A New Worker Node Is Not Always the Solution

If a Pod cannot be scheduled, do not immediately assume:

> We need to add a worker node.

First check:

```text
Why is the Pod Pending?
        |
        +-- Insufficient CPU?
        |
        +-- Insufficient memory?
        |
        +-- Incorrect resource requests?
        |
        +-- Taints?
        |
        +-- Node selectors?
        |
        +-- Affinity rules?
        |
        +-- Pod topology constraints?
        |
        +-- Quota exceeded?
        |
        +-- Other scheduling constraints?
```

Only after understanding the actual constraint should we decide whether additional capacity is required.

---

# Production Troubleshooting Checklist

When a Pod enters `CrashLoopBackOff`, use the following checklist.

* [ ] Run `kubectl get pods`
* [ ] Check Pod restart count
* [ ] Run `kubectl describe pod <pod_name>`
* [ ] Check the `Events` section
* [ ] Check container `Last State`
* [ ] Check termination reason
* [ ] Check exit code
* [ ] Run `kubectl logs <pod_name>`
* [ ] Run `kubectl logs <pod_name> --previous`
* [ ] Check CPU requests
* [ ] Check memory requests
* [ ] Check CPU limits
* [ ] Check memory limits
* [ ] Check current Pod resource usage
* [ ] Run `kubectl top pod <pod_name>` if Metrics Server is available
* [ ] Check node resource utilization
* [ ] Run `kubectl top nodes` if Metrics Server is available
* [ ] Check ResourceQuota
* [ ] Check LimitRange
* [ ] Check application metrics
* [ ] Check JVM metrics for Java applications
* [ ] Check GC behavior
* [ ] Collect a thread dump when appropriate
* [ ] Collect a heap dump when technically possible
* [ ] Share evidence with the Development team
* [ ] Perform Root Cause Analysis
* [ ] Fix the application
* [ ] Build a new version
* [ ] Deploy through CI/CD
* [ ] Monitor the new version

---

# Useful Kubernetes Commands

## Check Pods

```bash
kubectl get pods
```

```bash
kubectl get pods -n <namespace>
```

---

## Check Detailed Pod Information

```bash
kubectl describe pod <pod_name> -n <namespace>
```

---

## Check Pod YAML

```bash
kubectl get pod <pod_name> -n <namespace> -o yaml
```

---

## Check Logs

```bash
kubectl logs <pod_name> -n <namespace>
```

---

## Check Previous Container Logs

```bash
kubectl logs <pod_name> -n <namespace> --previous
```

---

## Check Resource Usage

```bash
kubectl top pod <pod_name> -n <namespace>
```

---

## Check Node Resource Usage

```bash
kubectl top nodes
```

---

## Check ResourceQuota

```bash
kubectl get resourcequota -n <namespace>
```

Detailed information:

```bash
kubectl describe resourcequota -n <namespace>
```

---

## Check LimitRange

```bash
kubectl get limitrange -n <namespace>
```

Detailed information:

```bash
kubectl describe limitrange -n <namespace>
```

---

## Enter a Container

```bash
kubectl exec -it <pod_name> -n <namespace> -- sh
```

If Bash is available:

```bash
kubectl exec -it <pod_name> -n <namespace> -- bash
```

---

# Example Resource Configuration

A typical application Deployment may look like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: payment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      containers:
        - name: payment-api
          image: payment-api:v2.4.2
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
```

---

# Complete Resource Management Example

The overall configuration may look like:

```text
                    Kubernetes Cluster
                           |
                           v
                    Worker Nodes
                           |
                           v
                      Namespace
                           |
             +-------------+-------------+
             |                           |
             v                           v
       ResourceQuota                LimitRange
             |                           |
             |                           |
             +-------------+-------------+
                           |
                           v
                         Pods
                           |
                           v
                      Containers
                           |
             +-------------+-------------+
             |                           |
             v                           v
        CPU/Memory                  CPU/Memory
         Requests                    Limits
             |                           |
             v                           v
       Pod Scheduling             Consumption Control
                                         |
                                         v
                                    OOMKilled
```

---

# Final Technical Understanding

The complete relationship between Kubernetes resource management and application troubleshooting can be summarized as:

```text
                    Kubernetes Cluster
                           |
                           v
                     ResourceQuota
                           |
                           v
                       Namespace
                           |
                           v
                      LimitRange
                           |
                           v
                    Pod / Container
                           |
              +------------+------------+
              |                         |
              v                         v
       Resource Request          Resource Limit
              |                         |
              v                         v
        Pod Scheduling          Consumption Limit
              |                         |
              v                         v
           Node                    OOMKilled
                                      |
                                      v
                               Application Issue
                                      |
                                      v
                               Troubleshooting
                                      |
                                      v
                              Development Team
                                      |
                                      v
                                  RCA / Fix
                                      |
                                      v
                              New Application
                                  Version
                                      |
                                      v
                                   CI/CD
                                      |
                                      v
                                  Deploy
                                      |
                                      v
                                Monitoring
```

---

# Key Takeaways

The most important concepts to remember are:

1. **ResourceQuota is namespace-scoped.**
2. **LimitRange is namespace-scoped.**
3. **Resource requests influence Pod scheduling.**
4. **Resource limits constrain container resource consumption.**
5. **Namespace isolation reduces the blast radius of problematic workloads.**
6. **ResourceQuota does not create physical resources.**
7. **Additional worker nodes are required only when the cluster genuinely lacks sufficient capacity.**
8. **`OOMKilled` does not automatically mean there is a memory leak.**
9. **`CrashLoopBackOff` is a restart/backoff condition, not the root cause itself.**
10. **Thread dumps and heap dumps provide different types of diagnostic information.**
11. **A thread dump is useful for investigating thread behavior, deadlocks, blocked threads, and high CPU situations.**
12. **A heap dump is useful for investigating JVM memory usage and possible memory leaks.**
13. **A container-level OOM kill may happen before the JVM can generate a heap dump interactively.**
14. **Java applications should be configured and monitored with container memory constraints in mind.**
15. **Application-level memory problems ultimately need to be investigated and fixed by the Development team.**
16. **DevOps should provide the infrastructure configuration, monitoring, troubleshooting evidence, and deployment support required to resolve the issue.**
17. **After deploying the fix, we must monitor the application to verify that the problem has actually been resolved.**

---

# Final Real-Time Scenario

A concise real-world explanation would be:

```text
One of our production Pods is showing CrashLoopBackOff.

First, I check:

kubectl get pods

Then I investigate the Pod:

kubectl describe pod <pod_name>

I check the Events and Last State.

If I find:

Reason: OOMKilled
Exit Code: 137

I then check:

kubectl logs <pod_name> --previous

I verify the resource requests and limits.

I also check current resource utilization:

kubectl top pod <pod_name>

Then I check:

- ResourceQuota
- LimitRange
- Memory utilization
- CPU utilization
- JVM metrics
- GC behavior
- Application logs
- Traffic patterns

If it is a Java application, I collect a thread dump or
heap dump when technically possible and appropriate.

I provide all relevant evidence to the Development team.

The Development team performs the Root Cause Analysis and
determines whether the issue is related to:

- Memory leak
- JVM configuration
- Application code
- Cache growth
- Excessive object allocation
- High traffic
- Insufficient memory limit
- Native memory consumption
- Third-party library behavior

After the issue is fixed, Development provides a new
application version.

We deploy the new version through the CI/CD pipeline and
continue monitoring the application.

The goal is not simply to restart the Pod or increase the
memory limit.

The goal is to identify and fix the actual root cause.
```

---

# Conclusion

Kubernetes provides multiple mechanisms to control and isolate resource consumption.

At the namespace level, `ResourceQuota` controls the aggregate amount of resources that workloads can request or consume.

At the container level, `LimitRange` can establish default and maximum/minimum resource requirements.

Resource **requests** help Kubernetes determine where a Pod can be scheduled, while resource **limits** restrict the maximum resources a container can consume.

These controls significantly reduce the blast radius of a problematic workload.

However, Kubernetes resource management cannot fix application-level problems.

For example, if a Java application has a memory leak, Kubernetes can restrict the container's memory consumption and eventually terminate it with `OOMKilled`, but Kubernetes cannot fix the underlying memory leak.

Therefore, production troubleshooting requires collaboration between DevOps and Development.

The DevOps team is responsible for infrastructure, Kubernetes configuration, resource management, monitoring, troubleshooting evidence, and deployment.

The Development team is responsible for analyzing and fixing application-level problems.

The complete production troubleshooting cycle is:

```text
Detect
  |
  v
Investigate
  |
  v
Collect Evidence
  |
  v
Identify Root Cause
  |
  v
Fix Application
  |
  v
Test
  |
  v
Deploy
  |
  v
Monitor
  |
  v
Verify Resolution
```

> **Kubernetes resource controls can limit the blast radius of a problematic workload, but they cannot fix an application-level memory leak.**

The ultimate goal of production troubleshooting is not just to make the Pod `Running` again. The goal is to **identify the root cause, fix the underlying problem, prevent recurrence, and verify the solution through monitoring.**
