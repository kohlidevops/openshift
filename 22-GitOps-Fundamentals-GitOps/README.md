# OpenShift Learning – Lesson 22: GitOps Fundamentals

## 🚀 Lesson 22: GitOps – Git as Source of Truth → Desired State → Actual State → Drift → Reconciliation

> **Environment:** OpenShift Developer Sandbox  
> **Approach:** OpenShift-native GitOps fundamentals  
> **Git:** GitHub  
> **Deployment:** Declarative YAML  
> **GitOps Controller:** Argo CD concepts only  
> **Dockerfile:** Not required  
> **Cluster-admin access:** Not required

---

# 🎯 Learning Objectives

By the end of this lesson, I should understand:

- What GitOps means
- Declarative deployment
- Imperative vs declarative deployment
- Git as the source of truth
- Desired state
- Actual state
- Configuration drift
- Reconciliation
- Git-based application configuration
- OpenShift YAML manifests
- Deployment changes through Git
- Rollout and rollback
- GitOps pull model
- Argo CD fundamentals
- Why GitOps controllers reconcile desired and actual state

---

# 1. What Is GitOps?

GitOps is an approach where the desired configuration of an application or infrastructure is stored in Git.

The basic concept:

```text
Git
 ↓
Desired State
 ↓
OpenShift
 ↓
Actual State
```

The Git repository becomes the source of truth for the desired environment.

---

# 2. GitOps Architecture

Our practical exercise:

```text
GitHub Repository
       |
       | YAML
       v
Desired State
       |
       | oc apply
       v
OpenShift
       |
       v
Actual State
```

Later, a GitOps controller such as Argo CD can automate the reconciliation:

```text
GitHub
   |
   | Desired State
   v
Argo CD
   |
   | Compare
   v
OpenShift
   |
   | Actual State
   v
Drift Detection
   |
   v
Reconciliation
```

---

# 3. Declarative Deployment

Instead of telling OpenShift exactly how to perform every operation, we describe the desired final state.

Example:

```yaml
spec:
  replicas: 2
```

This means:

```text
I want 2 replicas.
```

OpenShift determines how to achieve that state.

---

# 4. Imperative vs Declarative

## Imperative

Example:

```cmd
oc scale deployment lesson22-app --replicas=3
```

You are directly telling OpenShift what action to perform.

---

## Declarative

The YAML contains:

```yaml
spec:
  replicas: 3
```

Then:

```cmd
oc apply -f lesson22-deployment.yaml
```

You are declaring the desired state.

GitOps is primarily based on the declarative approach.

---

# 5. Desired State vs Actual State

Suppose Git contains:

```yaml
replicas: 2
```

Then:

```text
Desired State = 2
```

If OpenShift currently has:

```text
Actual State = 2
```

then:

```text
Desired = 2
Actual  = 2
```

Everything is synchronized.

---

# 6. Configuration Drift

Suppose Git still contains:

```yaml
replicas: 2
```

but someone manually changes OpenShift:

```cmd
oc scale deployment lesson22-app --replicas=1
```

Now:

```text
Git Desired State = 2
OpenShift Actual State = 1
```

This difference is called:

```text
Configuration Drift
```

---

# 7. Reconciliation

If we apply the Git configuration again:

```cmd
oc apply -f lesson22-deployment.yaml
```

OpenShift returns to:

```text
Desired State = 2
Actual State  = 2
```

This process is:

```text
Reconciliation
```

A GitOps controller such as Argo CD can automate this process.

---

# 8. Create GitHub Repository

Create:

```text
lesson22-gitops
```

Recommended structure:

```text
lesson22-gitops/
│
├── lesson22-deployment.yaml
├── lesson22-service.yaml
└── lesson22-route.yaml
```

---

# 9. Create lesson22-deployment.yaml

> **Important:** We previously encountered an NGINX non-root permission problem with the standard `nginx:1.25` image in the Developer Sandbox.
>
> The previous approach attempted to modify `/etc/nginx/conf.d/default.conf` and use `/var/cache/nginx`, which resulted in `Permission denied`.
>
> Therefore, this updated lesson uses the **unprivileged NGINX image** and does not modify protected NGINX configuration files.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson22-app
  labels:
    app: lesson22-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson22-app
  template:
    metadata:
      labels:
        app: lesson22-app
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
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

### Why use `nginxinc/nginx-unprivileged`?

The Developer Sandbox uses restricted security contexts.

The standard NGINX image can attempt operations that require write/root permissions, such as:

```text
/etc/nginx/conf.d/
```

and:

```text
/var/cache/nginx/
```

This previously caused:

```text
Permission denied
```

The unprivileged NGINX image is designed to run without root privileges and listen on port `8080`.

Therefore, we don't need:

```text
sed
```

or modifications to:

```text
/etc/nginx/conf.d/default.conf
```

or:

```text
/var/cache/nginx
```

---

# 10. Create lesson22-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson22-service
  labels:
    app: lesson22-app
spec:
  selector:
    app: lesson22-app
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

The Service connects:

```text
Service
   ↓
Pods
```

---

# 11. Create lesson22-route.yaml

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson22-route
spec:
  to:
    kind: Service
    name: lesson22-service
  port:
    targetPort: http
```

The Route provides external access:

```text
Browser
   ↓
Route
   ↓
Service
   ↓
Pod
```

---

# 12. Validate the YAML

Before creating anything:

```cmd
oc apply --dry-run=server -f lesson22-deployment.yaml
```

Then:

```cmd
oc apply --dry-run=server -f lesson22-service.yaml
```

Then:

```cmd
oc apply --dry-run=server -f lesson22-route.yaml
```

Expected output should indicate that the resources are valid.

---

# 13. Push YAML to Git

Initialize Git:

```cmd
git init
```

Check:

```cmd
git status
```

Add files:

```cmd
git add .
```

Commit:

```cmd
git commit -m "Add Lesson 22 GitOps manifests"
```

Add your GitHub repository:

```cmd
git remote add origin <your-github-repository>
```

Push:

```cmd
git branch -M main
```

```cmd
git push -u origin main
```

Now GitHub contains the desired state.

---

# 14. Apply Deployment

```cmd
oc apply -f lesson22-deployment.yaml
```

Expected:

```text
deployment.apps/lesson22-app created
```

Check:

```cmd
oc get deployment
```

---

# 15. Apply Service

```cmd
oc apply -f lesson22-service.yaml
```

Expected:

```text
service/lesson22-service created
```

Check:

```cmd
oc get service
```

---

# 16. Apply Route

```cmd
oc apply -f lesson22-route.yaml
```

Expected:

```text
route.route.openshift.io/lesson22-route created
```

Check:

```cmd
oc get route
```

---

# 17. Check Pods

```cmd
oc get pods
```

Expected:

```text
lesson22-app-xxxxx   1/1   Running
lesson22-app-yyyyy   1/1   Running
```

We requested:

```yaml
replicas: 2
```

Therefore:

```text
Desired = 2
Actual  = 2
```

---

# 18. Check Deployment

```cmd
oc get deployment lesson22-app
```

You should see something similar to:

```text
NAME           READY   UP-TO-DATE   AVAILABLE
lesson22-app   2/2     2            2
```

---

# 19. Check Pod Logs

Find a Pod:

```cmd
oc get pods
```

Then:

```cmd
oc logs <pod-name>
```

The previous NGINX errors should NOT appear:

```text
sed: couldn't open temporary file ... Permission denied
```

or:

```text
mkdir() "/var/cache/nginx/client_temp" failed
```

---

# 20. Test the Route

Get the Route:

```cmd
oc get route lesson22-route
```

Copy the hostname.

Open it in your browser.

You should see the NGINX welcome page.

---

# 21. Check Complete Application Flow

```text
Browser
   ↓
OpenShift Route
   ↓
lesson22-service
   ↓
lesson22-app Deployment
   ↓
Pod 1
Pod 2
```

---

# 22. Git Is the Source of Truth

Our Git repository contains:

```text
lesson22-deployment.yaml
lesson22-service.yaml
lesson22-route.yaml
```

The Deployment contains:

```yaml
replicas: 2
```

Therefore:

```text
Git
 ↓
Desired State = 2
 ↓
OpenShift
 ↓
Actual State = 2
```

Everything is synchronized.

---

# 23. Practice Configuration Drift

Now intentionally change OpenShift manually:

```cmd
oc scale deployment lesson22-app --replicas=1
```

Check:

```cmd
oc get deployment lesson22-app
```

Check:

```cmd
oc get pods
```

Now:

```text
Git Desired State = 2
OpenShift Actual State = 1
```

This is configuration drift.

---

# 24. Understand the Drift

Git still contains:

```yaml
replicas: 2
```

But OpenShift was manually changed:

```text
replicas = 1
```

Therefore:

```text
             Git
              |
              v
       Desired = 2
              |
              X
              |
       Actual = 1
          OpenShift
```

---

# 25. Reconcile the Drift

Apply the Git configuration:

```cmd
oc apply -f lesson22-deployment.yaml
```

Check:

```cmd
oc get deployment lesson22-app
```

Then:

```cmd
oc get pods
```

You should return to:

```text
Desired = 2
Actual  = 2
```

This demonstrates reconciliation.

---

# 26. Important Difference: `oc scale` vs Git

When you run:

```cmd
oc scale deployment lesson22-app --replicas=1
```

you change:

```text
Actual Cluster State
```

You do NOT change:

```text
Git Desired State
```

Git still says:

```yaml
replicas: 2
```

Therefore:

```text
Git       = 2
OpenShift = 1
```

This is drift.

---

# 27. Change Desired State Through Git

Change:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

in:

```text
lesson22-deployment.yaml
```

Check:

```cmd
git diff
```

You should see:

```diff
- replicas: 2
+ replicas: 3
```

---

# 28. Commit the Desired State

```cmd
git add lesson22-deployment.yaml
```

```cmd
git commit -m "Scale Lesson 22 application to three replicas"
```

Push:

```cmd
git push
```

Now:

```text
Git Desired State = 3
OpenShift Actual State = 2
```

There is temporarily a difference.

---

# 29. Apply the New Desired State

For this manual GitOps simulation:

```cmd
oc apply -f lesson22-deployment.yaml
```

Check:

```cmd
oc get pods
```

You should now have:

```text
3 Pods
```

Therefore:

```text
Git Desired State = 3
OpenShift Actual State = 3
```

---

# 30. GitOps Reconciliation Concept

What we manually performed:

```text
Git
 ↓
Desired State
 ↓
Compare
 ↓
OpenShift
 ↓
Actual State
 ↓
Drift?
 ↓
Yes
 ↓
Reconcile
 ↓
Actual State = Desired State
```

A GitOps controller automates this process.

---

# 31. Change the Desired Image

Change:

```yaml
image: nginxinc/nginx-unprivileged:1.25
```

to:

```yaml
image: nginxinc/nginx-unprivileged:1.26
```

Check:

```cmd
git diff
```

Commit:

```cmd
git add lesson22-deployment.yaml
```

```cmd
git commit -m "Update Lesson 22 NGINX image"
```

Push:

```cmd
git push
```

Apply:

```cmd
oc apply -f lesson22-deployment.yaml
```

---

# 32. Verify the New Image

Run:

```cmd
oc get deployment lesson22-app -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Expected:

```text
nginxinc/nginx-unprivileged:1.26
```

Check rollout:

```cmd
oc rollout status deployment/lesson22-app
```

---

# 33. Check Deployment History

```cmd
oc rollout history deployment/lesson22-app
```

This shows deployment revisions.

---

# 34. Practice Rollback

If a deployment has a problem:

```cmd
oc rollout undo deployment/lesson22-app
```

Check:

```cmd
oc rollout status deployment/lesson22-app
```

Important:

A manual rollback changes the cluster, but does NOT automatically change Git.

Therefore:

```text
Git = Desired Configuration
Cluster = Actual Configuration
```

If Git still contains the newer configuration, rollback can create drift.

---

# 35. Reconcile After Rollback

Restore the Git-defined state:

```cmd
oc apply -f lesson22-deployment.yaml
```

Check:

```cmd
oc rollout status deployment/lesson22-app
```

This reinforces:

```text
Git
 ↓
Desired State
 ↓
OpenShift
 ↓
Actual State
```

---

# 36. Compare Git With OpenShift

Inspect the live Deployment:

```cmd
oc get deployment lesson22-app -o yaml
```

Compare it with:

```text
lesson22-deployment.yaml
```

This helps identify configuration drift.

---

# 37. Git History

Run:

```cmd
git log --oneline
```

You should see commits similar to:

```text
Update Lesson 22 NGINX image
Scale Lesson 22 application to three replicas
Add Lesson 22 GitOps manifests
```

Git now provides a history of desired-state changes.

---

# 38. Why Git Is Useful as Source of Truth

Git provides:

```text
Version history
Change tracking
Audit trail
Code review
Rollback capability
Collaboration
Desired-state storage
```

Example:

```text
Commit 1
replicas: 2

        ↓

Commit 2
replicas: 3

        ↓

Commit 3
image: nginxinc/nginx-unprivileged:1.26
```

---

# 39. Argo CD Concept

Argo CD is a GitOps continuous delivery tool commonly used with Kubernetes and OpenShift.

Conceptually:

```text
GitHub
   |
   | Desired State
   v
Argo CD
   |
   | Compare
   v
OpenShift
   |
   | Actual State
   v
Drift Detection
```

Argo CD can determine whether the cluster matches the Git repository.

---

# 40. Without Argo CD

Our current exercise uses:

```text
Git
 ↓
oc apply
 ↓
OpenShift
```

We manually perform reconciliation.

---

# 41. With Argo CD

A GitOps controller can perform reconciliation:

```text
Git
 ↓
Argo CD
 ↓
Detect Difference
 ↓
Sync
 ↓
OpenShift
```

This is the major difference.

---

# 42. GitOps Pull Model

Traditional CI/CD can look like:

```text
CI/CD Server
      ↓
Push deployment
      ↓
OpenShift
```

GitOps commonly uses:

```text
Git
 ↓
GitOps Controller
 ↓
OpenShift
```

The GitOps controller observes the Git repository and the cluster and works to keep the cluster synchronized with the desired state.

---

# 43. Developer Sandbox Limitation

We will NOT make Argo CD installation a requirement for Lesson 22.

Earlier, the Developer Sandbox returned:

```text
Forbidden
```

when attempting cluster-scoped operator operations.

Therefore, Lesson 22 focuses on concepts that work directly within our project namespace:

```text
Git
+
YAML
+
OpenShift
+
Desired State
+
Actual State
+
Drift
+
Reconciliation
```

If Argo CD is already available in the environment, it can be explored separately.

---

# 44. Final GitOps Exercise

Start with:

```yaml
replicas: 2
```

Apply:

```cmd
oc apply -f lesson22-deployment.yaml
```

Verify:

```cmd
oc get pods
```

Then manually change OpenShift:

```cmd
oc scale deployment lesson22-app --replicas=1
```

Now:

```text
Git = 2
OpenShift = 1
```

Identify the drift.

Reconcile:

```cmd
oc apply -f lesson22-deployment.yaml
```

Now:

```text
Git = 2
OpenShift = 2
```

---


# 🧠 Final Memory Trick

Remember:

```text
G → D → O → A → R
```

Where:

```text
G = Git
D = Desired State
O = OpenShift
A = Actual State
R = Reconciliation
```

The complete concept:

```text
             Git
              |
              v
       Desired State
              |
              v
        GitOps Controller
              |
              v
          OpenShift
              |
              v
         Actual State
              |
              v
        Compare States
              |
        +-----+-----+
        |           |
      Same        Different
        |           |
        v           v
   In Sync       Drift
                    |
                    v
              Reconciliation
                    |
                    v
              Actual State
              matches Git
```

---

# 🔧 Important Commands

## Git

```cmd
git status
```

```cmd
git diff
```

```cmd
git add .
```

```cmd
git commit -m "Update desired state"
```

```cmd
git push
```

```cmd
git log --oneline
```

---

## Validate YAML

```cmd
oc apply --dry-run=server -f lesson22-deployment.yaml
```

---

## Apply YAML

```cmd
oc apply -f lesson22-deployment.yaml
```

```cmd
oc apply -f lesson22-service.yaml
```

```cmd
oc apply -f lesson22-route.yaml
```

---

## Deployment

```cmd
oc get deployment
```

```cmd
oc describe deployment lesson22-app
```

```cmd
oc rollout status deployment/lesson22-app
```

```cmd
oc rollout history deployment/lesson22-app
```

```cmd
oc rollout undo deployment/lesson22-app
```

---

## Pods

```cmd
oc get pods
```

```cmd
oc describe pod <pod-name>
```

```cmd
oc logs <pod-name>
```

---

## Service

```cmd
oc get service
```

```cmd
oc describe service lesson22-service
```

---

## Route

```cmd
oc get route
```

```cmd
oc get route lesson22-route
```

---

## Check Image

```cmd
oc get deployment lesson22-app -o jsonpath="{.spec.template.spec.containers[0].image}"
```

---

## Create Drift

```cmd
oc scale deployment lesson22-app --replicas=1
```

Restore desired state:

```cmd
oc apply -f lesson22-deployment.yaml
```

---

# ⚠️ Important NGINX Lesson 22 Issue

The original Deployment used:

```text
nginx:1.25
```

with commands that modified NGINX configuration.

In the OpenShift Developer Sandbox this caused:

```text
sed: couldn't open temporary file /etc/nginx/conf.d/...: Permission denied
```

and:

```text
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

The corrected Deployment uses:

```text
nginxinc/nginx-unprivileged:1.25
```

and:

```text
containerPort: 8080
```

No `sed` command is required.

No modification of:

```text
/etc/nginx/conf.d/
```

is required.

No modification of:

```text
/var/cache/nginx/
```

is required.

This corrected YAML should be used for the Lesson 22 GitOps exercises.

---

# ⚠️ Developer Sandbox GitHub Webhook Limitation

This lesson does NOT depend on GitHub webhooks.

In Lesson 21 we confirmed that:

```cmd
oc auth can-i create buildconfigs/webhooks --as=system:unauthenticated -n lakshminarayananredh-dev
```

returned:

```text
no
```

and the GitHub webhook returned:

```text
403 Forbidden
```

That is a Developer Sandbox permission limitation.

Lesson 22 does not require that webhook.

Our GitOps fundamentals exercise uses:

```text
GitHub
   ↓
YAML
   ↓
oc apply
   ↓
OpenShift
```

The important concepts are still fully demonstrated.

---

# ✅ Lesson 22 Completion Checklist

- [ ] Understand GitOps
- [ ] Understand declarative configuration
- [ ] Understand imperative commands
- [ ] Understand desired state
- [ ] Understand actual state
- [ ] Understand Git as source of truth
- [ ] Create Deployment YAML
- [ ] Create Service YAML
- [ ] Create Route YAML
- [ ] Understand why the unprivileged NGINX image is used
- [ ] Avoid NGINX non-root permission problems
- [ ] Validate YAML using `--dry-run=server`
- [ ] Store YAML in Git
- [ ] Push YAML to GitHub
- [ ] Deploy YAML to OpenShift
- [ ] Verify Pods
- [ ] Verify Deployment
- [ ] Verify Service
- [ ] Verify Route
- [ ] Access application
- [ ] Create configuration drift
- [ ] Identify drift
- [ ] Reconcile drift
- [ ] Change desired replica count
- [ ] Change desired image
- [ ] Practice deployment rollout
- [ ] Practice rollback
- [ ] Understand why rollback can create Git drift
- [ ] Understand Git history
- [ ] Understand reconciliation
- [ ] Understand GitOps pull model
- [ ] Understand Argo CD concept
- [ ] Understand why Argo CD installation is not required for this Sandbox lesson

---

# 🏁 Lesson 22 Goal


> **GitOps is a declarative approach where Git stores the desired state of an application or infrastructure. OpenShift represents the actual state. When the desired and actual states differ, configuration drift occurs. A GitOps controller such as Argo CD detects the difference and reconciles the OpenShift environment toward the state defined in Git.**

For this Developer Sandbox, the practical workflow is:

```text
GitHub
   ↓
YAML
   ↓
Desired State
   ↓
oc apply
   ↓
OpenShift
   ↓
Actual State
   ↓
Create Drift
   ↓
Detect Drift
   ↓
Reconcile
   ↓
Desired = Actual
```

Final architecture:

```text
                    GitHub
                       |
                       v
                Desired State
                       |
                       v
              GitOps Configuration
                       |
                       v
                 OpenShift
                       |
                       v
                 Actual State
                       |
                       v
              Compare / Reconcile
                       |
                       v
                  Application
```

**Lesson 22 Topic: GitOps Fundamentals – Git as Source of Truth → Declarative Deployment → Desired State → Actual State → Configuration Drift → Reconciliation → Argo CD Concepts**
