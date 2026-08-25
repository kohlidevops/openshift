# OpenShift Learning – Lesson 16: Logging & Monitoring

## 🚀 Lesson 16: OpenShift Logging & Monitoring

> **Environment Note:** I am using an OpenShift Developer Sandbox with a restricted user and Windows Command Prompt (CMD). Some cluster-level monitoring commands such as `oc top` are not available in my environment. This lesson focuses on logging, Events, Pod restart counts, resource requests/limits, and monitoring concepts that I can practice with my current permissions.

---

## 🎯 Learning Objectives

By the end of this lesson, I should understand:

- Application logs vs container logs
- `oc logs`
- `oc logs -f`
- `oc logs --previous`
- Log timestamps
- stdout vs stderr
- Log filtering using Windows CMD
- Pod restart counts
- OpenShift Events
- Monitoring concepts
- CPU and memory metrics
- Resource requests and limits
- Four Golden Signals
- Troubleshooting using logs + Events + resources
- Why `oc top` may not be available in my environment

---

# 1. Logging vs Monitoring

### Logging

Logs answer:

```text
What happened?
```

Examples:

```text
Application started
Database connection failed
Request received
Permission denied
```

### Monitoring

Metrics answer:

```text
How much?
How often?
How many?
```

Examples:

```text
CPU
Memory
Request rate
Error rate
Latency
```

### Easy Memory

```text
Logs    → What happened?

Metrics → How much is happening?
```

---

# 2. Create a Logging Application

Create a simple application that continuously generates logs:

```cmd
oc create deployment lesson16-logger --image=busybox:1.36 -- sh -c "while true; do echo Application-Running; sleep 5; done"
```

Check:

```cmd
oc get pods
```

Wait until the Pod shows:

```text
1/1 Running
```

---

# 3. View Container Logs

Get the Pod name:

```cmd
oc get pods
```

Then:

```cmd
oc logs <pod-name>
```

Expected output:

```text
Application-Running
Application-Running
Application-Running
```

`oc logs` displays the logs produced by the container.

---

# 4. Follow Logs in Real Time

Run:

```cmd
oc logs -f <pod-name>
```

You should see a new message every few seconds.

Stop following:

```text
Ctrl + C
```

### Remember

```text
oc logs
    ↓
Show logs

oc logs -f
    ↓
Follow logs continuously
```

---

# 5. Add Timestamps

Run:

```cmd
oc logs <pod-name> --timestamps
```

Example:

```text
2026-08-25T05:30:01Z Application-Running
```

Timestamps help determine exactly when an application event occurred.

---

# 6. stdout and stderr

Create an application that generates both normal and error messages:

```cmd
oc run lesson16-logtest --image=busybox:1.36 -- sh -c "while true; do echo INFO Application-running; echo ERROR Example-error >&2; sleep 5; done"
```

Check:

```cmd
oc get pods
```

Read the logs:

```cmd
oc logs lesson16-logtest
```

You should see messages similar to:

```text
INFO Application-running
ERROR Example-error
```

### Understand

```text
stdout
 ↓
Normal application output

stderr
 ↓
Error/warning output
```

OpenShift captures the container's stdout/stderr as container logs.

---

# 7. Filter Logs in Windows CMD

Because I am using **Windows Command Prompt**, use `findstr` instead of PowerShell's `Select-String`.

Search for `ERROR`:

```cmd
oc logs lesson16-logtest | findstr "ERROR"
```

Search for `INFO`:

```cmd
oc logs lesson16-logtest | findstr "INFO"
```

Expected:

```text
ERROR Example-error
```

This is useful when applications produce large amounts of logs.

---

# 8. Create a Crashing Application

Create:

```cmd
oc create deployment lesson16-crash --image=busybox:1.36 -- sh -c "echo Application-started; echo Database-connection-failed >&2; exit 1"
```

Check:

```cmd
oc get pods
```

The Pod should eventually show:

```text
CrashLoopBackOff
```

---

# 9. Troubleshoot CrashLoopBackOff

Get the Pod name:

```cmd
oc get pods
```

Check current logs:

```cmd
oc logs <pod-name>
```

Check previous container logs:

```cmd
oc logs <pod-name> --previous
```

Describe the Pod:

```cmd
oc describe pod <pod-name>
```

Check Events:

```cmd
oc get events --sort-by=.lastTimestamp
```

### Troubleshooting flow

```text
CrashLoopBackOff
       ↓
oc logs
       ↓
Application error
       ↓
oc logs --previous
       ↓
oc describe pod
       ↓
Check Events
       ↓
Find root cause
```

---

# 10. Why `--previous` Is Important

When a container crashes, the current container may not contain the useful logs from the previous failed instance.

Use:

```cmd
oc logs <pod-name> --previous
```

This is especially useful for:

```text
CrashLoopBackOff
Container restarts
Startup failures
Application crashes
```

---

# 11. Check Pod Restart Count

Run:

```cmd
oc get pods
```

Look at:

```text
RESTARTS
```

Example:

```text
NAME                    READY   STATUS             RESTARTS
lesson16-crash-xxxxx   0/1     CrashLoopBackOff   5
```

A high restart count is a warning sign.

Investigate using:

```cmd
oc logs <pod-name> --previous
```

and:

```cmd
oc describe pod <pod-name>
```

---

# 12. OpenShift Events

Run:

```cmd
oc get events --sort-by=.lastTimestamp
```

Events may show:

```text
Scheduled
Pulling
Pulled
Created
Started
Failed
BackOff
```

### Remember

```text
Events = Timeline of what happened to the resource
```

---

# 13. `oc describe` vs `oc logs`

This is an important distinction.

### `oc logs`

```text
Application information
Application errors
Application output
Runtime information
```

Command:

```cmd
oc logs <pod-name>
```

### `oc describe`

```text
Pod configuration
Container state
Restart count
Events
Volumes
Scheduling information
```

Command:

```cmd
oc describe pod <pod-name>
```

### Easy Memory

```text
logs
 ↓
What did the application say?

describe
 ↓
What does OpenShift know about the Pod?
```

---

# 14. Deployment Logs

You can also retrieve logs through a Deployment:

```cmd
oc logs deployment/lesson16-logger
```

This is useful when working with applications managed by Deployments.

---

# 15. Multiple Containers

A Pod can contain multiple containers.

Check container names:

```cmd
oc get pod <pod-name> -o jsonpath="{.spec.containers[*].name}"
```

If there are multiple containers:

```cmd
oc logs <pod-name> -c <container-name>
```

This allows you to troubleshoot a specific container.

---

# 16. Monitoring Concepts

Monitoring answers questions such as:

```text
How much CPU is being used?
How much memory is being used?
How many requests are coming in?
How many errors are happening?
How long are requests taking?
```

Common infrastructure metrics:

```text
CPU
Memory
Network
Storage
Pod count
Container restarts
```

Common application metrics:

```text
Request rate
Response time
Error rate
Database connections
Active users
```

---

# 17. Four Golden Signals

A useful monitoring concept is the **Four Golden Signals**:

```text
Latency
Traffic
Errors
Saturation
```

### Latency

How long does a request take?

### Traffic

How much demand does the application receive?

### Errors

How many requests fail?

### Saturation

How much of the available resources are being consumed?

Example:

```text
CPU       → 90%
Memory    → 85%
Requests  → 500/sec
Errors    → 10%
Latency   → 2 seconds
```

---

# 18. CPU Metrics

CPU is commonly represented using:

```text
m
```

Examples:

```text
100m  = 0.1 CPU
500m  = 0.5 CPU
1000m = 1 CPU
```

Remember:

```text
1000m = 1 CPU core
```

---

# 19. Memory Metrics

Memory is commonly represented using:

```text
Mi
Gi
```

Examples:

```text
128Mi
512Mi
1Gi
```

Memory monitoring can help identify:

```text
Memory pressure
Memory leaks
OOMKilled
Incorrect memory limits
```

---

# 20. `oc top` in My Environment

I tested:

```cmd
oc top pods
```

but my OpenShift CLI returned:

```text
error: unknown command "top" for "oc"
```

Therefore, `oc top` is **not available in my current environment**.

I will not install or modify cluster monitoring components just to make this command work.

For this lesson:

```text
Live metrics
    ↓
Learn conceptually

Logging
Events
Restart counts
Requests/Limits
    ↓
Practice hands-on
```

---

# 21. Resource Requests and Limits

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson16-resources
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo Resource-test; sleep 3600"]
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 512Mi
```

Save as:

```text
lesson16-resources.yaml
```

Apply:

```cmd
oc apply -f lesson16-resources.yaml
```

Check:

```cmd
oc get pod lesson16-resources
```

---

# 22. Inspect Resource Configuration

Run:

```cmd
oc describe pod lesson16-resources
```

Look for:

```text
Requests
Limits
```

Also run:

```cmd
oc get pod lesson16-resources -o yaml
```

Find:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

---

# 23. Resource Requests

Requests tell OpenShift:

> How much resource does this container need?

Example:

```text
CPU request    = 100m
Memory request = 128Mi
```

Think:

```text
Request
   ↓
Expected/required resource
```

---

# 24. Resource Limits

Limits tell OpenShift:

> What is the maximum resource this container can use?

Example:

```text
CPU limit    = 500m
Memory limit = 512Mi
```

Think:

```text
Limit
   ↓
Maximum allowed resource
```

---

# 25. Logs + Events + Resources

A strong DevOps troubleshooting approach combines multiple sources.

```text
Application Problem
       |
       +---- Pod Status
       |
       +---- Restart Count
       |
       +---- Logs
       |
       +---- Previous Logs
       |
       +---- Events
       |
       +---- Resource Configuration
       |
       v
    Root Cause
```

Don't rely only on:

```cmd
oc get pods
```

---

# 26. Application Health vs Resource Health

An application can have:

```text
CPU = Low
Memory = Low
```

and still be broken.

Example:

```text
CPU = 5%
Memory = 50Mi
Application = Database connection failed
```

Therefore:

```text
Low resource usage
        ≠
Healthy application
```

We need:

```text
Application Health
        +
Resource Health
```

---


# 32. Important Commands

## Logs

```cmd
oc logs <pod-name>
```

```cmd
oc logs -f <pod-name>
```

```cmd
oc logs <pod-name> --previous
```

```cmd
oc logs <pod-name> --timestamps
```

```cmd
oc logs <pod-name> -c <container-name>
```

```cmd
oc logs deployment/<deployment-name>
```

## Log Filtering – Windows CMD

```cmd
oc logs <pod-name> | findstr "ERROR"
```

```cmd
oc logs <pod-name> | findstr "INFO"
```

## Pod Troubleshooting

```cmd
oc get pods
```

```cmd
oc describe pod <pod-name>
```

```cmd
oc get pod <pod-name> -o yaml
```

## Events

```cmd
oc get events --sort-by=.lastTimestamp
```

## Resource Configuration

```cmd
oc describe pod <pod-name>
```

```cmd
oc get pod <pod-name> -o yaml
```

---

# 🧠 Final Memory Trick

```text
Logs
 ↓
What happened?

Events
 ↓
What did OpenShift do?

Restarts
 ↓
Is the container repeatedly failing?

Requests
 ↓
What resources does the application need?

Limits
 ↓
What is the maximum allowed?

Metrics
 ↓
How much is being consumed?
```

Complete troubleshooting flow:

```text
Pod Status
    ↓
Restart Count
    ↓
Logs
    ↓
Previous Logs
    ↓
Events
    ↓
Resource Configuration
    ↓
Metrics (when available)
    ↓
Root Cause
```

---

# ✅ Lesson 16 Completion Checklist

- [ ] Understand logging vs monitoring
- [ ] Understand application/container logs
- [ ] Use `oc logs`
- [ ] Use `oc logs -f`
- [ ] Use `oc logs --previous`
- [ ] Use `oc logs --timestamps`
- [ ] Get logs from a specific container
- [ ] Get logs from a Deployment
- [ ] Understand stdout/stderr
- [ ] Filter logs using `findstr`
- [ ] Understand OpenShift Events
- [ ] Check Pod restart counts
- [ ] Understand CPU metrics
- [ ] Understand memory metrics
- [ ] Understand monitoring concepts
- [ ] Understand the Four Golden Signals
- [ ] Understand resource requests
- [ ] Understand resource limits
- [ ] Inspect resource configuration
- [ ] Combine logs + Events for troubleshooting
- [ ] Troubleshoot a crashing application
- [ ] Understand why `oc top` is unavailable in my environment
- [ ] Use Windows CMD-compatible OpenShift commands

---

# 🏁 Lesson 16 Goal


```text
Logs
+
Events
+
Restart Count
+
Resource Configuration
+
Monitoring Concepts
```

instead of relying only on:

```text
oc get pods
```

The key DevOps troubleshooting mindset is:

```text
Don't just ask:

"Is the Pod Running?"

Ask:

"What is the application doing?"
"What happened?"
"Why did it restart?"
"What did OpenShift report?"
"What resources does it require?"
"What could be the root cause?"
```

**Lesson 16 Topic: OpenShift Logging & Monitoring – Application Logs, Container Logs, Events, Resource Configuration & Troubleshooting**
