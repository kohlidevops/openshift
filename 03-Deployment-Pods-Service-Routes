# OpenShift Learning – Lesson 3

## Deployments, Pods, Services, Routes & Troubleshooting

In this lesson, I learned how OpenShift manages application workloads using **Deployments, ReplicaSets, Pods, Services and Routes**.

I performed hands-on practice using the **Red Hat OpenShift Developer Sandbox**.

---

## 🎯 Learning Objectives

- Understand Deployments
- Understand ReplicaSets
- Understand Pods
- Understand Labels and Selectors
- Create and use Services
- Understand ClusterIP
- Understand Service Endpoints and EndpointSlices
- Expose applications using OpenShift Routes
- Scale applications
- Perform rolling updates
- Perform rollbacks
- Troubleshoot `CrashLoopBackOff`
- Troubleshoot Service selector issues
- Understand OpenShift container security basics

---

# 1. Application Architecture

```text
                         OpenShift
                             |
                             v
                        Deployment
                             |
                             v
                        ReplicaSet
                             |
                 +-----------+-----------+
                 |                       |
                 v                       v
               Pod 1                   Pod 2
                 |                       |
                 +-----------+-----------+
                             |
                             v
                          Service
                             |
                             v
                           Route
                             |
                             v
                         End User
```

---

# 2. Deployment

A Deployment manages the desired state of an application.

### `lesson3-app1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson3-web
  labels:
    app: lesson3-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson3-web
  template:
    metadata:
      labels:
        app: lesson3-web
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080
```

Apply the Deployment:

```bash
oc apply -f lesson3-app1.yaml
```

Check the Deployment:

```bash
oc get deployments
```

Detailed information:

```bash
oc describe deployment lesson3-web
```

---

# 3. ReplicaSet

A Deployment creates and manages a ReplicaSet.

The ReplicaSet ensures that the desired number of Pods are running.

```text
Deployment
     |
     v
ReplicaSet
     |
     +-- Pod 1
     |
     +-- Pod 2
```

Check ReplicaSets:

```bash
oc get replicasets
```

---

# 4. Pods

A Pod is the basic execution unit where containers run.

Check Pods:

```bash
oc get pods
```

Filter Pods using a label:

```bash
oc get pods -l app=lesson3-web
```

Display Pod labels:

```bash
oc get pods --show-labels
```

Inspect a Pod:

```bash
oc describe pod <pod-name>
```

View Pod logs:

```bash
oc logs <pod-name>
```

---

# 5. Labels

Labels are key/value pairs attached to OpenShift/Kubernetes resources.

Our Pods use:

```yaml
labels:
  app: lesson3-web
```

Check labels:

```bash
oc get pods --show-labels
```

Example:

```text
app=lesson3-web
```

Labels allow other resources, such as Services, to identify the correct Pods.

---

# 6. Selectors

A selector identifies resources based on matching labels.

Our Service uses:

```yaml
selector:
  app: lesson3-web
```

Our Pods have:

```yaml
labels:
  app: lesson3-web
```

Therefore:

```text
Service
   |
   | selector: app=lesson3-web
   |
   v
Pods
   |
   | label: app=lesson3-web
```

If the selector and labels do not match, the Service cannot find the Pods.

---

# 7. Service

A Service provides a stable network endpoint for accessing Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson3-web
  labels:
    app: lesson3-web
spec:
  type: ClusterIP
  selector:
    app: lesson3-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

Check Services:

```bash
oc get svc
```

Inspect the Service:

```bash
oc describe svc lesson3-web
```

---

# 8. Service Port vs TargetPort

Our Service uses:

```yaml
port: 80
targetPort: 8080
```

This means:

```text
Client
  |
  | Service Port 80
  v
Service
  |
  | Target Port 8080
  v
NGINX Container :8080
```

The Service provides port `80`, while the NGINX container listens on `8080`.

---

# 9. Service Endpoints

A Service uses its selector to find matching Pods.

Check endpoints:

```bash
oc get endpoints lesson3-web
```

On newer OpenShift/Kubernetes versions, the `Endpoints` API may show a deprecation warning.

The newer API is EndpointSlice:

```bash
oc get endpointslice
```

For our Service:

```bash
oc get endpointslice -l kubernetes.io/service-name=lesson3-web
```

Conceptually:

```text
Service
   |
   +-- Pod IP 1
   |
   +-- Pod IP 2
```

If there are no matching Pods:

```text
ENDPOINTS
<none>
```

the Service cannot route traffic to the application.

---

# 10. Test Service Connectivity

A temporary curl Pod was used to test Service connectivity:

```bash
oc run curl-test \
  --image=curlimages/curl:8.10.1 \
  --rm -it \
  --restart=Never \
  -- curl http://lesson3-web
```

Expected result:

```text
NGINX HTML response
```

Architecture:

```text
curl-test Pod
      |
      v
lesson3-web Service
      |
      v
NGINX Pods
```

---

# 11. OpenShift Route

OpenShift Routes provide external access to Services.

Create a Route:

```bash
oc expose service lesson3-web
```

Check Routes:

```bash
oc get route
```

Get the Route hostname:

```bash
oc get route lesson3-web -o jsonpath="{.spec.host}"
```

Application architecture:

```text
Internet
   |
   v
OpenShift Route
   |
   v
Service
   |
   +-- Pod 1
   |
   +-- Pod 2
```

The application was successfully exposed through an OpenShift Route.

---

# 12. Scaling

The Deployment initially used:

```yaml
replicas: 2
```

Scale to 4 replicas:

```bash
oc scale deployment lesson3-web --replicas=4
```

Check Pods:

```bash
oc get pods -l app=lesson3-web
```

Scale back to 2:

```bash
oc scale deployment lesson3-web --replicas=2
```

Verify:

```bash
oc get deployment lesson3-web
```

Scaling:

```text
2 Pods
   |
   v
Scale
   |
   v
4 Pods
   |
   v
Scale Down
   |
   v
2 Pods
```

---

# 13. Rolling Update

Check the current image:

```bash
oc get deployment lesson3-web \
  -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Initial image:

```text
nginxinc/nginx-unprivileged:1.27
```

Update to the newer image:

```bash
oc set image deployment/lesson3-web \
  nginx=nginxinc/nginx-unprivileged:1.28
```

Check rollout status:

```bash
oc rollout status deployment/lesson3-web
```

Expected:

```text
deployment "lesson3-web" successfully rolled out
```

Rolling update:

```text
Old Version
   |
   +-- Pod 1
   |
   +-- Pod 2
        |
        v
Rolling Update
        |
        v
New Version
   |
   +-- Pod 1
   |
   +-- Pod 2
```

---

# 14. Rollout History

View Deployment revisions:

```bash
oc rollout history deployment/lesson3-web
```

Inspect a specific revision:

```bash
oc rollout history deployment/lesson3-web --revision=2
```

---

# 15. Rollback

Rollback to the previous revision:

```bash
oc rollout undo deployment/lesson3-web
```

Check rollout status:

```bash
oc rollout status deployment/lesson3-web
```

Verify the image:

```bash
oc get deployment lesson3-web \
  -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Rollback architecture:

```text
Version 1
    |
    v
Rolling Update
    |
    v
Version 2
    |
    v
Rollback
    |
    v
Version 1
```

---

# 16. Troubleshooting Scenario – CrashLoopBackOff

During this lesson, the first NGINX Deployment resulted in:

```text
CrashLoopBackOff
```

The Pod Events showed:

```text
Successfully pulled image
Created container
Started container
Back-off restarting failed container
```

The image was successfully pulled and the container started.

The application logs showed:

```text
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

Therefore, the issue was **not an image pull problem**.

The application was starting but could not write to the expected filesystem location under OpenShift's restricted security model.

---

# 17. CrashLoopBackOff Troubleshooting Process

Commands used:

```bash
oc get pods
```

```bash
oc describe pod <pod-name>
```

```bash
oc logs <pod-name>
```

For the previous container instance:

```bash
oc logs <pod-name> --previous
```

Troubleshooting flow:

```text
CrashLoopBackOff
       |
       v
oc get pods
       |
       v
oc describe pod
       |
       v
Check Events
       |
       v
oc logs
       |
       v
Identify Root Cause
       |
       v
Fix Application
       |
       v
Verify Pod
```

---

# 18. OpenShift Security and Container Images

This incident introduced an important OpenShift concept.

A container image that works in a normal Docker environment may not automatically work in OpenShift.

OpenShift applies additional security controls and commonly runs application containers using non-root/random user IDs.

Therefore, applications that assume:

```text
Run as root
      +
Write to protected filesystem locations
```

may fail.

For the working Lesson 3 application, an unprivileged NGINX image was used:

```text
nginxinc/nginx-unprivileged:1.27
```

Detailed Security Context Constraints (SCC) will be covered in a later lesson.

---

# 19. Troubleshooting Scenario – Service With No Endpoints

A Service selector was deliberately changed from:

```text
app=lesson3-web
```

to:

```text
app=wrong-app
```

The Service then showed:

```text
ENDPOINTS
<none>
```

---

# 20. Diagnose Service Selector Mismatch

Check the Service:

```bash
oc get svc lesson3-web -o yaml
```

Check Pod labels:

```bash
oc get pods --show-labels
```

The mismatch was:

```text
Service selector:
app=wrong-app

Pod label:
app=lesson3-web
```

Therefore:

```text
Service
   |
   | app=wrong-app
   X
   |
Pods
   |
   | app=lesson3-web
```

No Pods matched the Service selector.

---

# 21. Fix Service Selector

The selector was corrected to:

```yaml
selector:
  app: lesson3-web
```

Then verified:

```bash
oc get endpoints lesson3-web
```

The Pod endpoints appeared again.

The newer EndpointSlice API can also be checked:

```bash
oc get endpointslice \
  -l kubernetes.io/service-name=lesson3-web
```

---

# 22. Service Troubleshooting Flow

```text
Application not reachable
          |
          v
Check Service
          |
          v
Check Service selector
          |
          v
Check Pod labels
          |
          v
Check EndpointSlice
          |
          v
No endpoints?
          |
          v
Check selector/label mismatch
          |
          v
Fix selector
          |
          v
Endpoints appear
          |
          v
Test application
```

---

# 23. Important Commands Learned

### Deployment

```bash
oc apply -f lesson3-app1.yaml
oc get deployments
oc describe deployment lesson3-web
```

### Pods

```bash
oc get pods
oc get pods -l app=lesson3-web
oc get pods --show-labels
oc describe pod <pod-name>
oc logs <pod-name>
```

### ReplicaSets

```bash
oc get replicasets
```

### Services

```bash
oc get svc
oc describe svc lesson3-web
```

### Endpoints

```bash
oc get endpoints lesson3-web
```

### EndpointSlices

```bash
oc get endpointslice
```

### Routes

```bash
oc get route
oc expose service lesson3-web
```

### Scaling

```bash
oc scale deployment lesson3-web --replicas=4
```

### Rollout

```bash
oc rollout status deployment/lesson3-web
```

### Rollout History

```bash
oc rollout history deployment/lesson3-web
```

### Rollback

```bash
oc rollout undo deployment/lesson3-web
```

---

# 24. Deployment vs Pod vs Service

| Resource | Responsibility |
|---|---|
| **Pod** | Runs application container(s) |
| **ReplicaSet** | Maintains the desired number of Pods |
| **Deployment** | Manages versions, replicas, rolling updates and rollbacks |
| **Service** | Provides stable network access to matching Pods |
| **Route** | Exposes an OpenShift Service externally |

Easy way to remember:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
    |
    v
Service
    |
    v
Route
```

---

# 25. Hands-on Checklist

- [x] Created Deployment
- [x] Created ReplicaSet through Deployment
- [x] Created and verified Pods
- [x] Inspected Pods
- [x] Viewed Pod logs
- [x] Learned Labels
- [x] Learned Selectors
- [x] Created ClusterIP Service
- [x] Verified Service
- [x] Verified Endpoints
- [x] Tested Service connectivity
- [x] Created OpenShift Route
- [x] Accessed application externally
- [x] Scaled application `2 → 4 → 2`
- [x] Performed rolling update
- [x] Viewed rollout history
- [x] Performed rollback
- [x] Troubleshot `CrashLoopBackOff`
- [x] Investigated container permission issue
- [x] Learned OpenShift security implications
- [x] Deliberately broke Service selector
- [x] Diagnosed `<none>` endpoints
- [x] Fixed Service selector
- [x] Verified endpoints after fixing

---

# 🧠 Key Takeaways

### Deployment

> Manages the desired state, replicas, rolling updates and rollbacks of an application.

### ReplicaSet

> Ensures the desired number of Pods are running.

### Pod

> Runs the application container(s).

### Service

> Provides stable networking to a group of Pods selected by labels.

### Route

> Provides external access to an OpenShift Service.

### Labels

> Identify and categorize resources.

### Selectors

> Select resources based on matching labels.

### Endpoints

> Represent the Pods currently selected by a Service.

### Rolling Update

> Gradually replaces old application Pods with new ones.

### Rollback

> Returns the Deployment to a previous revision.

### CrashLoopBackOff

> Indicates that a container repeatedly starts and then fails/exits.

### Important Troubleshooting Rule

> **When a Pod is in `CrashLoopBackOff`, check `oc describe pod` and `oc logs` before making changes.**

---

# 🎤 Interview Questions

### 1. What is a Deployment?

A Deployment manages application Pods and provides declarative updates, replica management, rolling updates and rollbacks.

### 2. What is a Pod?

A Pod is the smallest deployable unit in Kubernetes/OpenShift and contains one or more containers.

### 3. What is a ReplicaSet?

A ReplicaSet ensures that the desired number of Pod replicas are running.

### 4. Deployment vs ReplicaSet?

A Deployment manages application lifecycle and versions, while the ReplicaSet maintains the desired number of Pods.

### 5. What is a Service?

A Service provides a stable network endpoint for accessing a group of Pods.

### 6. Why do we need a Service?

Pod IP addresses can change when Pods are recreated. A Service provides a stable endpoint and routes traffic to matching Pods.

### 7. What is a selector?

A selector identifies resources using matching labels.

### 8. Why would a Service have no endpoints?

Commonly because the Service selector does not match the labels on the available Pods, or there are no ready matching Pods.

### 9. How do you troubleshoot a Service with no endpoints?

```bash
oc get svc
oc describe svc <service>
oc get pods --show-labels
oc get endpoints <service>
oc get endpointslice
```

Then compare the Service selector with Pod labels.

### 10. How do you troubleshoot CrashLoopBackOff?

```bash
oc get pods
oc describe pod <pod>
oc logs <pod>
oc logs <pod> --previous
```

Then investigate the container's exit/error message.

### 11. How do you scale an OpenShift Deployment?

```bash
oc scale deployment <deployment> --replicas=<number>
```

### 12. How do you perform a rollback?

```bash
oc rollout undo deployment/<deployment>
```

### 13. How do you check rollout status?

```bash
oc rollout status deployment/<deployment>
```

### 14. How do you expose a Service in OpenShift?

```bash
oc expose service <service>
```

This creates an OpenShift Route.

---


### `lesson3-app1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson3-web
  labels:
    app: lesson3-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson3-web
  template:
    metadata:
      labels:
        app: lesson3-web
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
  name: lesson3-web
  labels:
    app: lesson3-web
spec:
  type: ClusterIP
  selector:
    app: lesson3-web
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

---


## ✅ Lesson 3 Completed

The main architecture learned in this lesson:

```text
                    OpenShift
                        |
                        v
                   Deployment
                        |
                        v
                   ReplicaSet
                        |
              +---------+---------+
              |                   |
              v                   v
            Pod 1               Pod 2
              |                   |
              +---------+---------+
                        |
                        v
                     Service
                        |
                        v
                      Route
                        |
                        v
                      User
```

**Main lesson:** A Deployment manages the application workload, a ReplicaSet maintains the desired Pod count, Pods run the containers, Services provide stable networking, and OpenShift Routes expose applications externally.
