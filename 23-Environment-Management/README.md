# OpenShift Learning – Lesson 23: Environment Management

## 🚀 Lesson 23: DEV → QA → STAGE → UAT → PROD

> **Environment:** OpenShift Developer Sandbox  
> **Approach:** Namespace-level / project-level practice  
> **Dockerfile:** Not required  
> **Argo CD:** Not required  
> **Cluster-admin access:** Not required  
> **NGINX:** Uses `nginxinc/nginx-unprivileged` to avoid the non-root permission problems encountered in previous lessons.

---

# 🎯 Learning Objectives

By the end of this lesson, I should understand:

- DEV environment
- QA environment
- STAGE environment
- UAT environment
- PROD environment
- Environment-specific configuration
- ConfigMaps
- Secrets
- Environment variables
- Labels
- Service selectors
- Environment-specific replica counts
- Same application / different configuration
- Environment promotion
- Configuration drift
- GitOps integration with environments

---

# 1. What Are Environments?

A typical application lifecycle looks like:

```text
Developer
   ↓
DEV
   ↓
QA
   ↓
STAGE
   ↓
UAT
   ↓
PRODUCTION
```

Each environment has a different purpose.

| Environment | Purpose |
|---|---|
| DEV | Developer development |
| QA | Quality/testing |
| STAGE | Production-like testing |
| UAT | User acceptance testing |
| PROD | Real users |

---

# 2. Developer Sandbox Limitation

In a real organization, environments are often separated into namespaces or clusters:

```text
project-dev
project-qa
project-stage
project-uat
project-prod
```

However, our Developer Sandbox has restrictive permissions.

We have already encountered:

```text
Forbidden
```

for cluster-level and namespace-level operations.

Therefore, we will simulate multiple environments **inside our existing accessible OpenShift project**.

We will create:

```text
lesson23-dev
lesson23-qa
lesson23-stage
lesson23-uat
lesson23-prod
```

Each environment gets its own:

```text
Deployment
Service
Route
ConfigMap
```

---

# 3. Architecture

```text
                         OpenShift Project
                                |
        +-----------------------+-----------------------+
        |                       |                       |
        v                       v                       v
       DEV                     QA                    STAGE
  lesson23-dev            lesson23-qa          lesson23-stage
        |                       |                       |
        +-----------------------+-----------------------+
                                |
                         +------+------+
                         |             |
                         v             v
                        UAT           PROD
                  lesson23-uat   lesson23-prod
```

---

# 4. Same Application, Different Configuration

The application image is the same:

```text
nginxinc/nginx-unprivileged:1.25
```

But configuration changes between environments.

```text
DEV
replicas = 1
LOG_LEVEL = debug

QA
replicas = 2
LOG_LEVEL = info

STAGE
replicas = 2
LOG_LEVEL = info

UAT
replicas = 2
LOG_LEVEL = warn

PROD
replicas = 3
LOG_LEVEL = warn
```

The principle is:

> **Same application + environment-specific configuration**

---

# 5. Environment Variables

We will use:

```text
APP_ENV
LOG_LEVEL
APP_VERSION
```

DEV:

```text
APP_ENV=development
LOG_LEVEL=debug
APP_VERSION=1.0
```

QA:

```text
APP_ENV=qa
LOG_LEVEL=info
APP_VERSION=1.0
```

STAGE:

```text
APP_ENV=staging
LOG_LEVEL=info
APP_VERSION=1.0
```

UAT:

```text
APP_ENV=uat
LOG_LEVEL=warn
APP_VERSION=1.0
```

PROD:

```text
APP_ENV=production
LOG_LEVEL=warn
APP_VERSION=1.0
```

---

# 6. Create DEV ConfigMap

```cmd
oc create configmap lesson23-dev-config --from-literal=APP_ENV=development --from-literal=LOG_LEVEL=debug --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson23-dev-config
```

Describe:

```cmd
oc describe configmap lesson23-dev-config
```

---

# 7. Create QA ConfigMap

```cmd
oc create configmap lesson23-qa-config --from-literal=APP_ENV=qa --from-literal=LOG_LEVEL=info --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson23-qa-config
```

---

# 8. Create STAGE ConfigMap

```cmd
oc create configmap lesson23-stage-config --from-literal=APP_ENV=staging --from-literal=LOG_LEVEL=info --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson23-stage-config
```

---

# 9. Create UAT ConfigMap

```cmd
oc create configmap lesson23-uat-config --from-literal=APP_ENV=uat --from-literal=LOG_LEVEL=warn --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson23-uat-config
```

---

# 10. Create PROD ConfigMap

```cmd
oc create configmap lesson23-prod-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=warn --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson23-prod-config
```

---

# 11. Verify All ConfigMaps

```cmd
oc get configmap
```

Expected:

```text
lesson23-dev-config
lesson23-qa-config
lesson23-stage-config
lesson23-uat-config
lesson23-prod-config
```

---

# 12. DEV Deployment

Create:

```text
lesson23-dev.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson23-dev
  labels:
    app: lesson23
    environment: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lesson23
      environment: dev
  template:
    metadata:
      labels:
        app: lesson23
        environment: dev
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: lesson23-dev-config
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
```

Apply:

```cmd
oc apply -f lesson23-dev.yaml
```

---

# 13. Verify DEV

```cmd
oc get deployment lesson23-dev
```

```cmd
oc get pods -l environment=dev
```

Expected:

```text
1 Pod
```

Check environment variables:

```cmd
oc exec <pod-name> -- env
```

Look for:

```text
APP_ENV=development
LOG_LEVEL=debug
APP_VERSION=1.0
```

---

# 14. DEV Service

Create:

```text
lesson23-dev-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson23-dev-service
  labels:
    app: lesson23
    environment: dev
spec:
  selector:
    app: lesson23
    environment: dev
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson23-dev-service.yaml
```

---

# 15. DEV Route

Create:

```text
lesson23-dev-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson23-dev-route
spec:
  to:
    kind: Service
    name: lesson23-dev-service
  port:
    targetPort: http
```

Apply:

```cmd
oc apply -f lesson23-dev-route.yaml
```

Check:

```cmd
oc get route lesson23-dev-route
```

Open the Route URL in the browser.

---

# 16. QA Deployment

Create:

```text
lesson23-qa.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson23-qa
  labels:
    app: lesson23
    environment: qa
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson23
      environment: qa
  template:
    metadata:
      labels:
        app: lesson23
        environment: qa
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: lesson23-qa-config
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
```

Apply:

```cmd
oc apply -f lesson23-qa.yaml
```

---

# 17. QA Service

Create:

```text
lesson23-qa-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson23-qa-service
  labels:
    app: lesson23
    environment: qa
spec:
  selector:
    app: lesson23
    environment: qa
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson23-qa-service.yaml
```

---

# 18. QA Route

Create:

```text
lesson23-qa-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson23-qa-route
spec:
  to:
    kind: Service
    name: lesson23-qa-service
  port:
    targetPort: http
```

Apply:

```cmd
oc apply -f lesson23-qa-route.yaml
```

Check:

```cmd
oc get route lesson23-qa-route
```

---

# 19. Verify QA

```cmd
oc get deployment lesson23-qa
```

Expected:

```text
2/2
```

Check Pods:

```cmd
oc get pods -l environment=qa
```

Check environment variables:

```cmd
oc exec <pod-name> -- env
```

Expected:

```text
APP_ENV=qa
LOG_LEVEL=info
APP_VERSION=1.0
```

---

# 20. STAGE Deployment

Create:

```text
lesson23-stage.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson23-stage
  labels:
    app: lesson23
    environment: stage
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson23
      environment: stage
  template:
    metadata:
      labels:
        app: lesson23
        environment: stage
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: lesson23-stage-config
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
```

Apply:

```cmd
oc apply -f lesson23-stage.yaml
```

---

# 21. STAGE Service

Create:

```text
lesson23-stage-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson23-stage-service
  labels:
    app: lesson23
    environment: stage
spec:
  selector:
    app: lesson23
    environment: stage
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson23-stage-service.yaml
```

---

# 22. STAGE Route

Create:

```text
lesson23-stage-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson23-stage-route
spec:
  to:
    kind: Service
    name: lesson23-stage-service
  port:
    targetPort: http
```

Apply:

```cmd
oc apply -f lesson23-stage-route.yaml
```

Check:

```cmd
oc get route lesson23-stage-route
```

---

# 23. Verify STAGE

```cmd
oc get deployment lesson23-stage
```

```cmd
oc get pods -l environment=stage
```

Check environment:

```cmd
oc exec <pod-name> -- env
```

Expected:

```text
APP_ENV=staging
LOG_LEVEL=info
APP_VERSION=1.0
```

---

# 24. UAT Deployment

Create:

```text
lesson23-uat.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson23-uat
  labels:
    app: lesson23
    environment: uat
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson23
      environment: uat
  template:
    metadata:
      labels:
        app: lesson23
        environment: uat
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: lesson23-uat-config
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
```

Apply:

```cmd
oc apply -f lesson23-uat.yaml
```

---

# 25. UAT Service

Create:

```text
lesson23-uat-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson23-uat-service
  labels:
    app: lesson23
    environment: uat
spec:
  selector:
    app: lesson23
    environment: uat
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson23-uat-service.yaml
```

---

# 26. UAT Route

Create:

```text
lesson23-uat-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson23-uat-route
spec:
  to:
    kind: Service
    name: lesson23-uat-service
  port:
    targetPort: http
```

Apply:

```cmd
oc apply -f lesson23-uat-route.yaml
```

Check:

```cmd
oc get route lesson23-uat-route
```

---

# 27. Verify UAT

```cmd
oc get deployment lesson23-uat
```

```cmd
oc get pods -l environment=uat
```

Check environment:

```cmd
oc exec <pod-name> -- env
```

Expected:

```text
APP_ENV=uat
LOG_LEVEL=warn
APP_VERSION=1.0
```

---

# 28. PROD Deployment

Create:

```text
lesson23-prod.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson23-prod
  labels:
    app: lesson23
    environment: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lesson23
      environment: prod
  template:
    metadata:
      labels:
        app: lesson23
        environment: prod
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: lesson23-prod-config
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
```

Apply:

```cmd
oc apply -f lesson23-prod.yaml
```

---

# 29. PROD Service

Create:

```text
lesson23-prod-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson23-prod-service
  labels:
    app: lesson23
    environment: prod
spec:
  selector:
    app: lesson23
    environment: prod
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson23-prod-service.yaml
```

---

# 30. PROD Route

Create:

```text
lesson23-prod-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson23-prod-route
spec:
  to:
    kind: Service
    name: lesson23-prod-service
  port:
    targetPort: http
```

Apply:

```cmd
oc apply -f lesson23-prod-route.yaml
```

Check:

```cmd
oc get route lesson23-prod-route
```

---

# 31. Verify PROD

```cmd
oc get deployment lesson23-prod
```

Expected:

```text
3/3
```

Check:

```cmd
oc get pods -l environment=prod
```

Check environment:

```cmd
oc exec <pod-name> -- env
```

Expected:

```text
APP_ENV=production
LOG_LEVEL=warn
APP_VERSION=1.0
```

---

# 32. Verify All Deployments

```cmd
oc get deployment
```

Expected:

```text
lesson23-dev       1/1
lesson23-qa        2/2
lesson23-stage     2/2
lesson23-uat       2/2
lesson23-prod      3/3
```

---

# 33. Verify All Pods

```cmd
oc get pods -l app=lesson23
```

Expected:

```text
DEV    → 1 Pod
QA     → 2 Pods
STAGE  → 2 Pods
UAT    → 2 Pods
PROD   → 3 Pods
```

Total:

```text
10 Pods
```

---

# 34. Verify All Services

```cmd
oc get service
```

Expected:

```text
lesson23-dev-service
lesson23-qa-service
lesson23-stage-service
lesson23-uat-service
lesson23-prod-service
```

---

# 35. Verify All Routes

```cmd
oc get route
```

Expected:

```text
lesson23-dev-route
lesson23-qa-route
lesson23-stage-route
lesson23-uat-route
lesson23-prod-route
```

---

# 36. Why Service Selectors Matter

DEV Deployment:

```yaml
labels:
  app: lesson23
  environment: dev
```

DEV Service:

```yaml
selector:
  app: lesson23
  environment: dev
```

Therefore:

```text
DEV Service
     ↓
DEV Pods only
```

Similarly:

```text
QA Service
     ↓
QA Pods only
```

```text
STAGE Service
     ↓
STAGE Pods only
```

```text
UAT Service
     ↓
UAT Pods only
```

```text
PROD Service
     ↓
PROD Pods only
```

This prevents traffic from accidentally crossing between environments.

---

# 37. Environment Labels

We use:

```text
app=lesson23
environment=dev
```

Query DEV:

```cmd
oc get pods -l environment=dev
```

Query QA:

```cmd
oc get pods -l environment=qa
```

Query STAGE:

```cmd
oc get pods -l environment=stage
```

Query UAT:

```cmd
oc get pods -l environment=uat
```

Query PROD:

```cmd
oc get pods -l environment=prod
```

---

# 38. Environment-Specific Scaling

Our environments use:

```text
DEV    = 1
QA     = 2
STAGE  = 2
UAT    = 2
PROD   = 3
```

This demonstrates that resource requirements can differ between environments.

---

# 39. Environment-Specific Configuration

Our configuration is:

```text
DEV
APP_ENV=development
LOG_LEVEL=debug

QA
APP_ENV=qa
LOG_LEVEL=info

STAGE
APP_ENV=staging
LOG_LEVEL=info

UAT
APP_ENV=uat
LOG_LEVEL=warn

PROD
APP_ENV=production
LOG_LEVEL=warn
```

The application image remains the same.

---

# 40. Same Application / Different Configuration

This is an important DevOps principle:

```text
                    Application
                         |
              +----------+----------+
              |          |          |
              v          v          v
             DEV        QA         PROD
              |          |          |
          ConfigMap   ConfigMap   ConfigMap
              |          |          |
           replicas=1 replicas=2 replicas=3
```

We don't need to create a completely different application for every environment.

---

# 41. ConfigMap vs Secret

Use ConfigMap for non-sensitive values:

```text
APP_ENV
LOG_LEVEL
APP_VERSION
API_URL
```

Use Secret for sensitive values:

```text
DATABASE_PASSWORD
API_TOKEN
ACCESS_KEY
```

Never commit real passwords or tokens to Git.

---

# 42. Environment Promotion

A typical promotion flow is:

```text
DEV
 ↓
Testing
 ↓
QA
 ↓
Testing
 ↓
STAGE
 ↓
UAT
 ↓
Approval
 ↓
PROD
```

The application should ideally be promoted consistently.

---

# 43. Real-World Deployment Flow

A real CI/CD + GitOps workflow could look like:

```text
Developer
   ↓
Git Commit
   ↓
Build
   ↓
Automated Tests
   ↓
DEV
   ↓
QA
   ↓
STAGE
   ↓
UAT
   ↓
Approval
   ↓
PRODUCTION
```

This connects our previous lessons:

```text
Lesson 21
CI/CD
   ↓
Lesson 22
GitOps
   ↓
Lesson 23
Environment Management
```

---

# 44. Practice: Change DEV Replicas

Change:

```yaml
replicas: 1
```

to:

```yaml
replicas: 2
```

in:

```text
lesson23-dev.yaml
```

Apply:

```cmd
oc apply -f lesson23-dev.yaml
```

Verify:

```cmd
oc get deployment lesson23-dev
```

QA should remain:

```text
2 replicas
```

---

# 45. Practice: Change PROD Replicas

Change:

```yaml
replicas: 3
```

to:

```yaml
replicas: 4
```

in:

```text
lesson23-prod.yaml
```

Apply:

```cmd
oc apply -f lesson23-prod.yaml
```

Verify:

```cmd
oc get deployment lesson23-prod
```

Expected:

```text
4/4
```

---

# 46. Practice: Change Environment Configuration

Change DEV:

```text
LOG_LEVEL=debug
```

to:

```text
LOG_LEVEL=info
```

Update the ConfigMap:

```cmd
oc edit configmap lesson23-dev-config
```

After saving, restart the Deployment:

```cmd
oc rollout restart deployment/lesson23-dev
```

Check:

```cmd
oc get pods -l environment=dev
```

Then:

```cmd
oc exec <pod-name> -- env
```

Verify:

```text
LOG_LEVEL=info
```

---

# 47. Why Restart Is Required

We use:

```yaml
envFrom:
  - configMapRef:
      name: lesson23-dev-config
```

The ConfigMap values become container environment variables when the container starts.

Therefore, after changing the ConfigMap:

```text
ConfigMap changed
       ↓
Restart Pod
       ↓
Container starts
       ↓
New environment variables
```

---

# 48. Practice: Configuration Drift

Suppose Git says:

```text
DEV LOG_LEVEL=debug
```

but someone manually changes OpenShift to:

```text
DEV LOG_LEVEL=info
```

Now:

```text
Git Desired Configuration
          ≠
OpenShift Actual Configuration
```

This is configuration drift.

This connects directly to Lesson 22.

---

# 49. Git Repository Structure

A real environment-management repository could look like:

```text
lesson23-environments/
│
├── dev/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml
│   └── configmap.yaml
│
├── qa/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml
│   └── configmap.yaml
│
├── stage/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml
│   └── configmap.yaml
│
├── uat/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml
│   └── configmap.yaml
│
└── prod/
    ├── deployment.yaml
    ├── service.yaml
    ├── route.yaml
    └── configmap.yaml
```

For this lesson, we are keeping the files directly in one directory to make the concepts easier to understand.

---

# 50. Git Environment Management

Initialize Git:

```cmd
git init
```

Check:

```cmd
git status
```

Add:

```cmd
git add .
```

Commit:

```cmd
git commit -m "Add Lesson 23 environments"
```

Push:

```cmd
git push
```

Git now contains the environment configuration.

---

# 51. Git as Source of Truth

The concept becomes:

```text
Git
 ↓
Environment Configuration
 ↓
Desired State
 ↓
OpenShift
 ↓
Actual State
```

If the actual environment changes unexpectedly:

```text
Desired State
      ≠
Actual State
```

we have drift.

---

# 52. Troubleshooting

Check Pods:

```cmd
oc get pods
```

Describe a Pod:

```cmd
oc describe pod <pod-name>
```

Check logs:

```cmd
oc logs <pod-name>
```

Check Events:

```cmd
oc get events --sort-by=.lastTimestamp
```

Check Deployment:

```cmd
oc describe deployment <deployment-name>
```

Check rollout:

```cmd
oc rollout status deployment/<deployment-name>
```

---

# 53. NGINX Permission Issue Avoided

Previous lessons encountered errors such as:

```text
sed: couldn't open temporary file /etc/nginx/conf.d/...: Permission denied
```

and:

```text
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

The Lesson 23 Deployment therefore uses:

```text
nginxinc/nginx-unprivileged:1.25
```

with:

```text
containerPort: 8080
```

We do NOT use:

```text
sed
```

to modify protected NGINX files.

We do NOT depend on:

```text
/var/cache/nginx
```

being writable by root.

This makes the Deployment suitable for the restricted OpenShift Developer Sandbox environment.

---

# 54. Final Environment Architecture

```text
                         Git
                          |
                          v
              Environment Configuration
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v
       DEV               QA               STAGE
        |                 |                 |
   ConfigMap          ConfigMap          ConfigMap
        |                 |                 |
   Deployment         Deployment         Deployment
        |                 |                 |
      Service           Service           Service
        |                 |                 |
      Route             Route             Route
        |
        v
       UAT
        |
   ConfigMap
        |
   Deployment
        |
     Service
        |
      Route
        |
        v
      PROD
        |
   ConfigMap
        |
   Deployment
        |
     Service
        |
      Route
```

---

# 55. Final Environment Comparison

| Environment | Replicas | APP_ENV | LOG_LEVEL |
|---|---:|---|---|
| DEV | 1 | development | debug |
| QA | 2 | qa | info |
| STAGE | 2 | staging | info |
| UAT | 2 | uat | warn |
| PROD | 3 | production | warn |

---


# 🧠 Important Concepts

### Environment

A separate stage used for a particular purpose.

```text
DEV
QA
STAGE
UAT
PROD
```

### Environment-Specific Configuration

Values that change between environments.

```text
APP_ENV
LOG_LEVEL
API_URL
replicas
```

### Same Application

The application image can remain the same.

```text
nginxinc/nginx-unprivileged:1.25
```

### Different Configuration

Each environment can have different settings.

```text
DEV → debug
QA → info
PROD → warn
```

### ConfigMap

Used for non-sensitive configuration.

### Secret

Used for sensitive configuration.

### Promotion

Moving a tested version from:

```text
DEV → QA → STAGE → UAT → PROD
```

---

# 🔗 Connection With Previous Lessons

```text
Lesson 14
ConfigMaps / Secrets
        ↓
Lesson 18
Deployment Strategies
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
```

We are now combining several previous concepts.

---

# 🏁 Lesson 23 Goal


> **Environment management means running the same application across DEV, QA, STAGE, UAT and PROD while controlling environment-specific configuration separately. The application should remain consistent while values such as replica counts, logging levels, API endpoints and secrets can change according to the environment.**

The main pattern is:

```text
Same Application
        +
Environment-Specific Configuration
        ↓
DEV
 ↓
QA
 ↓
STAGE
 ↓
UAT
 ↓
PROD
```

Combined with GitOps:

```text
Git
 ↓
Environment Configuration
 ↓
Desired State
 ↓
OpenShift
 ↓
Actual State
 ↓
Drift
 ↓
Reconciliation
```

---

# ✅ Lesson 23 Completion Checklist

- [ ] Understand DEV
- [ ] Understand QA
- [ ] Understand STAGE
- [ ] Understand UAT
- [ ] Understand PROD
- [ ] Understand environment-specific configuration
- [ ] Understand same application / different configuration
- [ ] Create environment-specific ConfigMaps
- [ ] Create DEV Deployment
- [ ] Create DEV Service
- [ ] Create DEV Route
- [ ] Create QA Deployment
- [ ] Create QA Service
- [ ] Create QA Route
- [ ] Create STAGE Deployment
- [ ] Create STAGE Service
- [ ] Create STAGE Route
- [ ] Create UAT Deployment
- [ ] Create UAT Service
- [ ] Create UAT Route
- [ ] Create PROD Deployment
- [ ] Create PROD Service
- [ ] Create PROD Route
- [ ] Understand environment labels
- [ ] Understand Service selectors
- [ ] Configure different replica counts
- [ ] Configure different LOG_LEVEL values
- [ ] Verify environment variables inside Pods
- [ ] Understand ConfigMap vs Secret
- [ ] Understand environment promotion
- [ ] Understand configuration drift
- [ ] Connect environment management with GitOps
- [ ] Troubleshoot environment-specific Pods
- [ ] Understand why we simulated environments inside one Sandbox project
- [ ] Avoid NGINX non-root permission problems

---

# 🧠 Final Memory Trick

Remember:

```text
SAME APP
   +
DIFFERENT CONFIG
   =
MULTIPLE ENVIRONMENTS
```

And:

```text
DEV
 ↓
QA
 ↓
STAGE
 ↓
UAT
 ↓
PROD
```

With GitOps:

```text
Git
 ↓
Desired Environment
 ↓
OpenShift
 ↓
Actual Environment
 ↓
Compare
 ↓
Drift?
 ↓
Reconcile
```

**Lesson 23 Topic: Environment Management – DEV → QA → STAGE → UAT → PROD → Environment-Specific Configuration → Promotion → GitOps Integration**
