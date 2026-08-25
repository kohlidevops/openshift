# OpenShift Learning – Lesson 15: SCC & Container Security

## 🚀 Lesson 15: OpenShift Security – SCC & Container Security

In this lesson, I learned how OpenShift secures containers using **Security Context Constraints (SCC)** and Kubernetes/OpenShift `securityContext`.

> **Note:** ServiceAccounts, Users, Roles and RBAC were already covered in **Lesson 2**, so they are intentionally not repeated here.

---

## 🎯 Learning Objectives

- Understand Security Context Constraints (SCC)
- Understand root vs non-root containers
- Understand `runAsNonRoot`
- Understand `runAsUser`
- Understand `fsGroup`
- Understand `allowPrivilegeEscalation`
- Understand Linux capabilities
- Understand privileged containers
- Understand Pod `securityContext`
- Understand Container `securityContext`
- Understand filesystem permissions
- Troubleshoot non-root container failures
- Understand why some container images fail when running as non-root
- Understand Pod immutability when changing container configuration
- Apply container security best practices

---

# 1. What is SCC?

SCC means:

```text
Security Context Constraints
```

SCC is an OpenShift security mechanism that controls what a Pod is allowed to do.

Examples:

```text
Can the container run as root?
Can it use privileged mode?
Can it use host networking?
Can it use host filesystem?
Can it use Linux capabilities?
Can it escalate privileges?
```

Conceptually:

```text
Pod
 |
 v
SCC
 |
 v
Security Restrictions
 |
 v
Container
```

---

# 2. Why Does OpenShift Use SCC?

A container with excessive privileges can create a security risk.

For example:

```text
Privileged Container
        |
        v
Greater access to host resources
        |
        v
Higher security risk
```

OpenShift follows the principle of:

```text
Least Privilege
```

Only the permissions required by the workload should be provided.

---

# 3. Root vs Non-Root Containers

A root container normally runs with:

```text
UID 0
```

A non-root container runs with another UID.

Example:

```text
Root:
UID 0

Non-root:
UID != 0
```

For production workloads, running as non-root is generally preferred when the application supports it.

---

# 4. Check the User Running Inside a Container

Create a simple Pod:

```powershell
oc run lesson15-user --image=busybox:1.36 --command -- sleep 3600
```

Check:

```powershell
oc get pod lesson15-user
```

Check the UID:

```powershell
oc exec lesson15-user -- id
```

The important thing to observe is whether the process is running as:

```text
uid=0
```

or a non-zero UID.

Clean up:

```powershell
oc delete pod lesson15-user
```

---

# 5. What is SecurityContext?

`securityContext` defines security-related settings for a Pod or container.

Example:

```yaml
securityContext:
  runAsNonRoot: true
```

Conceptually:

```text
Pod
 |
 +---- SecurityContext
 |
 +---- Container
```

---

# 6. `runAsNonRoot`

Example:

```yaml
securityContext:
  runAsNonRoot: true
```

This tells OpenShift/Kubernetes that the container must not run as UID 0.

Conceptually:

```text
runAsNonRoot: true
        |
        v
Container cannot run as root
```

---

# 7. Practice: Non-Root Container

Initially, we created a Pod using:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson15-nonroot
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: app
      image: nginx:latest
```

The Pod entered:

```text
0/1     CrashLoopBackOff
```

This was an important troubleshooting exercise.

---

# 8. Troubleshooting the CrashLoopBackOff

We checked:

```powershell
oc logs lesson15-nonroot
```

The important errors were:

```text
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

and:

```text
the "user" directive makes sense only if the master process runs with super-user privileges
```

The important error was:

```text
Permission denied
```

---

# 9. Why Did NGINX Fail?

The Pod was configured with:

```yaml
runAsNonRoot: true
```

Therefore, NGINX was not running as root.

The standard:

```text
nginx:latest
```

image expected to write to:

```text
/var/cache/nginx/client_temp
```

The non-root user did not have permission to create that directory.

The result was:

```text
runAsNonRoot
      |
      v
NGINX runs as non-root
      |
      v
NGINX needs /var/cache/nginx
      |
      v
Permission denied
      |
      v
NGINX exits
      |
      v
Container restarts
      |
      v
CrashLoopBackOff
```

---

# 10. Important Lesson From the NGINX Problem

This is an important real-world container security concept:

> An application can work as root but fail as a non-root user because of filesystem permissions.

Therefore, when converting an application to run as non-root, we must verify:

```text
Application user
Filesystem permissions
Application configuration
Writable directories
Temporary directories
Log directories
Cache directories
Mounted volumes
```

---

# 11. Use a Non-Root-Friendly Practice Container

Instead of continuing with the standard NGINX image, use BusyBox for the basic `runAsNonRoot` practice.

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson15-nonroot
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo Running as non-root; id; sleep 3600"]
```

Save as:

```text
lesson15-nonroot.yaml
```

---

# 12. Important Pod Immutability Issue

When we tried to apply the updated YAML to the existing Pod:

```powershell
oc apply -f lesson15-nonroot.yaml
```

OpenShift returned:

```text
The Pod "lesson15-nonroot" is invalid:
spec: Forbidden: pod updates may not change fields other than ...
```

The reason was that we attempted to change the container command of an existing Pod.

Most Pod specification fields are immutable after creation.

---

# 13. Correct Way to Change the Pod

Delete the existing Pod:

```powershell
oc delete pod lesson15-nonroot
```

Then recreate it:

```powershell
oc apply -f lesson15-nonroot.yaml
```

Check:

```powershell
oc get pod lesson15-nonroot
```

Check logs:

```powershell
oc logs lesson15-nonroot
```

Check the UID:

```powershell
oc exec lesson15-nonroot -- id
```

Expected concept:

```text
Container
    |
    v
Non-zero UID
    |
    v
Non-root execution
```

---

# 14. Pod Immutability – Important Concept

Remember:

```text
Pod created
    |
    v
Most Pod spec fields cannot be changed
    |
    v
Delete and recreate the Pod
```

For application workloads, normally use a Deployment:

```text
Deployment
    |
    v
ReplicaSet
    |
    v
Pods
```

When the Deployment is updated, OpenShift creates new Pods automatically.

Therefore:

```text
Pod
→ Usually recreate for major spec changes

Deployment
→ Update Deployment and let it perform a rollout
```

---

# 15. `runAsUser`

You can specify a UID:

```yaml
securityContext:
  runAsUser: 1001
```

This tells the container to run using that UID.

However, do not randomly choose UIDs in OpenShift.

OpenShift projects may have namespace-specific UID ranges.

Therefore:

```text
Do not blindly hardcode UID 1001
```

unless you understand the image and OpenShift UID configuration.

---

# 16. `fsGroup`

`fsGroup` is commonly used to help manage permissions for mounted volumes.

Example:

```yaml
securityContext:
  fsGroup: 1001
```

Conceptually:

```text
Persistent Volume
       |
       v
Filesystem Permissions
       |
       v
fsGroup
       |
       v
Container Access
```

This becomes especially important when an application needs to write to a PVC.

---

# 17. `allowPrivilegeEscalation`

Example:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

This prevents a process from gaining additional privileges through privilege escalation mechanisms.

For applications that support it:

```text
allowPrivilegeEscalation: false
```

is generally a useful security setting.

---

# 18. Linux Capabilities

Linux capabilities divide privileged operations into smaller permissions.

Instead of giving broad privileges, individual capabilities can be controlled.

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

This removes unnecessary Linux capabilities from the container.

Conceptually:

```text
Container
   |
   +---- Capability A
   +---- Capability B
   +---- Capability C
```

Instead of granting everything:

```text
Full Privileges
```

we follow:

```text
Only Required Capabilities
```

---

# 19. Practice: Drop Capabilities

Create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson15-capabilities
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          drop:
            - ALL
```

Apply:

```powershell
oc apply -f lesson15-capabilities.yaml
```

Check:

```powershell
oc get pod lesson15-capabilities
```

Inspect:

```powershell
oc describe pod lesson15-capabilities
```

---

# 20. Privileged Containers

A privileged container has significantly greater access to host-level capabilities.

Normal container:

```text
Restricted Access
```

Privileged container:

```text
Greater Host Access
       |
       v
Higher Security Risk
```

Avoid privileged containers unless there is a specific and justified requirement.

---

# 21. Do Not Try to Enable Privileged Mode

For this lesson, do not attempt to bypass OpenShift security restrictions.

Your Developer Sandbox user is restricted.

Commands such as:

```powershell
oc get scc
```

may return:

```text
Forbidden
```

This is expected.

The objective is to **understand container security**, not bypass OpenShift security controls.

---

# 22. Pod SecurityContext vs Container SecurityContext

### Pod-level SecurityContext

```yaml
spec:
  securityContext:
```

Example:

```yaml
spec:
  securityContext:
    runAsNonRoot: true
```

### Container-level SecurityContext

```yaml
containers:
  - name: app
    securityContext:
```

Example:

```yaml
containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
```

Conceptually:

```text
Pod
 |
 +---- Pod SecurityContext
 |
 +---- Container A
 |       |
 |       +---- Container SecurityContext
 |
 +---- Container B
         |
         +---- Container SecurityContext
```

---

# 23. Read-Only Root Filesystem

A container can optionally use:

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

This makes the root filesystem read-only.

Conceptually:

```text
Application
    |
    +---- Read files
    |
    X---- Cannot modify root filesystem
```

The application must be designed to work with this setting.

---

# 24. Secure Container Example

A secure container configuration can look like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson15-secure
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
```

Apply:

```powershell
oc apply -f lesson15-secure.yaml
```

Check:

```powershell
oc get pod lesson15-secure
```

Check UID:

```powershell
oc exec lesson15-secure -- id
```

Inspect configuration:

```powershell
oc get pod lesson15-secure -o yaml
```

---

# 25. Security Troubleshooting Flow

When a secure Pod fails:

```text
Pod Failed
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
Check SecurityContext
    |
    v
Check Container Image
    |
    v
Check Filesystem Permissions
    |
    v
Check User/UID Requirements
```

---

# 26. Common Security Problem

Example:

```text
Application
     |
     v
Writes to /app/data
     |
     v
Directory owned by root
     |
     v
Container runs as non-root
     |
     v
Permission denied
```

Troubleshooting:

```powershell
oc logs <pod-name>
```

Look for:

```text
Permission denied
```

Then check:

```powershell
oc exec <pod-name> -- id
```

and:

```powershell
oc exec <pod-name> -- ls -ld /app/data
```

---

# 27. Container Security Best Practices

For production containers:

```text
[ ] Run as non-root
[ ] Avoid privileged mode
[ ] Disable privilege escalation when possible
[ ] Drop unnecessary capabilities
[ ] Use minimal base images
[ ] Keep images updated
[ ] Never store secrets inside images
[ ] Use OpenShift Secrets for credentials
[ ] Use read-only filesystem when possible
[ ] Set appropriate resource limits
[ ] Follow least privilege
[ ] Verify filesystem permissions
```

---

# 28. Hands-On Challenge

Create a Pod named:

```text
lesson15-final
```

Requirements:

```text
Non-root
No privilege escalation
Drop all capabilities
BusyBox image
Keep container running
```

Verify:

```powershell
oc get pod lesson15-final
```

Check UID:

```powershell
oc exec lesson15-final -- id
```

Check security configuration:

```powershell
oc get pod lesson15-final -o yaml
```

Check Events:

```powershell
oc get events --sort-by=.lastTimestamp
```

---

# 29. Final Troubleshooting Challenge

For `lesson15-final`, answer:

### What UID is the container using?

```powershell
oc exec lesson15-final -- id
```

### Is the container running?

```powershell
oc get pod lesson15-final
```

### Is privilege escalation disabled?

```powershell
oc get pod lesson15-final -o yaml
```

### Are capabilities being dropped?

```powershell
oc get pod lesson15-final -o yaml
```

### Are there any security-related Events?

```powershell
oc get events --sort-by=.lastTimestamp
```

---

# 30. Important Commands

Check Pods:

```powershell
oc get pods
```

Describe Pod:

```powershell
oc describe pod <pod-name>
```

Check logs:

```powershell
oc logs <pod-name>
```

Check previous logs:

```powershell
oc logs <pod-name> --previous
```

Check container UID:

```powershell
oc exec <pod-name> -- id
```

Check filesystem permissions:

```powershell
oc exec <pod-name> -- ls -ld <directory>
```

Check security configuration:

```powershell
oc get pod <pod-name> -o yaml
```

Check Events:

```powershell
oc get events --sort-by=.lastTimestamp
```

Check SCC:

```powershell
oc get scc
```

> `oc get scc` may return `Forbidden` in the Developer Sandbox because SCC is cluster-scoped. This is expected.

---

# 🧠 Final Memory Trick

```text
SCC
 ↓
OpenShift Pod security restrictions

SecurityContext
 ↓
Security configuration for Pod/container

runAsNonRoot
 ↓
Don't run as root

runAsUser
 ↓
Specify container UID

fsGroup
 ↓
Help manage mounted-volume permissions

allowPrivilegeEscalation
 ↓
Control privilege escalation

Capabilities
 ↓
Control Linux privileges

Privileged
 ↓
Greater host access
```

Remember:

```text
RBAC → What can the identity do?

SCC → What can the Pod/container do?

SecurityContext → How should the container run?
```

---

# 📝 Important Issues Faced During Lesson 15

## Issue 1 – NGINX CrashLoopBackOff

Initial Pod:

```text
lesson15-nonroot   0/1   CrashLoopBackOff
```

We checked:

```powershell
oc logs lesson15-nonroot
```

The important error was:

```text
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

### Root Cause

The Pod was configured with:

```yaml
runAsNonRoot: true
```

The standard `nginx:latest` image attempted to write to a directory where the non-root user did not have sufficient permissions.

### Lesson Learned

```text
Non-root container
       |
       v
Application must support non-root execution
       |
       v
Filesystem permissions become important
```

---

## Issue 2 – Pod Update Rejected

After changing the container command, we ran:

```powershell
oc apply -f lesson15-nonroot.yaml
```

OpenShift returned:

```text
The Pod "lesson15-nonroot" is invalid:
spec: Forbidden: pod updates may not change fields other than ...
```

### Root Cause

Most Pod specification fields are immutable after the Pod has been created.

### Correct Solution

Delete the existing Pod:

```powershell
oc delete pod lesson15-nonroot
```

Then recreate it:

```powershell
oc apply -f lesson15-nonroot.yaml
```

### Lesson Learned

```text
Pod
 ↓
Created
 ↓
Most spec fields are immutable
 ↓
Delete + recreate
```

For normal application deployments, use a Deployment so that OpenShift can manage Pod replacement through ReplicaSets and rolling updates.

---

# ✅ Lesson 15 Completion Checklist

- [ ] Understand SCC
- [ ] Understand why OpenShift uses SCC
- [ ] Understand root vs non-root containers
- [ ] Check the UID inside a container
- [ ] Understand `runAsNonRoot`
- [ ] Understand `runAsUser`
- [ ] Understand `fsGroup`
- [ ] Understand `allowPrivilegeEscalation`
- [ ] Understand Linux capabilities
- [ ] Practice dropping capabilities
- [ ] Understand privileged containers
- [ ] Understand Pod SecurityContext
- [ ] Understand Container SecurityContext
- [ ] Understand `readOnlyRootFilesystem`
- [ ] Practice a secure Pod configuration
- [ ] Troubleshoot non-root container failures
- [ ] Understand filesystem permission problems
- [ ] Understand Pod immutability
- [ ] Understand why a Pod may need to be recreated
- [ ] Understand OpenShift least-privilege security
- [ ] Understand why `oc get scc` may return `Forbidden`

---

# 🏁 Lesson 15 Goal


> **OpenShift uses SCC and container security settings to prevent workloads from running with unnecessary privileges. A secure container should normally run as non-root, avoid privilege escalation, use only required Linux capabilities, and have appropriate filesystem permissions.**

The main security model is:

```text
                 OpenShift Container Security
                            |
              +-------------+-------------+
              |                           |
             SCC                    SecurityContext
              |                           |
      OpenShift Security           Container Settings
              |                           |
              +-------------+-------------+
                            |
                            v
                         Container
                            |
                            v
                    Least Privilege
```

**Lesson 15 Topic: OpenShift Security – SCC & Container Security**
