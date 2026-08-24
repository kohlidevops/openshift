# OpenShift Learning – Lesson 10: Resource Requests, Limits & Autoscaling

## 🎯 Objectives

In this lesson, I learned and practiced:

- CPU and Memory resources
- Resource Requests
- Resource Limits
- Requests vs Limits
- CPU throttling
- Memory limits
- `OOMKilled`
- QoS Classes
- Horizontal Pod Autoscaler (HPA)
- HPA minimum and maximum replicas
- CPU-based autoscaling
- Resource troubleshooting

---
## 1. Why Do We Need Resource Requests and Limits?

Multiple applications may run on the same OpenShift worker node.

```text
OpenShift Worker Node
+-----------------------------+
| Application A               |
| Application B               |
| Application C               |
| Application D               |
+-----------------------------+
```

If one application consumes too much CPU or memory, it can affect other applications.

Resource requests and limits help OpenShift manage resource usage.

```text
Application
    |
    +---- CPU Request
    +---- CPU Limit
    +---- Memory Request
    +---- Memory Limit
```

---
## 2. CPU and Memory

CPU examples:

```text
100m
250m
500m
1
```

`1000m` is approximately:

```text
1 CPU
```

`500m` is:

```text
0.5 CPU
```

Memory examples:

```text
128Mi
256Mi
512Mi
1Gi
```

---
## 3. Resource Requests

A request tells OpenShift:

> This container needs this amount of resource.

Example:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
```

The scheduler uses resource requests when deciding where to place the Pod.

```text
Pod
 |
 +---- CPU Request: 100m
 |
 +---- Memory Request: 128Mi
```

---
## 4. Resource Limits

A limit tells OpenShift:

> This container should not use more than this amount.

Example:

```yaml
resources:
  limits:
    cpu: 500m
    memory: 256Mi
```

Conceptually:

```text
Request → Expected resource requirement

Limit   → Maximum resource usage
```

---
## 5. Requests vs Limits

| Request | Limit |
|---|---|
| Used mainly for scheduling | Maximum resource usage |
| Helps scheduler select a node | Restricts container usage |
| Represents expected requirement | Represents maximum allowed usage |

Easy way to remember:

```text
Request → "I need this much"

Limit → "Don't let me go beyond this"
```

---
## 6. Create Application with Resources

Created `lesson10-resources.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson10-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson10-web
  template:
    metadata:
      labels:
        app: lesson10-web
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

Apply:

```bash
oc apply -f lesson10-resources.yaml
```

Check:

```bash
oc get pods
```

---
## 7. Inspect Resource Configuration

Check the Deployment:

```bash
oc describe deployment lesson10-web
```

Check a Pod:

```bash
oc describe pod <pod-name>
```

Look for:

```text
Requests:
  cpu: 100m
  memory: 128Mi

Limits:
  cpu: 500m
  memory: 256Mi
```

You can also check:

```bash
oc get deployment lesson10-web -o yaml
```

---
## 8. Why Requests Matter for Scheduling

Imagine a worker node has:

```text
CPU: 4
Memory: 8Gi
```

Your Pod requests:

```text
CPU: 500m
Memory: 512Mi
```

OpenShift checks whether a node has enough available requested resources.

```text
Worker Node
+---------------------------+
| CPU:    4 CPU             |
| Memory: 8Gi               |
|                           |
| Pod Request               |
| CPU:    500m              |
| Memory: 512Mi             |
+---------------------------+
```

The scheduler uses these requests when selecting a suitable node.

---
## 9. CPU Limit

Suppose:

```yaml
limits:
  cpu: 500m
```

If the application tries to use more CPU than its limit, CPU usage can be throttled.

Conceptually:

```text
Application
    |
    v
CPU usage increases
    |
    v
CPU limit reached
    |
    v
CPU throttling
```

Reaching the CPU limit does not normally cause the container to be killed simply because it reached the CPU limit.

---
## 10. Memory Limit

Suppose:

```yaml
limits:
  memory: 256Mi
```

If the container consumes more memory than allowed, it can be terminated with:

```text
OOMKilled
```

`OOM` means:

```text
Out Of Memory
```

Check:

```bash
oc get pods
```

Check the Pod:

```bash
oc describe pod <pod-name>
```

---
## 11. Practice OOMKilled

Created `lesson10-oom.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson10-oom
spec:
  containers:
    - name: memory-test
      image: polinux/stress
      resources:
        requests:
          memory: 32Mi
        limits:
          memory: 64Mi
      command:
        - stress
      args:
        - --vm
        - "1"
        - --vm-bytes
        - "128M"
        - --vm-hang
        - "1"
```

Apply:

```bash
oc apply -f lesson10-oom.yaml
```

Check:

```bash
oc get pod lesson10-oom
```

Describe:

```bash
oc describe pod lesson10-oom
```

If the container restarted, check:

```bash
oc logs lesson10-oom --previous
```

---
## 12. Check Container Termination Reason

Run:

```bash
oc get pod lesson10-oom -o jsonpath="{.status.containerStatuses[0].lastState.terminated.reason}"
```

If the container exceeded its memory limit, the result may be:

```text
OOMKilled
```

This is a common production troubleshooting scenario.

---
## 13. Production Resource Configuration

A typical application might use:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1
    memory: 512Mi
```

Meaning:

```text
Expected:
CPU     → 250m
Memory  → 256Mi

Maximum:
CPU     → 1 CPU
Memory  → 512Mi
```

Actual values should be based on application usage and monitoring rather than arbitrary values.

---
## 14. QoS Classes

OpenShift/Kubernetes assigns Pods a Quality of Service class.

The three common classes are:

```text
Guaranteed
Burstable
BestEffort
```

### Guaranteed

Requests and limits are configured appropriately for all containers.

### Burstable

Requests and/or limits are configured, but they are not all equal.

### BestEffort

No CPU or memory requests/limits are configured.

Check a Pod's QoS class:

```bash
oc get pod <pod-name> -o jsonpath="{.status.qosClass}"
```

Example:

```text
Burstable
```

---
## 15. Check Resource Usage

Depending on Developer Sandbox permissions, try:

```bash
oc adm top pods
```

If permitted, you may see:

```text
NAME              CPU(cores)   MEMORY(bytes)
lesson10-web-xxx  5m           10Mi
lesson10-web-yyy  4m           9Mi
```

If the command is not permitted in the Developer Sandbox, that is okay.

Configured requests and limits can still be inspected using:

```bash
oc describe pod <pod-name>
```

---
## 16. Introduction to HPA

HPA means:

```text
Horizontal Pod Autoscaler
```

HPA automatically changes the number of Pod replicas based on resource usage or supported metrics.

Without HPA:

```text
Deployment
    |
    +---- Pod 1
    +---- Pod 2
```

With HPA:

```text
Low traffic
    |
    v
2 Pods

High traffic
    |
    v
4 Pods

Very high traffic
    |
    v
5 Pods
```

The important concept:

```text
HPA → Changes the number of Pod replicas
```

---
## 17. Create HPA

The Deployment already has a CPU request:

```yaml
resources:
  requests:
    cpu: 100m
```

Create an HPA:

```bash
oc autoscale deployment lesson10-web \
  --min=2 \
  --max=5 \
  --cpu-percent=70
```

Check:

```bash
oc get hpa
```

---
## 18. Understand HPA Configuration

The command:

```bash
oc autoscale deployment lesson10-web \
  --min=2 \
  --max=5 \
  --cpu-percent=70
```

means:

```text
Minimum replicas = 2
Maximum replicas = 5
Target CPU       = 70%
```

Conceptually:

```text
CPU below target
       |
       v
Fewer replicas may be needed

CPU above target
       |
       v
More replicas may be created
```

---
## 19. Check HPA

Run:

```bash
oc get hpa
```

Detailed information:

```bash
oc describe hpa lesson10-web
```

Check Deployment:

```bash
oc get deployment lesson10-web
```

Check Pods:

```bash
oc get pods
```

Watch:

```bash
oc get pods -w
```

---
## 20. Important HPA Requirement

CPU-based HPA generally needs CPU requests configured.

Example:

```yaml
resources:
  requests:
    cpu: 100m
```

HPA calculates CPU utilization relative to the CPU request.

Conceptually:

```text
Current CPU Usage
       |
       v
CPU Request
       |
       v
Utilization %
       |
       v
HPA Decision
       |
       v
Increase / Decrease Replicas
```

---
## 21. HPA Architecture

```text
                 Metrics
                    |
                    v
                   HPA
                    |
          +---------+---------+
          |                   |
          v                   v
      Scale Up             Scale Down
          |                   |
          v                   v
     More Pods             Fewer Pods
```

---
## 22. Real-World Example

Imagine an online shopping application.

Normal traffic:

```text
2 Pods
```

During a sale:

```text
CPU usage increases
        |
        v
HPA detects high utilization
        |
        v
3 Pods
        |
        v
4 Pods
        |
        v
5 Pods
```

After traffic decreases:

```text
Traffic decreases
        |
        v
HPA detects lower utilization
        |
        v
Pods scale down
```

This allows an application to adapt automatically to changing load.

---
## 23. HPA Troubleshooting

Suppose:

```bash
oc get hpa
```

shows:

```text
TARGETS: <unknown>
```

Check:

```bash
oc describe hpa lesson10-web
```

Check the Deployment:

```bash
oc get deployment lesson10-web -o yaml
```

Verify CPU requests:

```yaml
resources:
  requests:
    cpu: 100m
```

Check metrics if permitted:

```bash
oc adm top pods
```

If the Developer Sandbox does not allow this command, it may be a platform permission limitation.

---
## 24. Useful Troubleshooting Commands

### Resource Configuration

```bash
oc get deployment lesson10-web -o yaml
```

```bash
oc describe pod <pod-name>
```

```bash
oc get pod <pod-name> -o jsonpath="{.status.qosClass}"
```

### HPA

```bash
oc get hpa
```

```bash
oc describe hpa lesson10-web
```

### Pods

```bash
oc get pods
```

```bash
oc get pods -w
```

### Events

```bash
oc get events --sort-by=.lastTimestamp
```

### Logs

```bash
oc logs <pod-name>
```

```bash
oc logs <pod-name> --previous
```

---
## 25. Interview Questions

1. What is a resource request?
2. What is a resource limit?
3. What is the difference between requests and limits?
4. How does the scheduler use resource requests?
5. What happens when a container exceeds its CPU limit?
6. What happens when a container exceeds its memory limit?
7. What is OOMKilled?
8. What is QoS?
9. What are Guaranteed, Burstable and BestEffort?
10. What is HPA?
11. What does HPA scale?
12. What is the difference between horizontal and vertical scaling?
13. Why are CPU requests important for CPU-based HPA?
14. How would you troubleshoot HPA showing `<unknown>`?
15. How would you troubleshoot an OOMKilled Pod?

---
## 🧹 Cleanup

Remove HPA:

```bash
oc delete hpa lesson10-web
```

Remove the application:

```bash
oc delete -f lesson10-resources.yaml
```

Remove the OOM test:

```bash
oc delete -f lesson10-oom.yaml
```

---
## ✅ Lesson 10 Completion Checklist

- [x] Understand CPU and Memory
- [x] Understand Resource Requests
- [x] Understand Resource Limits
- [x] Understand Requests vs Limits
- [x] Configure CPU Requests
- [x] Configure CPU Limits
- [x] Configure Memory Requests
- [x] Configure Memory Limits
- [x] Understand CPU throttling
- [x] Understand OOMKilled
- [x] Troubleshoot an OOMKilled Pod
- [x] Understand QoS basics
- [x] Understand HPA
- [x] Create an HPA
- [x] Check HPA status
- [x] Understand minimum and maximum replicas
- [x] Understand CPU target utilization
- [x] Understand basic HPA troubleshooting

---
## 🧠 Final Memory Trick

```text
Request → What the application expects/needs
Limit   → Maximum resource it can consume

CPU     → Can be throttled
Memory  → Can cause OOMKilled

HPA     → Automatically changes Pod count
```

---
Final concept:

```text
Application
     |
     v
Resource Requests + Limits
     |
     v
OpenShift Scheduler
     |
     v
Running Pods
     |
     v
HPA
     |
     +---- Scale Up
     |
     +---- Scale Down
```
