# OpenShift Learning – Lesson 25: Production Project

## 🚀 Lesson 25 – OpenShift Production Project

> **Environment:** OpenShift Developer Sandbox  
> **Type:** Final hands-on production-style project  
> **Dockerfile:** Not required  
> **Argo CD:** Not required  
> **GitHub Webhook:** Not used — Developer Sandbox returns `403 Forbidden`  
> **Cluster-admin:** Not required  
> **`oc top pods`:** Not available in this Sandbox  
> **Storage:** RWO PVC + single replica for the persistence exercise

---

# 🎯 Objective

Build a complete production-style application on OpenShift by combining the concepts learned throughout the previous lessons.

We will practice:

- Application deployment
- ConfigMap
- Secret
- Persistent storage
- PVC
- Service
- NetworkPolicy
- Route
- TLS
- Resource requests
- Resource limits
- Health probes
- Non-root security
- Autoscaling concepts
- Monitoring
- Logs
- Events
- OpenShift CI/CD concepts
- Production troubleshooting

---

# 🏗️ Final Architecture

```text
                              GitHub
                                 |
                         Manual Build Trigger
                                 |
                                 v
                         +---------------+
                         |  BuildConfig  |
                         +---------------+
                                 |
                                 v
                              Build
                                 |
                                 v
                           ImageStream
                                 |
                                 v
                         +---------------+
                         |  Deployment   |
                         |               |
                         |  Replicas     |
                         |  Probes       |
                         |  Resources    |
                         |  Security     |
                         +---------------+
                           |     |     |
                           |     |     |
                           v     v     v
                         PVC  Config  Secret
                           |
                           v
                         Service
                           |
                     NetworkPolicy
                           |
                           v
                       Route + TLS
                           |
                           v
                         HTTPS
                           |
                         Browser
```

Monitoring:

```text
                     Application
                          |
             +------------+------------+
             |            |            |
             v            v            v
           Logs         Events       Metrics
             |            |            |
             +------------+------------+
                          |
                    Troubleshooting
```

---

# ⚠️ Developer Sandbox Limitations

This lesson is specifically adapted to the restrictions we encountered in the OpenShift Developer Sandbox.

## GitHub Webhook

We previously tested the OpenShift BuildConfig GitHub webhook and received:

```text
403 Forbidden
```

Therefore Lesson 25 does **not** depend on the GitHub webhook.

Instead:

```text
GitHub
   |
   | Manual trigger
   v
OpenShift BuildConfig
```

Use:

```cmd
oc start-build <buildconfig-name> --follow
```

---

## `oc top pods`

The Sandbox CLI returns:

```text
unknown command "top" for "oc"
```

Therefore do not use:

```cmd
oc top pods
```

For monitoring, use:

```cmd
oc logs <pod-name>
```

```cmd
oc get events --sort-by=.lastTimestamp
```

and the OpenShift Web Console metrics where available.

---

## SCC Administration

Do not attempt:

```cmd
oc create scc
```

or:

```cmd
oc adm policy ...
```

These operations require elevated permissions.

We will **understand SCC**, but we will not modify cluster-wide SCC configuration.

---

# 📚 Lessons Combined

Lesson 25 combines the previous lessons:

```text
Lesson 14
Secrets
   ↓
Lesson 15
Non-root Security
   ↓
Lesson 16
Logging & Monitoring
   ↓
Lesson 17
Networking
   ↓
Lesson 18
Storage
   ↓
Lesson 19
Templates
   ↓
Lesson 20
OpenShift Pipelines
   ↓
Lesson 21
OpenShift CI/CD
   ↓
Lesson 22
GitOps Fundamentals
   ↓
Lesson 23
Environment Management
   ↓
Lesson 24
Production Security
   ↓
Lesson 25
COMPLETE PRODUCTION PROJECT
```

---

# Part 1 – Check Your Project

## 1. Check Current Project

```cmd
oc project
```

Make sure you are working in your Developer Sandbox project.

---

## 2. Check Available Resources

```cmd
oc api-resources
```

We are interested in:

```text
deployments
services
routes
configmaps
secrets
persistentvolumeclaims
networkpolicies
horizontalpodautoscalers
buildconfigs
imagestreams
```

---

# Part 2 – Application Configuration

## 3. Create ConfigMap

```cmd
oc create configmap lesson25-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson25-config
```

Describe:

```cmd
oc describe configmap lesson25-config
```

---

# 4. Create Secret

```cmd
oc create secret generic lesson25-secret --from-literal=USERNAME=admin --from-literal=PASSWORD='ChangeThisPassword123!'
```

Verify:

```cmd
oc get secret lesson25-secret
```

Describe:

```cmd
oc describe secret lesson25-secret
```

Important:

```text
Base64 ≠ Encryption
```

Never commit real credentials to Git.

---

# Part 3 – Persistent Storage

## 5. Check StorageClasses

```cmd
oc get storageclass
```

Do not assume a StorageClass name.

Use one that is available in your Sandbox.

---

# 6. Create PVC

Create:

```text
lesson25-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lesson25-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

If your Sandbox requires a StorageClass, add:

```yaml
storageClassName: <your-storage-class>
```

Apply:

```cmd
oc apply -f lesson25-pvc.yaml
```

---

# 7. Verify PVC

```cmd
oc get pvc lesson25-pvc
```

Expected:

```text
STATUS
Bound
```

If it is:

```text
Pending
```

run:

```cmd
oc describe pvc lesson25-pvc
```

Check the Events section.

---

# 8. Understand RWO

Our PVC uses:

```text
ReadWriteOnce
```

Conceptually:

```text
RWO
 ↓
Read/write access from one node
```

This is important because we previously encountered:

```text
Multi-Attach error
```

when two application Pods attempted to use the same RWO volume across nodes.

Therefore, for this persistence exercise:

```text
lesson25 Deployment
        |
        v
replicas: 1
        |
        v
RWO PVC
```

---

# Part 4 – Production Deployment

## 9. Create Deployment

Create:

```text
lesson25-deployment.yaml
```

Use the unprivileged NGINX image.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson25-app
  labels:
    app: lesson25
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lesson25
  template:
    metadata:
      labels:
        app: lesson25
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080

          envFrom:
            - configMapRef:
                name: lesson25-config

          env:
            - name: APP_USERNAME
              valueFrom:
                secretKeyRef:
                  name: lesson25-secret
                  key: USERNAME

            - name: APP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: lesson25-secret
                  key: PASSWORD

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi

          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL

          volumeMounts:
            - name: app-data
              mountPath: /tmp/app-data

          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 20

      volumes:
        - name: app-data
          persistentVolumeClaim:
            claimName: lesson25-pvc
```

Apply:

```cmd
oc apply -f lesson25-deployment.yaml
```

---

# 10. Verify Deployment

```cmd
oc get deployment lesson25-app
```

Expected:

```text
READY
1/1
```

Check Pods:

```cmd
oc get pods -l app=lesson25
```

Expected:

```text
Running
```

---

# 11. Check Pod Details

```cmd
oc describe pod <pod-name>
```

Look for:

```text
Readiness probe
Liveness probe
Volume
Environment
Resources
```

---

# Part 5 – Container Security

## 12. Verify Container User

```cmd
oc exec <pod-name> -- id
```

The container should run as a non-root user.

The exact UID may vary because OpenShift can assign a UID according to its security policy.

---

# 13. Verify SecurityContext

```cmd
oc get pod <pod-name> -o yaml
```

Look for:

```yaml
allowPrivilegeEscalation: false
```

and:

```yaml
capabilities:
  drop:
    - ALL
```

---

# 14. Why Non-root?

Running as non-root reduces the impact of a container compromise.

```text
Root
 ↓
More privileges
 ↓
Larger attack surface
```

versus:

```text
Non-root
 ↓
Reduced privileges
 ↓
Smaller attack surface
```

---

# Part 6 – Persistent Data Test

## 15. Create Test Data

Get the Pod:

```cmd
oc get pods -l app=lesson25
```

Create a file:

```cmd
oc exec <pod-name> -- sh -c "echo Lesson25-Persistent-Data > /tmp/app-data/test.txt"
```

Verify:

```cmd
oc exec <pod-name> -- cat /tmp/app-data/test.txt
```

Expected:

```text
Lesson25-Persistent-Data
```

---

# 16. Delete the Pod

```cmd
oc delete pod <pod-name>
```

Watch:

```cmd
oc get pods -l app=lesson25 -w
```

A replacement Pod should be created.

---

# 17. Verify Data Survived

Get the new Pod:

```cmd
oc get pods -l app=lesson25
```

Then:

```cmd
oc exec <new-pod-name> -- cat /tmp/app-data/test.txt
```

Expected:

```text
Lesson25-Persistent-Data
```

This demonstrates:

```text
Pod deleted
    ↓
New Pod
    ↓
Same PVC
    ↓
Data remains
```

---

# Part 7 – Service

## 18. Create Service

Create:

```text
lesson25-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson25-service
  labels:
    app: lesson25
spec:
  selector:
    app: lesson25
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson25-service.yaml
```

---

# 19. Verify Service

```cmd
oc get service lesson25-service
```

Check endpoints:

```cmd
oc get endpoints lesson25-service
```

The Service should point to the application Pod.

Architecture:

```text
Service
   |
   v
lesson25 Pod
```

---

# Part 8 – NetworkPolicy

## 20. Create NetworkPolicy

Create:

```text
lesson25-networkpolicy.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lesson25-networkpolicy
spec:
  podSelector:
    matchLabels:
      app: lesson25

  policyTypes:
    - Ingress

  ingress:
    - ports:
        - protocol: TCP
          port: 8080
```

Apply:

```cmd
oc apply -f lesson25-networkpolicy.yaml
```

Verify:

```cmd
oc get networkpolicy
```

---

# 21. Understand NetworkPolicy

The policy targets:

```text
app=lesson25
```

and controls:

```text
Ingress
```

on:

```text
TCP 8080
```

Concept:

```text
NetworkPolicy
      ↓
Network traffic control
      ↓
Reduced attack surface
```

---

# Part 9 – Route + TLS

## 22. Create HTTPS Route

Create:

```text
lesson25-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson25-route
spec:
  to:
    kind: Service
    name: lesson25-service

  port:
    targetPort: http

  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Apply:

```cmd
oc apply -f lesson25-route.yaml
```

---

# 23. Get Route

```cmd
oc get route lesson25-route
```

Open:

```text
https://<route-hostname>
```

---

# 24. Understand Edge TLS

```text
Browser
   |
 HTTPS
   |
   v
OpenShift Router
   |
 TLS termination
   |
 HTTP
   |
   v
Service
   |
   v
Pod
```

This is:

```text
Edge TLS termination
```

---

# Part 10 – Resource Management

## 25. Requests

Our application uses:

```yaml
requests:
  cpu: 100m
  memory: 128Mi
```

Requests help the scheduler determine where the Pod can run.

```text
Request
   ↓
Scheduling requirement
```

---

# 26. Limits

Our application uses:

```yaml
limits:
  cpu: 500m
  memory: 256Mi
```

Limits define the maximum resources available to the container.

```text
Limit
   ↓
Maximum resource usage
```

---

# 27. Verify Resources

```cmd
oc get pod <pod-name> -o jsonpath="{.spec.containers[*].resources}"
```

---

# Part 11 – Monitoring

## 28. Do Not Use `oc top`

The Developer Sandbox CLI does not provide:

```cmd
oc top pods
```

Therefore this command is intentionally excluded from Lesson 25.

---

# 29. Application Logs

```cmd
oc logs <pod-name>
```

For a restarted container:

```cmd
oc logs <pod-name> --previous
```

---

# 30. Pod Status

```cmd
oc get pods
```

Useful states include:

```text
Running
Pending
Completed
CrashLoopBackOff
ImagePullBackOff
ContainerCreating
```

---

# 31. Events

```cmd
oc get events --sort-by=.lastTimestamp
```

Events can help identify:

```text
Scheduling problems
Image problems
PVC problems
Probe failures
Container crashes
Network problems
```

---

# 32. OpenShift Console Monitoring

Use the OpenShift Web Console.

Inspect the application/workload and available:

```text
Metrics
CPU
Memory
Pod health
Application status
```

The exact console menus can vary depending on the Sandbox version.

---

# 33. Metrics API Check

Check whether the metrics API is available:

```cmd
oc get apiservice v1beta1.metrics.k8s.io
```

If available:

```cmd
oc get --raw /apis/metrics.k8s.io/v1beta1/pods
```

If the Sandbox returns:

```text
Forbidden
```

or:

```text
NotFound
```

document the limitation.

Do not attempt to bypass permissions.

---

# Part 12 – Autoscaling

## 34. Check HPA

```cmd
oc api-resources | findstr /i horizontalpodautoscaler
```

Then:

```cmd
oc get hpa
```

---

# 35. Understand HPA

Horizontal Pod Autoscaler can increase or decrease the number of replicas according to supported metrics.

Concept:

```text
Low Load
   ↓
2 Pods

High Load
   ↓
3 Pods

Higher Load
   ↓
4 Pods
```

---

# 36. Check Metrics Before HPA

HPA requires metrics.

Therefore:

```text
HPA
 ↓
Metrics API
 ↓
Available?
```

If:

```text
YES
```

we can perform HPA practice.

If:

```text
NO
```

document the Sandbox limitation.

---

# 37. HPA Example

If metrics are available, create:

```text
lesson25-hpa.yaml
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lesson25-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lesson25-app

  minReplicas: 1
  maxReplicas: 3

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Apply:

```cmd
oc apply -f lesson25-hpa.yaml
```

Check:

```cmd
oc get hpa
```

---

# Part 13 – OpenShift CI/CD

## 38. Important CI/CD Decision

We already completed OpenShift-native CI/CD in Lesson 21.

Existing resources:

```text
BuildConfig
    |
    v
Build
    |
    v
ImageStream
    |
    v
Deployment
```

Your Sandbox currently has:

```text
lesson21-app
```

BuildConfig.

Existing Builds include:

```text
lesson21-app-1
lesson21-app-2
```

and ImageStream:

```text
lesson21-app
```

---

# 39. GitHub Webhook Limitation

The GitHub webhook returned:

```text
403 Forbidden
```

Therefore:

```text
GitHub
   |
Webhook
   X
403 Forbidden
```

is not part of this project.

We will use:

```text
GitHub
   |
Manual trigger
   |
OpenShift BuildConfig
```

---

# 40. Check BuildConfigs

```cmd
oc get buildconfig
```

Current known resources:

```text
lesson13-nodejs
lesson21-app
```

---

# 41. Check Builds

```cmd
oc get builds
```

A Build normally has a numeric suffix.

Example:

```text
lesson21-app-1
lesson21-app-2
```

Important distinction:

```text
BuildConfig
     |
     +--> lesson21-app
              |
              +--> Build
                   lesson21-app-1
```

---

# 42. Start a Build Manually

If using the existing Lesson 21 BuildConfig:

```cmd
oc start-build lesson21-app --follow
```

Then:

```cmd
oc get builds
```

You should see a new Build such as:

```text
lesson21-app-3
```

---

# 43. Build Troubleshooting

Check:

```cmd
oc describe build <build-name>
```

Logs:

```cmd
oc logs build/<build-name>
```

Build status:

```cmd
oc get builds
```

---

# Part 14 – Production Troubleshooting

## 44. Pod Troubleshooting

Start with:

```cmd
oc get pods
```

Then:

```cmd
oc describe pod <pod-name>
```

Then:

```cmd
oc logs <pod-name>
```

Then:

```cmd
oc get events --sort-by=.lastTimestamp
```

---

# 45. PVC Troubleshooting

```cmd
oc get pvc
```

Then:

```cmd
oc describe pvc lesson25-pvc
```

Check:

```text
Pending
Bound
StorageClass
Events
```

---

# 46. Service Troubleshooting

```cmd
oc get service lesson25-service
```

Then:

```cmd
oc get endpoints lesson25-service
```

If no endpoints exist, compare:

```text
Service selector
        |
        v
Pod labels
```

Our labels should match:

```text
app=lesson25
```

---

# 47. Route Troubleshooting

```cmd
oc get route lesson25-route
```

Then:

```cmd
oc describe route lesson25-route
```

---

# 48. NetworkPolicy Troubleshooting

```cmd
oc get networkpolicy
```

Then:

```cmd
oc describe networkpolicy lesson25-networkpolicy
```

---

# 49. Deployment Troubleshooting

```cmd
oc describe deployment lesson25-app
```

Check rollout:

```cmd
oc rollout status deployment/lesson25-app
```

---

# Part 15 – Production Validation

## 50. Deployment

```cmd
oc get deployment lesson25-app
```

Expected:

```text
1/1
```

---

# 51. Pod

```cmd
oc get pods -l app=lesson25
```

Expected:

```text
Running
```

---

# 52. PVC

```cmd
oc get pvc lesson25-pvc
```

Expected:

```text
Bound
```

---

# 53. Service

```cmd
oc get service lesson25-service
```

---

# 54. Service Endpoints

```cmd
oc get endpoints lesson25-service
```

---

# 55. NetworkPolicy

```cmd
oc get networkpolicy lesson25-networkpolicy
```

---

# 56. Route

```cmd
oc get route lesson25-route
```

---

# 57. Resources

```cmd
oc get pod <pod-name> -o jsonpath="{.spec.containers[*].resources}"
```

---

# 58. Security

```cmd
oc get pod <pod-name> -o yaml
```

Verify:

```text
allowPrivilegeEscalation: false
```

and:

```text
drop:
- ALL
```

---

# 59. Non-root

```cmd
oc exec <pod-name> -- id
```

---


# 🧠 Production Security Checklist

```text
☐ Non-root container
☐ No privilege escalation
☐ Drop unnecessary capabilities
☐ Secret for sensitive data
☐ ConfigMap for non-sensitive data
☐ HTTPS
☐ TLS
☐ NetworkPolicy
☐ Resource requests
☐ Resource limits
☐ Persistent storage
☐ Health probes
☐ Trusted image
☐ Fixed image version
```

---

# 🧠 Production Operations Checklist

```text
☐ Check Pod status
☐ Check logs
☐ Check events
☐ Check PVC
☐ Check Service endpoints
☐ Check Route
☐ Check NetworkPolicy
☐ Check resources
☐ Check metrics
☐ Check HPA
☐ Check BuildConfig
☐ Check Builds
☐ Troubleshoot failures
```

---

# 🧠 Questions You Should Be Able to Answer

## 1. Why use a Deployment?

```text
Declarative application management
+
Replica management
+
Self-healing
+
Rolling updates
```

---

## 2. Why use a Service?

```text
Stable network endpoint
+
Pod discovery
+
Load balancing
```

---

## 3. Why use a Route?

```text
External access
+
Hostname
+
TLS
```

---

## 4. Why use a PVC?

```text
Persistent application data
```

---

## 5. Why use NetworkPolicy?

```text
Control network communication
```

---

## 6. Why use resource requests?

```text
Scheduling
```

---

## 7. Why use resource limits?

```text
Resource protection
```

---

## 8. Why use HPA?

```text
Automatic horizontal scaling
```

---

## 9. Why use monitoring?

```text
Visibility
+
Troubleshooting
+
Capacity awareness
```

---

## 10. Why use CI/CD?

```text
Automated and repeatable
application delivery
```

---

# 🏁 Final OpenShift Production Model

```text
                           Git
                            |
                     Manual CI/CD
                            |
                         Build
                            |
                          Image
                            |
                       Deployment
                            |
            +---------------+---------------+
            |               |               |
           PVC          ConfigMap         Secret
            |               |               |
            +---------------+---------------+
                            |
                         Service
                            |
                      NetworkPolicy
                            |
                         Route
                            |
                           TLS
                            |
                          HTTPS
                            |
                          User
```

Around the application:

```text
Security
   +
Resource Management
   +
Autoscaling
   +
Monitoring
   +
Troubleshooting
```

---

# 🎓 OpenShift Learning Path – Completed

```text
Lesson 01
OpenShift Fundamentals
        ↓
Lesson 02
Projects, Users, ServiceAccounts & RBAC
        ↓
Lesson 03
Pods
        ↓
Lesson 04
Deployments
        ↓
Lesson 05
Services
        ↓
Lesson 06
Routes
        ↓
Lesson 07
ConfigMaps
        ↓
Lesson 08
Secrets
        ↓
Lesson 09
Storage
        ↓
Lesson 10
Networking
        ↓
Lesson 11
Security
        ↓
Lesson 12
ImageStreams
        ↓
Lesson 13
Builds
        ↓
Lesson 14
Secrets / Configuration
        ↓
Lesson 15
Non-root Containers
        ↓
Lesson 16
Logging & Monitoring
        ↓
Lesson 17
Advanced Networking
        ↓
Lesson 18
Persistent Storage
        ↓
Lesson 19
OpenShift Templates
        ↓
Lesson 20
OpenShift Pipelines
        ↓
Lesson 21
OpenShift CI/CD
        ↓
Lesson 22
GitOps Fundamentals
        ↓
Lesson 23
Environment Management
        ↓
Lesson 24
Production Security
        ↓
Lesson 25
PRODUCTION PROJECT
```

---

# 🏆 Final Goal

After completing Lesson 25, I should be able to explain and demonstrate a production-style OpenShift application:

```text
Application
     |
     +-- Deployment
     |
     +-- Service
     |
     +-- Route / TLS
     |
     +-- ConfigMap
     |
     +-- Secret
     |
     +-- Persistent Storage
     |
     +-- NetworkPolicy
     |
     +-- Resource Requests/Limits
     |
     +-- Health Probes
     |
     +-- Non-root Security
     |
     +-- Autoscaling Concepts
     |
     +-- Monitoring
     |
     +-- CI/CD
     |
     +-- Troubleshooting
```

## 🎯 Lesson 25 Completion Statement

> **I can deploy, secure, expose, store data for, troubleshoot, monitor, scale, and operate a production-style application on OpenShift, while understanding the limitations of a Developer Sandbox and using OpenShift-native CI/CD.**

---

# 🚀 OpenShift Course Status

## Core OpenShift Learning

**COMPLETED ✅**

## Production Application Skills

**COMPLETED / PRACTICED ✅**

## OpenShift CI/CD

**COMPLETED / PRACTICED ✅**

## Production Security

**COMPLETED / PRACTICED ✅**

## GitOps Fundamentals

**COMPLETED CONCEPTUALLY ✅**

## Argo CD

**Not completed in Sandbox ⚠️**

Reason:

```text
Developer Sandbox
      ↓
GitOps / Argo CD access restrictions
      ↓
Forbidden
```

This is an environment limitation, not a missing fundamental OpenShift skill.

---

# 📌 What Comes Next?

At this point, I should **not keep adding random OpenShift lessons**.

The next useful step is to move toward:

```text
OpenShift
    +
AWS
    +
Azure
    +
GCP
    +
Kubernetes
    +
Terraform
    +
Ansible
    +
CI/CD
    +
DevOps Architecture
```

The goal is to turn the OpenShift knowledge into **real DevOps project and interview capability**.

---

# 🏁 LESSON 25 COMPLETE

```text
                     OPENSHIFT
                         |
       +-----------------+-----------------+
       |                 |                 |
    Security           Storage          Networking
       |                 |                 |
      SCC               PVC          Service / Route
      Secret             |                 |
   Non-root              |                TLS
       |                 |                 |
       +-----------------+-----------------+
                         |
                    Deployment
                         |
              +----------+----------+
              |          |          |
          Resources     HPA     Monitoring
              |
             CI/CD
              |
          BuildConfig
              |
            Build
              |
          ImageStream
```

**Lesson 25 = OpenShift Production Project**

**OpenShift core learning path = COMPLETE ✅**
