# OpenShift Learning – Lesson 9: Routes & External Application Access

## 🎯 Objectives

In this lesson, I learned and practiced:

- OpenShift Routes
- Route vs Service
- Creating Routes
- Route hostname
- External application access
- Route → Service → Pod traffic flow
- HTTP and HTTPS
- TLS termination
- Edge TLS termination
- Re-encrypt TLS concept
- Passthrough TLS concept
- Route troubleshooting
- Service endpoint troubleshooting

## 1. What is an OpenShift Route?

A Route exposes a Service externally using HTTP or HTTPS.

Basic architecture:

```text
Browser
   |
   v
OpenShift Route
   |
   v
Service
   |
   v
Pod
```

The basic relationship is:

```text
Route → Service → Pod
```

---
## 2. Service vs Route

| Resource | Purpose |
|---|---|
| Pod | Runs the application |
| Service | Provides stable internal access |
| Route | Provides external HTTP/HTTPS access |

Architecture:

```text
Pod
 ↓
Service
 ↓
Route
 ↓
Internet / Browser
```

---
## 3. Check Current Project

```bash
oc project
```

If required:

```bash
oc project lakshminarayananredh-dev
```

---
## 4. Create Web Application

Created `lesson9-web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson9-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson9-web
  template:
    metadata:
      labels:
        app: lesson9-web
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
  name: lesson9-web
spec:
  type: ClusterIP
  selector:
    app: lesson9-web
  ports:
    - port: 80
      targetPort: 8080
```

Apply:

```bash
oc apply -f lesson9-web.yaml
```

Verify:

```bash
oc get pods
oc get svc lesson9-web
```

Expected:

```text
NAME                          READY   STATUS
lesson9-web-xxxxx             1/1     Running
lesson9-web-yyyyy             1/1     Running
```

---
## 5. Test the Service Internally

Create a temporary client Pod:

```bash
oc run lesson9-client \
  --image=curlimages/curl \
  --command -- sleep 3600
```

Check:

```bash
oc get pod lesson9-client
```

Test:

```bash
oc exec lesson9-client -- curl http://lesson9-web
```

The NGINX response should be returned.

Traffic flow:

```text
Client Pod
    |
    v
Service
    |
    v
lesson9-web Pods
```

---
## 6. Create an OpenShift Route

Expose the Service:

```bash
oc expose service lesson9-web
```

Check:

```bash
oc get route
```

Example:

```text
NAME          HOST/PORT
lesson9-web   lesson9-web-xxxxx.apps.example.com
```

---
## 7. Get Route Hostname

```bash
oc get route lesson9-web
```

Or:

```bash
oc get route lesson9-web -o jsonpath="{.spec.host}"
```

The hostname will be different for each OpenShift environment.

---
## 8. Access Application from Browser

Copy the Route hostname and open:

```text
http://<route-hostname>
```

Example:

```text
http://lesson9-web-xxxxx.apps.example.com
```

The NGINX welcome page should appear.

The application is now accessible externally.

---
## 9. Route Traffic Flow

External request:

```text
Browser
   |
   | HTTP/HTTPS
   v
OpenShift Route
   |
   v
Service
   |
   +------ Pod 1
   |
   +------ Pod 2
```

The Route provides external access while the Service continues to select the backend Pods.

---
## 10. Inspect Route YAML

```bash
oc get route lesson9-web -o yaml
```

Important section:

```yaml
spec:
  host:
  to:
    kind: Service
    name: lesson9-web
```

The Route points to the Service, not directly to the Pods.

```text
Route
  |
  +--> Service: lesson9-web
```

---
## 11. Check Route Status

```bash
oc describe route lesson9-web
```

Look for the `Admitted` condition.

A healthy Route should show:

```text
Admitted: True
```

---
## 12. Test Route Using curl

Get the hostname:

```bash
oc get route lesson9-web -o jsonpath="{.spec.host}"
```

Then:

```bash
curl http://<route-hostname>
```

The NGINX response should be returned.

---
## 13. HTTP vs HTTPS

OpenShift Routes can provide:

```text
HTTP
```

or:

```text
HTTPS
```

For production applications, HTTPS should normally be used.

Basic HTTPS architecture:

```text
HTTPS Client
     |
     v
OpenShift Route
     |
     v
Service
     |
     v
Pod
```

---
## 14. TLS Termination

OpenShift Routes support different TLS termination methods.

### Edge

```text
Client
  |
 HTTPS
  |
  v
Route
  |
 HTTP
  |
  v
Service
  |
  v
Pod
```

TLS is terminated at the OpenShift Router.

### Re-encrypt

```text
Client
  |
 HTTPS
  |
  v
Route
  |
 HTTPS
  |
  v
Service
  |
  v
Pod
```

TLS is terminated at the router and a new TLS connection is established to the backend.

### Passthrough

```text
Client
  |
 HTTPS
  |
  v
Route
  |
 HTTPS
  |
  v
Pod
```

The router does not terminate TLS. The application handles TLS.

---
## 15. Easy Way to Remember TLS Modes

```text
Edge
Client HTTPS → Route → HTTP → Application

Re-encrypt
Client HTTPS → Route → HTTPS → Application

Passthrough
Client HTTPS → Route → HTTPS → Application
              TLS handled by Application
```

For this beginner lesson, the main focus is understanding **Edge TLS termination**.

---
## 16. Create HTTPS Edge Route

Create an Edge Route:

```bash
oc create route edge lesson9-web-secure \
  --service=lesson9-web
```

Check:

```bash
oc get route
```

Get the hostname:

```bash
oc get route lesson9-web-secure -o jsonpath="{.spec.host}"
```

Access:

```text
https://<route-hostname>
```

Inspect:

```bash
oc describe route lesson9-web-secure
```

Look for:

```text
TLS Termination: edge
```

---
## 17. Route Troubleshooting

The normal traffic path is:

```text
Route
  |
  v
Service
  |
  v
Endpoints
  |
  v
Pods
```

Check the Route:

```bash
oc get route lesson9-web
```

Check the Service:

```bash
oc get svc lesson9-web
```

Check endpoints:

```bash
oc get endpoints lesson9-web
```

Check Pods:

```bash
oc get pods
```

---
## 18. Troubleshooting Scenario – No Service Endpoints

Break the Service selector:

```bash
oc patch service lesson9-web --type=merge -p '{"spec":{"selector":{"app":"wrong-app"}}}'
```

Check:

```bash
oc get endpoints lesson9-web
```

Expected:

```text
ENDPOINTS   <none>
```

The Route still exists, but the Service has no backend endpoints.

Traffic flow:

```text
Route
   |
   v
Service
   |
   X
No Endpoints
   |
   X
Application unavailable
```

Fix the Service selector:

```bash
oc patch service lesson9-web --type=merge -p '{"spec":{"selector":{"app":"lesson9-web"}}}'
```

Verify:

```bash
oc get endpoints lesson9-web
```

The backend Pod endpoints should appear again.

---
## 19. Route Troubleshooting Sequence

When a Route does not work, troubleshoot in this order.

### Step 1 – Check Pods

```bash
oc get pods
```

### Step 2 – Check Service

```bash
oc get svc lesson9-web
```

### Step 3 – Check Endpoints

```bash
oc get endpoints lesson9-web
```

### Step 4 – Check Route

```bash
oc get route lesson9-web
```

### Step 5 – Describe Route

```bash
oc describe route lesson9-web
```

### Step 6 – Test Service Internally

```bash
oc exec lesson9-client -- curl http://lesson9-web
```

### Step 7 – Test Route Externally

```bash
curl http://<route-hostname>
```

---
## 20. Common Route Problems

| Problem | Possible Cause |
|---|---|
| Route hostname does not resolve | DNS/Route issue |
| Route exists but application unavailable | Service/Pod problem |
| Service has no endpoints | Selector mismatch |
| Pod is NotReady | Readiness probe failure |
| HTTP works but HTTPS does not | TLS configuration |
| Route not admitted | Route/router configuration |
| 503 response | Backend unavailable |
| 404 response | Wrong application path |
| Connection timeout | Networking/Route issue |

---
## 21. Real-World Architecture

```text
                 Internet
                    |
                    v
              DNS / Domain
                    |
                    v
            OpenShift Route
                    |
                    v
                Service
                    |
          +---------+---------+
          |                   |
          v                   v
       Pod 1               Pod 2
          |                   |
          +---------+---------+
                    |
                    v
               Application
```

With health checks:

```text
Internet
   |
   v
Route
   |
   v
Service
   |
   +---- Ready Pod 1
   |
   +---- NotReady Pod 2
```

The Service sends traffic only to Ready Pods.

---
## 22. Hands-On Challenge

Create your own application:

```text
lesson9-challenge
```

Requirements:

- Deployment
- 2 replicas
- NGINX
- Service
- ClusterIP
- Readiness Probe
- OpenShift Route
- External browser access

Final architecture:

```text
Browser
   |
   v
Route
   |
   v
Service
   |
   +---- Pod 1
   |
   +---- Pod 2
```

Verify:

```bash
oc get deployment
oc get pods
oc get svc
oc get endpoints
oc get route
```

Then access the Route from the browser.

---
## 23. Interview Questions

1. What is an OpenShift Route?
2. Why do we need a Route?
3. What is the difference between a Service and a Route?
4. Can a Route point directly to a Pod?
5. What does a Route point to?
6. How do you create a Route?
7. How do you find the Route hostname?
8. What is TLS termination?
9. What is Edge TLS termination?
10. What is Re-encrypt TLS termination?
11. What is Passthrough TLS termination?
12. What happens if the Service has no endpoints?
13. How would you troubleshoot a Route returning an error?
14. What is the relationship between Route, Service and Pod?
15. How does a Readiness Probe affect Route traffic?

---
## 24. Useful Commands

```bash
oc get route
oc get route <route-name>
oc describe route <route-name>
oc get route <route-name> -o yaml
oc get route <route-name> -o jsonpath="{.spec.host}"

oc get svc
oc describe svc <service-name>

oc get endpoints <service-name>
oc get endpointslice

oc get pods
oc describe pod <pod-name>
oc logs <pod-name>
```

---
## 🧹 Cleanup

Remove Routes:

```bash
oc delete route lesson9-web
oc delete route lesson9-web-secure
```

Remove application:

```bash
oc delete -f lesson9-web.yaml
```

Remove client:

```bash
oc delete pod lesson9-client
```

---
## ✅ Lesson 9 Completion Checklist

- [x] Understand OpenShift Route
- [x] Understand Service vs Route
- [x] Create a Route
- [x] Find Route hostname
- [x] Access application using browser
- [x] Test Route using curl
- [x] Understand Route → Service → Pod
- [x] Understand HTTP vs HTTPS
- [x] Understand Edge TLS termination
- [x] Understand Re-encrypt concept
- [x] Understand Passthrough concept
- [x] Troubleshoot Route connectivity
- [x] Troubleshoot Service with no endpoints
- [x] Understand how Readiness affects traffic

---
## 🧠 Final Memory Trick

```text
Pod      → Runs application
Service  → Provides stable internal access
Route    → Provides external HTTP/HTTPS access
```

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
Ready Pods
```

---
**Key Concept:**

```text
Route → Service → Ready Pods
```
