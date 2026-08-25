# OpenShift Learning – Lesson 14: Troubleshooting & Debugging

## 🎯 Objectives

In this lesson, I learned and practiced:

- OpenShift troubleshooting methodology
- Pod troubleshooting
- `CrashLoopBackOff`
- `ImagePullBackOff`
- `CreateContainerConfigError`
- Pending Pods
- PVC troubleshooting
- Deployment troubleshooting
- Rollouts and Rollbacks
- Service troubleshooting
- Endpoint/EndpointSlice troubleshooting
- Route troubleshooting
- NetworkPolicy troubleshooting
- Application logs
- Previous container logs
- OpenShift Events
- Using `oc exec` for troubleshooting
- End-to-end application troubleshooting

---

# 1. OpenShift Troubleshooting Method

When an application is not working, don't randomly run commands.

Use this general flow:

```text
Application Problem
       |
       v
   Check Pods
       |
       v
  Describe Pod
       |
       v
   Check Logs
       |
       v
   Check Events
       |
       v
 Check Deployment
       |
       v
  Check Service
       |
       v
 Check Endpoints
       |
       v
   Check Route
       |
       v
 Check NetworkPolicy
```

---

# 2. Check Current Project

```bash
oc project
```

Check the overall project status:

```bash
oc status
```

Check all resources:

```bash
oc get all
```

---

# 3. Check Pod Status

```bash
oc get pods
```

Important Pod states:

```text
Running
Pending
Completed
Error
CrashLoopBackOff
ImagePullBackOff
CreateContainerConfigError
ContainerCreating
Terminating
```

The first troubleshooting step is usually:

```bash
oc get pods
```

---

# 4. Describe a Pod

Use:

```bash
oc describe pod <pod-name>
```

Pay special attention to:

```text
Events:
```

Events may show:

```text
Failed
BackOff
FailedMount
FailedScheduling
Failed to pull image
```

Memory trick:

```text
oc get pods
      ↓
Find the problem
      ↓
oc describe pod
      ↓
Find the reason
```

---

# 5. Check Application Logs

```bash
oc logs <pod-name>
```

Follow logs:

```bash
oc logs -f <pod-name>
```

For a specific container:

```bash
oc logs <pod-name> -c <container-name>
```

Logs are useful for finding:

```text
Application errors
Startup errors
Runtime errors
Configuration errors
Application crashes
```

---

# 6. Check Previous Container Logs

For containers that have restarted:

```bash
oc logs <pod-name> --previous
```

This is especially useful for:

```text
CrashLoopBackOff
```

Example troubleshooting flow:

```text
CrashLoopBackOff
       |
       v
oc describe pod
       |
       v
oc logs
       |
       v
oc logs --previous
       |
       v
Find application error
```

---

# 7. Practice: CrashLoopBackOff

Create an intentionally broken application:

```bash
oc create deployment lesson14-crash --image=busybox:1.36 -- /bin/sh -c "echo Starting application; exit 1"
```

Check:

```bash
oc get pods
```

The Pod should eventually show:

```text
CrashLoopBackOff
```

Now troubleshoot:

```bash
oc describe pod <pod-name>
```

```bash
oc logs <pod-name>
```

```bash
oc logs <pod-name> --previous
```

Expected application output:

```text
Starting application
```

The container exits with:

```text
exit 1
```

Therefore:

```text
Application Starts
      |
      v
Application Exits
      |
      v
Container Stops
      |
      v
Kubernetes Restarts Container
      |
      v
CrashLoopBackOff
```

---

# 8. Fix CrashLoopBackOff

Delete the broken Deployment:

```bash
oc delete deployment lesson14-crash
```

Create a working Deployment:

```bash
oc create deployment lesson14-crash --image=busybox:1.36 -- /bin/sh -c "echo Application running; sleep 3600"
```

Check:

```bash
oc get pods
```

Expected:

```text
1/1   Running
```

---

# 9. CreateContainerConfigError

Create a Secret:

```bash
oc create secret generic lesson14-secret --from-literal=USERNAME=admin
```

Create a Pod that expects a missing Secret key:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson14-config-error
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo $DB_USERNAME; sleep 3600"]
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: lesson14-secret
              key: DB_USERNAME
```

Save as:

```text
lesson14-config-error.yaml
```

Apply:

```bash
oc apply -f lesson14-config-error.yaml
```

Check:

```bash
oc get pod lesson14-config-error
```

Expected:

```text
CreateContainerConfigError
```

Troubleshoot:

```bash
oc describe pod lesson14-config-error
```

The problem is:

```text
DB_USERNAME
```

does not exist in:

```text
lesson14-secret
```

---

# 10. Fix CreateContainerConfigError

Create the missing key:

```bash
oc create secret generic lesson14-secret --from-literal=USERNAME=admin --from-literal=DB_USERNAME=mydbuser --dry-run=client -o yaml | oc apply -f -
```

Delete the failed Pod:

```bash
oc delete pod lesson14-config-error
```

Recreate it:

```bash
oc apply -f lesson14-config-error.yaml
```

Check:

```bash
oc get pods
```

Expected:

```text
1/1   Running
```

---

# 11. ImagePullBackOff

Create an application with an invalid image:

```bash
oc create deployment lesson14-image --image=nginx:this-image-does-not-exist
```

Check:

```bash
oc get pods
```

Expected:

```text
ImagePullBackOff
```

Troubleshoot:

```bash
oc describe pod <pod-name>
```

Look at the Events section.

Typical causes:

```text
Wrong image name
Wrong image tag
Image does not exist
Private registry authentication problem
Registry problem
```

---

# 12. Fix ImagePullBackOff

Delete the broken Deployment:

```bash
oc delete deployment lesson14-image
```

Create a valid Deployment:

```bash
oc create deployment lesson14-image --image=nginx:latest
```

Check:

```bash
oc get pods
```

Expected:

```text
1/1   Running
```

---

# 13. Pending Pod Troubleshooting

If:

```bash
oc get pods
```

shows:

```text
Pending
```

run:

```bash
oc describe pod <pod-name>
```

Check Events.

Possible causes:

```text
Insufficient CPU
Insufficient memory
Scheduling restrictions
Node constraints
PVC unavailable
```

General flow:

```text
Pending
   |
   v
oc describe pod
   |
   v
Check Events
   |
   v
Find scheduling/storage/resource problem
```

---

# 14. PVC Troubleshooting

Check PVCs:

```bash
oc get pvc
```

If:

```text
STATUS
Pending
```

run:

```bash
oc describe pvc <pvc-name>
```

Check StorageClasses:

```bash
oc get storageclass
```

Troubleshooting flow:

```text
PVC Pending
    |
    v
oc describe pvc
    |
    v
Check Events
    |
    v
Check StorageClass
```

---

# 15. Deployment Troubleshooting

Check Deployments:

```bash
oc get deployment
```

Describe:

```bash
oc describe deployment <deployment-name>
```

Check ReplicaSets:

```bash
oc get rs
```

Check Pods:

```bash
oc get pods
```

Understand the relationship:

```text
Deployment
     |
     v
ReplicaSet
     |
     v
Pods
```

---

# 16. Check Deployment Rollout

```bash
oc rollout status deployment/<deployment-name>
```

Check rollout history:

```bash
oc rollout history deployment/<deployment-name>
```

If a deployment update is broken:

```bash
oc rollout undo deployment/<deployment-name>
```

Check again:

```bash
oc rollout status deployment/<deployment-name>
```

Important concept:

```text
New Version
     |
     v
Deployment Rollout
     |
     v
Problem
     |
     v
Rollback
     |
     v
Previous Version
```

---

# 17. Service Troubleshooting

If the Pod is:

```text
Running
```

but the application is not accessible, check the Service.

```bash
oc get svc
```

Describe:

```bash
oc describe svc <service-name>
```

Check the Service selector.

Example:

```yaml
selector:
  app: lesson14-web
```

Check Pod labels:

```bash
oc get pods --show-labels
```

The Service selector must match the Pod labels.

Example:

```text
Service selector:

app=lesson14-web

        ↓

Pod label:

app=lesson14-web
```

---

# 18. Endpoint Troubleshooting

Check:

```bash
oc get endpoints <service-name>
```

If:

```text
ENDPOINTS
<none>
```

the Service is not finding matching Pods.

Also check EndpointSlices:

```bash
oc get endpointslice
```

Troubleshooting flow:

```text
Service
   |
   v
Check selector
   |
   v
Check Pod labels
   |
   v
Check EndpointSlice
```

---

# 19. Route Troubleshooting

Check Routes:

```bash
oc get route
```

Describe:

```bash
oc describe route <route-name>
```

Check:

```text
Hostname
Service
Target Port
```

Application flow:

```text
Browser
   |
   v
Route
   |
   v
Service
   |
   v
Pod
```

If the browser cannot access the application, troubleshoot each layer.

---

# 20. NetworkPolicy Troubleshooting

Check NetworkPolicies:

```bash
oc get networkpolicy
```

Describe:

```bash
oc describe networkpolicy <policy-name>
```

Check:

```text
PodSelector
Ingress
Egress
Ports
Source Pods
```

NetworkPolicy determines:

```text
Who can communicate?
Which direction?
Which port?
```

---

# 21. Test a Service From Inside the Cluster

Create a temporary client:

```bash
oc run lesson14-client --image=curlimages/curl --command -- sleep 3600
```

Check:

```bash
oc get pods
```

Test the Service:

```bash
oc exec lesson14-client -- curl http://<service-name>:<port>
```

This helps determine whether the problem is:

```text
Application/Service problem
```

or:

```text
External access/Route problem
```

---

# 22. Test Application From Inside a Pod

Enter a running Pod:

```bash
oc exec -it <pod-name> -- /bin/sh
```

Test localhost:

```bash
curl localhost:8080
```

or:

```bash
curl localhost:80
```

depending on the application.

Exit:

```bash
exit
```

---

# 23. Logs vs Describe

This is an important distinction.

### `oc describe`

Used for:

```text
Resource configuration
Events
Scheduling
Volumes
Networking
Environment
Status
```

Command:

```bash
oc describe pod <pod-name>
```

### `oc logs`

Used for:

```text
Application output
Application errors
Runtime errors
Startup failures
```

Command:

```bash
oc logs <pod-name>
```

Easy memory:

```text
describe → OpenShift/Kubernetes information

logs     → Application information
```

---

# 24. OpenShift Events

One of the most useful troubleshooting commands:

```bash
oc get events --sort-by=.lastTimestamp
```

Events can show:

```text
Scheduled
Pulling
Pulled
Created
Started
Failed
BackOff
FailedMount
FailedScheduling
```

Think of Events as:

```text
Events = Timeline of what happened
```

---

# 25. Complete Troubleshooting Decision Tree

```text
Application Not Working
        |
        v
   oc get pods
        |
        +---- Pending
        |       |
        |       v
        |   oc describe pod
        |
        +---- CrashLoopBackOff
        |       |
        |       v
        |   oc logs
        |   oc logs --previous
        |
        +---- ImagePullBackOff
        |       |
        |       v
        |   oc describe pod
        |
        +---- CreateContainerConfigError
        |       |
        |       v
        |   Check Secret/ConfigMap
        |
        +---- Running
                |
                v
           Check Service
                |
                v
         Check Endpoints
                |
                v
           Check Route
                |
                v
       Check NetworkPolicy
```

---

# 26. Final Hands-On Challenge

Create a simple NGINX application:

```bash
oc create deployment lesson14-web --image=nginx:latest
```

Check:

```bash
oc get pods
```

Expose the Deployment:

```bash
oc expose deployment lesson14-web --port=80
```

Check:

```bash
oc get svc
```

Create a Route:

```bash
oc expose service lesson14-web
```

Check:

```bash
oc get route
```

Open the Route in the browser.

---

# 27. Break the Service

Check the Service:

```bash
oc get svc lesson14-web -o yaml
```

Check Pod labels:

```bash
oc get pods --show-labels
```

Temporarily change the Service selector so that it does not match the Pods.

Then:

```bash
oc get endpoints lesson14-web
```

Expected:

```text
<none>
```

Now identify the problem:

```text
Service selector
       ↓
Pod labels
       ↓
No match
       ↓
No endpoints
```

Fix the selector.

Verify:

```bash
oc get endpoints lesson14-web
```

The Pod endpoints should appear again.

---

# 28. Full Troubleshooting Practice

Create:

```text
lesson14-web
```

with:

```text
Deployment
Service
Route
```

Verify:

```bash
oc get pods
oc get deployment
oc get svc
oc get route
oc get endpoints
```

Then practice these scenarios:

### Scenario 1 – CrashLoopBackOff

Find the problem using:

```bash
oc logs <pod>
oc logs <pod> --previous
```

### Scenario 2 – ImagePullBackOff

Find the problem using:

```bash
oc describe pod <pod>
```

### Scenario 3 – CreateContainerConfigError

Check:

```bash
oc describe pod <pod>
```

Then inspect:

```bash
oc get secret
oc get configmap
```

### Scenario 4 – Service Has No Endpoints

Check:

```bash
oc get endpoints <service>
```

Then compare:

```bash
oc get svc <service> -o yaml
oc get pods --show-labels
```

### Scenario 5 – Route Not Working

Troubleshoot:

```text
Route
  ↓
Service
  ↓
Endpoints
  ↓
Pod
```

---

# 29. Most Important Commands

## Pods

```bash
oc get pods
oc describe pod <pod>
oc logs <pod>
oc logs <pod> --previous
oc exec -it <pod> -- /bin/sh
```

## Events

```bash
oc get events --sort-by=.lastTimestamp
```

## Deployment

```bash
oc get deployment
oc describe deployment <deployment>
oc rollout status deployment/<deployment>
oc rollout history deployment/<deployment>
oc rollout undo deployment/<deployment>
```

## Service

```bash
oc get svc
oc describe svc <service>
```

## Endpoints

```bash
oc get endpoints <service>
oc get endpointslice
```

## Route

```bash
oc get route
oc describe route <route>
```

## Storage

```bash
oc get pvc
oc describe pvc <pvc>
oc get storageclass
```

## Network

```bash
oc get networkpolicy
oc describe networkpolicy <policy>
```

---

# 🧠 Final Memory Trick

When an OpenShift application doesn't work:

```text
1. POD
   ↓
oc get pods

2. WHY?
   ↓
oc describe pod

3. APPLICATION?
   ↓
oc logs

4. PREVIOUS CRASH?
   ↓
oc logs --previous

5. EVENTS?
   ↓
oc get events

6. DEPLOYMENT?
   ↓
oc get deployment

7. SERVICE?
   ↓
oc get svc

8. ENDPOINTS?
   ↓
oc get endpoints

9. ROUTE?
   ↓
oc get route

10. NETWORK?
   ↓
oc get networkpolicy
```

---

# ✅ Lesson 14 Completion Checklist

- [ ] Understand OpenShift troubleshooting methodology
- [ ] Understand Pod states
- [ ] Troubleshoot `CrashLoopBackOff`
- [ ] Use `oc logs`
- [ ] Use `oc logs --previous`
- [ ] Use `oc describe pod`
- [ ] Use OpenShift Events
- [ ] Troubleshoot `ImagePullBackOff`
- [ ] Troubleshoot `CreateContainerConfigError`
- [ ] Troubleshoot Pending Pods
- [ ] Troubleshoot PVC problems
- [ ] Troubleshoot Deployments
- [ ] Understand Deployment Rollouts
- [ ] Perform a Rollback
- [ ] Troubleshoot Services
- [ ] Troubleshoot Service selectors
- [ ] Troubleshoot Endpoints/EndpointSlices
- [ ] Troubleshoot Routes
- [ ] Troubleshoot NetworkPolicies
- [ ] Use `oc exec` for application testing
- [ ] Complete an end-to-end troubleshooting scenario

---

# 🏁 Lesson 14 Goal

By the end of this lesson, I should be able to troubleshoot an OpenShift application systematically instead of randomly trying commands.

The core methodology is:

```text
Pod
 ↓
Describe
 ↓
Logs
 ↓
Events
 ↓
Deployment
 ↓
Service
 ↓
Endpoints
 ↓
Route
 ↓
NetworkPolicy
```

**Lesson 14 Topic: OpenShift Troubleshooting & Debugging**

---
