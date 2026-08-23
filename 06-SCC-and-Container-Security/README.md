# OpenShift Learning – Lesson 6

## Security Context Constraints (SCC) & Container Security

In this lesson, I learned how OpenShift applies security controls to application workloads using **Security Context Constraints (SCC)** and Kubernetes `securityContext`.

I performed hands-on practice using the **Red Hat OpenShift Developer Sandbox**, including running containers as non-root users, checking UID/GID, configuring security contexts, restricting privilege escalation, dropping Linux capabilities, understanding ServiceAccounts, identifying the SCC applied to Pods, and troubleshooting container permission issues.

---

# 🎯 Learning Objectives

- Understand Security Context Constraints (SCC)
- Understand why OpenShift uses SCC
- Understand root vs non-root containers
- Understand Linux UID/GID
- Understand OpenShift's random/arbitrary UID concept
- Understand Kubernetes `securityContext`
- Understand `runAsNonRoot`
- Understand `runAsUser`
- Understand `allowPrivilegeEscalation`
- Understand Linux capabilities
- Understand `privileged`
- Understand `anyuid`
- Understand ServiceAccounts
- Identify the SCC applied to a Pod
- Understand restricted SCC behavior
- Troubleshoot container permission errors
- Understand why some Docker images fail on OpenShift
- Understand OpenShift-compatible container image design

---

# 1. Why Container Security Matters

A container can potentially run processes with high privileges.

A traditional container might run as:

```text
root
 |
 +-- UID 0
```

Running applications with unnecessary privileges increases the potential impact if an application is compromised.

OpenShift applies additional security controls to help prevent workloads from unnecessarily running with elevated privileges.

Conceptually:

```text
Application
     |
     ▼
Container
     |
     ▼
OpenShift Security Controls
     |
     ▼
Restricted / Non-root Execution
```

---

# 2. What is SCC?

SCC stands for:

> **Security Context Constraints**

SCC is an OpenShift security mechanism that controls what a Pod or container is allowed to do.

SCC can control things such as:

- Running as root
- Running in privileged mode
- Privilege escalation
- Linux capabilities
- UID ranges
- SELinux context
- Host networking
- Host PID namespace
- Volume types

Conceptually:

```text
Pod
 |
 ▼
Security Context
 |
 ▼
OpenShift SCC
 |
 ▼
Security Policy
 |
 ▼
Container
```

---

# 3. Kubernetes SecurityContext vs OpenShift SCC

These are related but different concepts.

### Kubernetes

A Pod can define:

```yaml
securityContext:
  runAsNonRoot: true
```

or:

```yaml
securityContext:
  runAsUser: 1000
```

### OpenShift

OpenShift additionally applies:

```text
SCC
```

to determine what security configuration is permitted.

Architecture:

```text
Pod YAML
   |
   | securityContext
   ▼
OpenShift SCC
   |
   ▼
Security rules
   |
   +---- Allowed ----> Pod runs
   |
   +---- Not allowed -> Pod rejected/modified
```

---

# 4. Check Available SCCs

List SCCs:

```bash
oc get scc
```

Depending on the OpenShift environment, SCCs may include:

```text
anyuid
hostaccess
hostmount-anyuid
hostnetwork
node-exporter
nonroot
nonroot-v2
privileged
restricted
restricted-v2
```

The exact list can vary between OpenShift environments.

---

# 5. Inspect `restricted-v2`

Inspect the restricted SCC:

```bash
oc describe scc restricted-v2
```

Important fields include:

```text
Allow Privileged
Allow Privilege Escalation
Run As User Strategy
UID
SELinux Context Strategy
FSGroup Strategy
```

The `restricted-v2` SCC is designed to enforce a restricted security posture for application workloads.

---

# 6. Inspect `restricted`

The older restricted SCC can also be inspected:

```bash
oc describe scc restricted
```

Modern OpenShift environments commonly use newer restricted SCC behavior such as:

```text
restricted-v2
```

for workloads.

---

# 7. Identify the SCC Used by a Pod

First list Pods:

```bash
oc get pods
```

Choose a running Pod.

Then inspect the Pod:

```bash
oc get pod <pod-name> -o yaml
```

Look for an annotation similar to:

```yaml
openshift.io/scc: restricted-v2
```

This identifies the SCC applied to the Pod.

---

# 8. Check SCC Using JSONPath

The SCC can be retrieved directly:

```bash
oc get pod <pod-name> -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

Example:

```bash
oc get pod lesson3-web-57ff9c4cf-n6p7x \
  -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

Possible result:

```text
restricted-v2
```

---

# 9. Root vs Non-root

Linux represents users using UIDs.

The root user is:

```text
UID 0
```

Example:

```text
root
 |
 +-- UID 0
```

A non-root user may have:

```text
UID 1000
```

or another UID.

OpenShift commonly runs application containers with non-root or restricted identities.

Conceptually:

```text
Container
    |
    +-- UID 0
    |     |
    |     X Root
    |
    +-- Non-root UID
          |
          ✓ Restricted execution
```

---

# 10. Why OpenShift Uses Non-root Containers

Running applications as non-root reduces the privileges available to the application.

Instead of:

```text
Application
    |
    ▼
Root
    |
    ▼
High privileges
```

OpenShift encourages:

```text
Application
    |
    ▼
Non-root UID
    |
    ▼
Restricted privileges
```

This helps implement the principle of least privilege.

---

# 11. Check the Current UID

Create:

### `lesson6-user.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson6-user
spec:
  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      command:
        - sh
        - -c
        - |
          echo "Current user:"
          id
          echo "UID:"
          id -u
          echo "GID:"
          id -g
          sleep 3600
```

Apply:

```bash
oc apply -f lesson6-user.yaml
```

Check:

```bash
oc get pod lesson6-user
```

Expected:

```text
NAME           READY   STATUS
lesson6-user   1/1     Running
```

View logs:

```bash
oc logs lesson6-user
```

---

# 12. Check UID Directly

Run:

```bash
oc exec lesson6-user -- id
```

Or:

```bash
oc exec lesson6-user -- id -u
```

Check GID:

```bash
oc exec lesson6-user -- id -g
```

The important observation is that the application is not necessarily running as:

```text
UID 0
```

---

# 13. Test Filesystem Permissions

A restricted non-root container should not assume that protected filesystem locations are writable.

For example:

```bash
oc exec lesson6-user -- sh -c "touch /test-root-file"
```

Depending on the image and permissions, this can result in:

```text
Permission denied
```

This demonstrates:

> An application running as a non-root user cannot automatically write to every directory inside the container filesystem.

---

# 14. The Lesson 3 NGINX Security Problem

In Lesson 3, the initial NGINX container entered:

```text
CrashLoopBackOff
```

The logs showed:

```text
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

The container expected to write to:

```text
/var/cache/nginx
```

but the OpenShift security model prevented the container from writing there.

The simplified flow was:

```text
NGINX
  |
  ▼
Attempts to write
/var/cache/nginx
  |
  ▼
Permission denied
  |
  ▼
NGINX exits
  |
  ▼
CrashLoopBackOff
```

---

# 15. Why the NGINX Image Failed

The original container image expected behavior similar to:

```text
root
 |
 +-- Can write protected directories
```

OpenShift instead applied restricted security:

```text
non-root UID
 |
 +-- Cannot write protected directories
```

Therefore:

```text
Permission denied
      |
      ▼
Container exits
      |
      ▼
CrashLoopBackOff
```

This is a common OpenShift troubleshooting scenario.

---

# 16. OpenShift-Compatible NGINX Image

The problem was solved by using an unprivileged NGINX image:

```text
nginxinc/nginx-unprivileged:1.27
```

Architecture:

```text
OpenShift
    |
    ▼
Restricted Security
    |
    ▼
Non-root UID
    |
    ▼
nginx-unprivileged
    |
    ▼
Application runs
```

This is preferable to unnecessarily granting root privileges.

---

# 17. SecurityContext

Kubernetes/OpenShift Pods can specify a `securityContext`.

Example:

### `lesson6-securitycontext.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson6-securitycontext
spec:
  securityContext:
    runAsNonRoot: true

  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      command:
        - sh
        - -c
        - |
          echo "Running as:"
          id
          sleep 3600
```

Apply:

```bash
oc apply -f lesson6-securitycontext.yaml
```

Check:

```bash
oc get pod lesson6-securitycontext
```

Check UID:

```bash
oc exec lesson6-securitycontext -- id
```

---

# 18. `runAsNonRoot`

The following configuration:

```yaml
securityContext:
  runAsNonRoot: true
```

means:

> The container must not run as UID 0.

Conceptually:

```text
UID 0
 |
 X Not allowed

UID != 0
 |
 ✓ Allowed
```

This is a useful security control for application workloads.

---

# 19. `runAsUser`

A specific UID can also be requested:

```yaml
securityContext:
  runAsUser: 1000
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson6-runasuser
spec:
  securityContext:
    runAsUser: 1000

  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      command:
        - sh
        - -c
        - |
          id
          sleep 3600
```

However, OpenShift SCC policies can restrict which UID values are permitted.

Therefore, applications should not blindly assume that a particular UID is always allowed.

---

# 20. `allowPrivilegeEscalation`

Another important security setting is:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson6-no-escalation
spec:
  securityContext:
    runAsNonRoot: true

  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      securityContext:
        allowPrivilegeEscalation: false
      command:
        - sh
        - -c
        - |
          id
          sleep 3600
```

Apply:

```bash
oc apply -f lesson6-no-escalation.yaml
```

---

# 21. Linux Capabilities

Linux capabilities divide some root privileges into individual permissions.

Examples include:

```text
NET_ADMIN
NET_RAW
SYS_ADMIN
```

Instead of granting unnecessary capabilities, they can be dropped.

Example:

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

Example Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson6-capabilities
spec:
  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      securityContext:
        capabilities:
          drop:
            - ALL
      command:
        - sh
        - -c
        - |
          echo "Capabilities reduced"
          sleep 3600
```

---

# 22. `privileged: true`

A container can request privileged mode:

```yaml
securityContext:
  privileged: true
```

Privileged containers receive significantly elevated permissions.

For normal application workloads:

```text
privileged: true
```

should generally be avoided.

Preferred approach:

```text
Least Privilege
       |
       ▼
Non-root
       |
       ▼
Restricted SCC
       |
       ▼
Only required capabilities
```

---

# 23. `anyuid`

OpenShift provides an SCC called:

```text
anyuid
```

It can allow workloads to run with arbitrary UIDs, including root where permitted.

It may make some incompatible applications work, but granting additional privileges should not be the first solution.

Avoid blindly changing workloads to use:

```text
anyuid
```

just to solve a filesystem permission problem.

Prefer:

```text
Fix container image
       |
       ▼
Run as non-root
       |
       ▼
Use restricted security
```

---

# 24. ServiceAccounts

Pods run using a ServiceAccount.

List ServiceAccounts:

```bash
oc get serviceaccount
```

Common ServiceAccounts may include:

```text
builder
default
deployer
```

A Pod that doesn't explicitly specify a ServiceAccount commonly uses:

```text
default
```

---

# 25. Check Pod ServiceAccount

Run:

```bash
oc get pod <pod-name> \
  -o jsonpath="{.spec.serviceAccountName}"
```

Example:

```bash
oc get pod lesson6-user \
  -o jsonpath="{.spec.serviceAccountName}"
```

Possible result:

```text
default
```

Conceptually:

```text
Pod
 |
 ▼
ServiceAccount
 |
 ▼
Authorization / Security Policies
 |
 ▼
SCC
```

---

# 26. SCC and ServiceAccount

SCC determines what security configuration a Pod is allowed to use.

The Pod runs using an identity such as a ServiceAccount.

Conceptually:

```text
User / ServiceAccount
          |
          ▼
       Pod
          |
          ▼
        SCC
          |
          ▼
 Security Restrictions
          |
          ▼
      Container
```

The exact authorization relationship depends on the SCC configuration and OpenShift RBAC.

---

# 27. Check SCC Applied to a Pod

Use:

```bash
oc get pod <pod-name> \
  -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

Example:

```bash
oc get pod lesson6-user \
  -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

Possible result:

```text
restricted-v2
```

---

# 28. Pod SecurityContext vs SCC

The Pod can request:

```yaml
securityContext:
  runAsNonRoot: true
```

The SCC determines whether that security configuration is permitted.

Conceptually:

```text
Pod securityContext
        |
        ▼
OpenShift SCC
        |
        ▼
Security Policy
        |
   +----+----+
   |         |
  Allow     Deny
   |         |
   ▼         ▼
 Pod       Rejected/
runs       modified
```

Therefore:

> `securityContext` defines requested container security settings, while SCC is an OpenShift-level policy mechanism that controls what security settings workloads are allowed to use.

---

# 29. OpenShift-Compatible Container Design

When creating custom Docker images for OpenShift:

### Avoid unnecessary root execution

Do not assume the application must run as:

```text
UID 0
```

### Don't assume a fixed UID

OpenShift may assign a non-root UID.

### Don't assume protected directories are writable

Avoid relying on:

```text
/root
/etc
/var/cache
```

being writable by the application.

### Use appropriate writable locations

For example:

```text
/tmp
/app
/data
```

with correct permissions.

---

# 30. Arbitrary UID Concept

A container image may assume:

```text
UID 1001
```

but OpenShift may run it using another non-root UID.

Conceptually:

```text
Container Image
       |
       | expects UID 1001
       |
       X
       |
OpenShift
       |
       +-- Runs using another allowed UID
```

Therefore, OpenShift-friendly images should be designed to work with arbitrary non-root UIDs where appropriate.

This is particularly important for custom application images.

---

# 31. Security Best Practices

For normal application workloads:

```text
✓ Run as non-root
✓ Use restricted SCC where possible
✓ Disable privilege escalation where appropriate
✓ Drop unnecessary capabilities
✓ Avoid privileged containers
✓ Avoid unnecessary anyuid usage
✓ Use least privilege
✓ Use writable directories intentionally
✓ Design images for arbitrary non-root UIDs
✓ Keep application images secure and minimal
```

---

# 32. Real-World Security Flow

```text
                    User
                     |
                     ▼
                Deployment
                     |
                     ▼
                    Pod
                     |
                     ▼
               ServiceAccount
                     |
                     ▼
                    SCC
                     |
          +----------+----------+
          |                     |
          ▼                     ▼
   Security Rules          UID / SELinux
          |                     |
          +----------+----------+
                     |
                     ▼
                 Container
                     |
                     ▼
                 Application
```

---

# 33. Final Hands-on Challenge

Create:

### `lesson6-final.yaml`

Requirements:

### Pod

Name:

```text
lesson6-secure-app
```

### Image

```text
registry.access.redhat.com/ubi9/ubi-minimal
```

### Security requirements

The container must:

```text
Run as non-root
Prevent privilege escalation
Drop all Linux capabilities
```

Use the equivalent of:

```yaml
securityContext:
  runAsNonRoot: true
```

and:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

The container should print:

```text
Security configuration:
Current UID
Current GID
```

and remain running.

---

# 34. Verify Final Pod

Check:

```bash
oc get pod lesson6-secure-app
```

Expected:

```text
NAME                  READY   STATUS
lesson6-secure-app    1/1     Running
```

Check UID:

```bash
oc exec lesson6-secure-app -- id
```

Verify:

```text
UID != 0
```

Check SCC:

```bash
oc get pod lesson6-secure-app \
  -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

The result identifies the SCC applied to the Pod.

---

# 35. Verify SecurityContext

Run:

```bash
oc get pod lesson6-secure-app -o yaml
```

Verify the security configuration:

```text
runAsNonRoot
allowPrivilegeEscalation
capabilities
```

---

# 36. Troubleshooting Security Problems

If a Pod doesn't start:

```bash
oc get pod <pod-name>
```

Then:

```bash
oc describe pod <pod-name>
```

Check Events.

Then:

```bash
oc logs <pod-name>
```

Check the Pod configuration:

```bash
oc get pod <pod-name> -o yaml
```

Check SCC:

```bash
oc get pod <pod-name> \
  -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

Check the ServiceAccount:

```bash
oc get pod <pod-name> \
  -o jsonpath="{.spec.serviceAccountName}"
```

---

# 37. Security Troubleshooting Flow

```text
Pod Not Running
      |
      ▼
oc get pod
      |
      ▼
oc describe pod
      |
      ▼
Check Events
      |
      ▼
oc logs
      |
      ▼
Check SecurityContext
      |
      ▼
Check SCC
      |
      ▼
Check ServiceAccount
      |
      ▼
Check UID / Permissions
      |
      ▼
Check Container Image
      |
      ▼
Fix Security/Application Configuration
```

---

# 🧪 Hands-on Checklist

- [x] Learned Security Context Constraints
- [x] Listed SCCs
- [x] Inspected `restricted-v2`
- [x] Inspected `restricted`
- [x] Identified SCC applied to a Pod
- [x] Learned root vs non-root
- [x] Checked UID/GID inside a container
- [x] Tested filesystem permissions
- [x] Revisited the Lesson 3 NGINX permission problem
- [x] Used an unprivileged NGINX image
- [x] Learned Kubernetes `securityContext`
- [x] Practiced `runAsNonRoot`
- [x] Practiced `runAsUser`
- [x] Practiced `allowPrivilegeEscalation`
- [x] Practiced dropping Linux capabilities
- [x] Learned about privileged containers
- [x] Learned about `anyuid`
- [x] Learned ServiceAccounts
- [x] Checked the ServiceAccount used by a Pod
- [x] Learned arbitrary UID concepts
- [x] Practiced OpenShift-compatible container design
- [x] Built a restricted security Pod
- [x] Troubleshot security-related permission problems

---

# 🧠 Key Takeaways

### SCC

> Security Context Constraints are an OpenShift mechanism for controlling the security capabilities of Pods and containers.

### SecurityContext

> Defines security-related settings requested by a Pod or container.

### UID 0

> UID 0 represents the root user.

### Non-root

> Running applications as non-root reduces unnecessary privileges.

### `runAsNonRoot`

> Ensures the container does not run as UID 0.

### `allowPrivilegeEscalation`

> Controls whether a process can gain more privileges than its parent process.

### Linux Capabilities

> Break certain root privileges into smaller individual permissions.

### `privileged`

> Gives a container significantly elevated privileges and should generally be avoided for normal application workloads.

### `anyuid`

> An SCC that allows workloads to run with arbitrary UIDs where authorized.

### ServiceAccount

> Provides an identity for Pods when interacting with OpenShift/Kubernetes APIs and security policies.

### Arbitrary UID

> OpenShift may run applications using a UID different from the UID assumed by the container image.

---

# 🎤 Interview Questions

## 1. What is SCC?

SCC stands for Security Context Constraints. It is an OpenShift mechanism used to control the security capabilities and privileges available to Pods.

---

## 2. Why does OpenShift use SCC?

SCC helps enforce security policies such as non-root execution, restricted privileges, capability restrictions and other container security controls.

---

## 3. What is the difference between SCC and SecurityContext?

`securityContext` defines security settings requested by a Pod/container.

SCC is an OpenShift-level security policy that controls which security settings workloads are allowed to use.

---

## 4. What is UID 0?

UID 0 represents the Linux root user.

---

## 5. Why should applications run as non-root?

Running as non-root follows the principle of least privilege and reduces the impact of a potential application compromise.

---

## 6. What is `runAsNonRoot`?

It ensures that the container must not run as UID 0.

---

## 7. What is `runAsUser`?

It specifies the UID under which a container process should run, subject to OpenShift security policy restrictions.

---

## 8. What is `allowPrivilegeEscalation`?

It controls whether a process can gain more privileges than its parent process.

---

## 9. What are Linux capabilities?

Linux capabilities split certain root privileges into individual permissions such as `NET_ADMIN` and `NET_RAW`.

---

## 10. Why would you drop capabilities?

To reduce unnecessary privileges and follow least-privilege security principles.

Example:

```yaml
capabilities:
  drop:
    - ALL
```

---

## 11. What is a privileged container?

A privileged container receives significantly elevated permissions and should generally not be used for normal application workloads.

---

## 12. What is `anyuid`?

`anyuid` is an OpenShift SCC that can allow workloads to run using arbitrary UIDs, including root where authorized.

---

## 13. Why shouldn't you immediately use `anyuid` to fix permission problems?

It may weaken the security posture. The preferred solution is often to make the container image compatible with non-root execution.

---

## 14. How do you find which SCC a Pod is using?

```bash
oc get pod <pod-name> \
  -o jsonpath="{.metadata.annotations.openshift\.io/scc}"
```

---

## 15. How do you find the ServiceAccount used by a Pod?

```bash
oc get pod <pod-name> \
  -o jsonpath="{.spec.serviceAccountName}"
```

---

## 16. Why can a Docker image work in Docker but fail in OpenShift?

The image may assume root privileges or write access to protected filesystem locations. OpenShift's restricted security model may prevent those operations.

---

## 17. How would you troubleshoot a permission denied error?

Start with:

```bash
oc describe pod <pod-name>
oc logs <pod-name>
```

Then inspect:

```text
SecurityContext
SCC
ServiceAccount
UID
Filesystem permissions
Container image design
```

---

## 18. Why did the NGINX container fail in Lesson 3?

The NGINX container attempted to create:

```text
/var/cache/nginx/client_temp
```

but the restricted OpenShift execution environment did not allow the container's user to write there.

The container then exited and entered:

```text
CrashLoopBackOff
```

---

## 19. How can you design a Docker image for OpenShift?

Design the image to:

```text
Run as non-root
Support arbitrary non-root UIDs
Avoid writing to protected directories
Use appropriate writable directories
Avoid unnecessary privileges
```

---

## 20. What is an arbitrary UID?

An arbitrary UID is a runtime UID assigned by the platform rather than a fixed UID assumed by the application image.

OpenShift workloads should be designed to work with allowed non-root UIDs rather than assuming a specific UID.

---


# ✅ Lesson 6 Completed

The main security architecture learned in this lesson:

```text
                    User
                     |
                     ▼
                Deployment
                     |
                     ▼
                    Pod
                     |
                     ▼
               ServiceAccount
                     |
                     ▼
                    SCC
                     |
          +----------+----------+
          |                     |
          ▼                     ▼
   Security Rules          UID / SELinux
          |                     |
          +----------+----------+
                     |
                     ▼
                 Container
                     |
                     ▼
                 Application
```

The main practical lesson was understanding why an application that works in a normal Docker environment can fail on OpenShift.

For example:

```text
NGINX
  |
  ▼
Writes to /var/cache/nginx
  |
  ▼
OpenShift restricted security
  |
  ▼
Permission denied
  |
  ▼
Container exits
  |
  ▼
CrashLoopBackOff
```

The preferred solution was to use an OpenShift-compatible unprivileged image:

```text
nginxinc/nginx-unprivileged:1.27
```

rather than unnecessarily granting elevated privileges.

---

# 🔐 Final Security Takeaway

A production OpenShift workload should follow the principle of least privilege:

```text
                    Application
                         |
                         ▼
                    Non-root UID
                         |
                         ▼
                  Restricted SCC
                         |
                         ▼
             No privilege escalation
                         |
                         ▼
             Drop unnecessary capabilities
                         |
                         ▼
                 Minimal permissions
```

The most important troubleshooting rule learned in this lesson is:

> **When an application fails with `Permission denied` on OpenShift, don't immediately grant root or privileged access. First check the container UID, filesystem permissions, securityContext, SCC, ServiceAccount and container image design.**
