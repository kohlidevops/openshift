# OpenShift Learning – Lesson 8: Health Checks & Self-Healing

## 🎯 Objectives

In this lesson, I learned and practiced:

- Health Probes
- Liveness Probe
- Readiness Probe
- Startup Probe
- Difference between Liveness, Readiness and Startup
- How OpenShift checks application health
- Container restart behavior
- Service traffic and readiness
- Probe troubleshooting
- Self-healing behavior

---
## 1. What Are Health Probes?

A container can be `Running` but the application inside it may not actually be working.

Health probes allow OpenShift to check the application.

```text
OpenShift
    |
    +---- Is the application alive?
    |
    +---- Is the application ready?
    |
    +---- Has the application finished starting?
```

There are three important probes:

| Probe | Question | Result |
|---|---|---|
| Liveness | Is the application alive? | Restart container if it fails |
| Readiness | Can the application receive traffic? | Remove Pod from Service endpoints |
| Startup | Has the application finished starting? | Protect slow-starting applications |

Easy way to remember:

```text
Startup   → Start
Readiness → Traffic
Liveness  → Restart
```

---
## 2. Create Basic Application

Created `lesson8-basic.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson8-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson8-web
  template:
    metadata:
      labels:
        app: lesson8-web
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: lesson8-web
spec:
  selector:
    app: lesson8-web
  ports:
    - port: 80
      targetPort: 8080
```

Apply:

```bash
oc apply -f lesson8-basic.yaml
```

Verify:

```bash
oc get pods
oc get svc lesson8-web
```

---
## 3. Readiness Probe

A Readiness Probe answers:

> Can this application receive traffic?

Created `lesson8-readiness.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson8-readiness
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson8-readiness
  template:
    metadata:
      labels:
        app: lesson8-readiness
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080

          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3

---
apiVersion: v1
kind: Service
metadata:
  name: lesson8-readiness
spec:
  selector:
    app: lesson8-readiness
  ports:
    - port: 80
      targetPort: 8080
```

Apply:

```bash
oc apply -f lesson8-readiness.yaml
```

Check:

```bash
oc get pods
```

Expected:

```text
READY   STATUS
1/1     Running
1/1     Running
```

Check the Pod details:

```bash
oc describe pod <pod-name>
```

---
## 4. How Readiness Works

If the readiness probe succeeds:

```text
Pod
 |
 +--> Ready
       |
       v
    Service
       |
       v
    Traffic
```

If the readiness probe fails:

```text
Pod
 |
 +--> Not Ready
       |
       X
    No Service traffic
```

A failed readiness probe does **not normally restart the container**.

Check Service endpoints:

```bash
oc get endpoints lesson8-readiness
```

Check EndpointSlices:

```bash
oc get endpointslice -l kubernetes.io/service-name=lesson8-readiness
```

---
## 5. Liveness Probe

A Liveness Probe answers:

> Is this application still alive?

Created `lesson8-liveness.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson8-liveness
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lesson8-liveness
  template:
    metadata:
      labels:
        app: lesson8-liveness
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080

          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
```

Apply:

```bash
oc apply -f lesson8-liveness.yaml
```

Check:

```bash
oc get pods
```

---
## 6. Test Liveness Probe Failure

Intentionally change:

```yaml
path: /
```

to:

```yaml
path: /does-not-exist
```

Apply:

```bash
oc apply -f lesson8-liveness.yaml
```

Watch the Pod:

```bash
oc get pods -w
```

Check restart count:

```bash
oc get pods
```

The `RESTARTS` value should increase when the liveness probe continuously fails.

---
## 7. Troubleshoot Liveness Failure

Describe the Pod:

```bash
oc describe pod <pod-name>
```

Check the Events section for probe failure messages.

Check logs:

```bash
oc logs <pod-name>
```

If the container has restarted, check the previous container logs:

```bash
oc logs <pod-name> --previous
```

Fix the probe by changing:

```yaml
path: /does-not-exist
```

back to:

```yaml
path: /
```

Apply:

```bash
oc apply -f lesson8-liveness.yaml
```

---
## 8. Startup Probe

A Startup Probe is useful for applications that take a long time to start.

Example:

```text
Application starts
      |
      | 60 seconds
      |
      v
Application ready
```

Without a Startup Probe, a liveness probe might start too early and restart the application before it has finished starting.

Created `lesson8-startup.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson8-startup
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lesson8-startup
  template:
    metadata:
      labels:
        app: lesson8-startup
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080

          startupProbe:
            httpGet:
              path: /
              port: 8080
            failureThreshold: 30
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: 8080
            periodSeconds: 10

          readinessProbe:
            httpGet:
              path: /
              port: 8080
            periodSeconds: 5
```

Apply:

```bash
oc apply -f lesson8-startup.yaml
```

Check:

```bash
oc get pods
```

---
## 9. Understand the Three Probes Together

```text
             Container Starts
                    |
                    v
              Startup Probe
                    |
             Has it started?
                    |
                    v
             Readiness Probe
                    |
          Can it receive traffic?
              /           \
            No             Yes
            |               |
            v               v
        Not Ready         Service
                              |
                              v
                           Traffic

             Liveness Probe
                    |
            Is it still alive?
               /          \
             Yes           No
              |             |
              v             v
           Continue       Restart
```

---
## 10. Probe Types

### HTTP Probe

```yaml
httpGet:
  path: /
  port: 8080
```

### TCP Probe

```yaml
tcpSocket:
  port: 8080
```

### Command Probe

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

For this lesson, the main focus was HTTP probes.

---
## 11. Real-World Example

An API might expose:

```text
/api/health
```

A readiness probe can use:

```yaml
readinessProbe:
  httpGet:
    path: /api/health
    port: 8080
```

If the application temporarily cannot serve traffic:

```text
Application
     |
     X
Health check failed
     |
     v
Pod becomes NotReady
     |
     v
Service stops sending traffic
```

The application does not necessarily need to be restarted.

---
## 12. Production Troubleshooting Scenario

Suppose:

```bash
oc get pods
```

shows:

```text
NAME       READY   STATUS    RESTARTS
app-xxxxx  0/1     Running   0
```

The Pod is `Running`, but it is not Ready.

First check:

```bash
oc describe pod app-xxxxx
```

Look for:

```text
Readiness probe failed
```

Then check:

```bash
oc logs app-xxxxx
```

Check Service endpoints:

```bash
oc get endpoints <service-name>
```

Possible situation:

```text
Pod       = Running
Pod       = NotReady
Service   = No endpoint
```

This is an important real-world troubleshooting scenario.

---
## 13. Useful Troubleshooting Commands

Check Pods:

```bash
oc get pods
```

Watch Pods:

```bash
oc get pods -w
```

Describe Pod:

```bash
oc describe pod <pod-name>
```

Check logs:

```bash
oc logs <pod-name>
```

Check previous container logs:

```bash
oc logs <pod-name> --previous
```

Check Services:

```bash
oc get svc
```

Check Endpoints:

```bash
oc get endpoints <service-name>
```

Check EndpointSlices:

```bash
oc get endpointslice
```

Check Deployment:

```bash
oc describe deployment <deployment-name>
```

Check Events:

```bash
oc get events --sort-by=.lastTimestamp
```

---
## 14. Key Learnings

### Liveness

```text
Application unhealthy
        ↓
Liveness fails
        ↓
Container restarted
```

### Readiness

```text
Application cannot serve traffic
        ↓
Readiness fails
        ↓
Pod becomes NotReady
        ↓
Service stops sending traffic
```

### Startup

```text
Application takes time to start
        ↓
Startup Probe protects startup period
        ↓
Application becomes ready
        ↓
Normal liveness/readiness checks continue
```

## 🧠 Easy Memory Trick

```text
Startup   → Start
Readiness → Traffic
Liveness  → Restart
```

## 🎤 Interview Questions

1. What is a Liveness Probe?
2. What is a Readiness Probe?
3. What is a Startup Probe?
4. What is the difference between Liveness and Readiness?
5. What happens when a Readiness Probe fails?
6. Does a Readiness Probe failure restart the container?
7. What happens when a Liveness Probe continuously fails?
8. Why do we need a Startup Probe?
9. What types of health probes are available?
10. How would you troubleshoot a failed Readiness Probe?
11. Why can a Pod show `Running` but `0/1 Ready`?
12. How does Readiness affect a Service?
13. How do you check probe failure events?
14. How do you check logs from a previous container after a restart?

## 🧹 Cleanup

```bash
oc delete -f lesson8-basic.yaml
oc delete -f lesson8-readiness.yaml
oc delete -f lesson8-liveness.yaml
oc delete -f lesson8-startup.yaml
```

---
## ✅ Lesson 8 Completion Checklist

- [x] Understand Health Probes
- [x] Understand Liveness Probe
- [x] Understand Readiness Probe
- [x] Understand Startup Probe
- [x] Create HTTP Readiness Probe
- [x] Create HTTP Liveness Probe
- [x] Create Startup Probe
- [x] Test failed Readiness Probe
- [x] Test failed Liveness Probe
- [x] Observe Pod restart behavior
- [x] Understand Service and Readiness
- [x] Troubleshoot probe failures
- [x] Check previous container logs
- [x] Understand self-healing


**Key concept:**

`Startup → Start | Readiness → Traffic | Liveness → Restart`
