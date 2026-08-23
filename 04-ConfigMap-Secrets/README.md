# OpenShift Learning – Lesson 4

## ConfigMaps, Secrets & Application Configuration

In this lesson, I learned how to manage **application configuration and sensitive information in OpenShift** using ConfigMaps and Secrets.

I performed hands-on practice using the **Red Hat OpenShift Developer Sandbox**, including creating ConfigMaps and Secrets, injecting them as environment variables, mounting configuration as files, updating configuration, and troubleshooting a real `CreateContainerConfigError`.

---

## 🎯 Learning Objectives

- Understand ConfigMaps
- Understand Secrets
- Create ConfigMaps using the `oc` CLI
- Create Secrets using the `oc` CLI
- Inject ConfigMap values into Pods
- Inject Secret values into Pods
- Use `env` and `envFrom`
- Understand `configMapKeyRef`
- Understand `secretKeyRef`
- Mount ConfigMaps as files
- Mount Secrets as files
- Update ConfigMaps
- Understand configuration changes in running Pods
- Understand Base64 encoding of Secrets
- Troubleshoot `CreateContainerConfigError`
- Understand ConfigMap vs Secret

---

# 1. Why Do We Need ConfigMaps and Secrets?

Applications usually require configuration such as:

```text
Application Name
Environment
Log Level
Database Host
API URL
```

Applications may also require sensitive information such as:

```text
Database Username
Database Password
API Token
Certificates
Private Keys
```

These values should not normally be hard-coded into the container image.

OpenShift provides:

```text
ConfigMap
    |
    +-- Non-sensitive configuration

Secret
    |
    +-- Sensitive configuration
```

---

# 2. Lesson 4 Architecture

```text
                    OpenShift
                        |
                        v
                   Deployment
                        |
                        v
                       Pods
                  +-----+-----+
                  |           |
                  v           v
              ConfigMap     Secret
                  |           |
                  v           v
              App Config   Credentials
```

Example:

```text
ConfigMap
--------
APP_NAME=OpenShift-Learning
APP_ENV=production
APP_VERSION=1.0

Secret
------
DB_USERNAME=admin
DB_PASSWORD=********
```

---

# 3. What is a ConfigMap?

A **ConfigMap** stores non-sensitive application configuration.

Examples:

```text
APP_NAME=OpenShift-Learning
APP_ENV=development
APP_VERSION=1.0
LOG_LEVEL=info
```

Instead of putting these values directly into a container image, they can be stored separately in OpenShift.

---

# 4. Create a ConfigMap

Create a ConfigMap using literal values:

```bash
oc create configmap lesson4-config --from-literal=APP_NAME=OpenShift-Learning --from-literal=APP_ENV=development --from-literal=APP_VERSION=1.0
```

PowerShell:

```powershell
oc create configmap lesson4-config --from-literal=APP_NAME=OpenShift-Learning --from-literal=APP_ENV=development --from-literal=APP_VERSION=1.0
```

Expected:

```text
configmap/lesson4-config created
```

---

# 5. Inspect the ConfigMap

List ConfigMaps:

```bash
oc get configmap
```

Inspect the ConfigMap:

```bash
oc describe configmap lesson4-config
```

View YAML:

```bash
oc get configmap lesson4-config -o yaml
```

Example data:

```yaml
data:
  APP_ENV: development
  APP_NAME: OpenShift-Learning
  APP_VERSION: "1.0"
```

---

# 6. ConfigMap as Environment Variables

A Pod can read individual ConfigMap values as environment variables.

### `lesson4-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson4-config-pod
  labels:
    app: lesson4-config
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Application starting..."
          echo "APP_NAME=$APP_NAME"
          echo "APP_ENV=$APP_ENV"
          echo "APP_VERSION=$APP_VERSION"
          sleep 3600
      env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: lesson4-config
              key: APP_NAME

        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: lesson4-config
              key: APP_ENV

        - name: APP_VERSION
          valueFrom:
            configMapKeyRef:
              name: lesson4-config
              key: APP_VERSION
```

Apply:

```bash
oc apply -f lesson4-pod.yaml
```

Check:

```bash
oc get pod lesson4-config-pod
```

Expected:

```text
NAME                 READY   STATUS    RESTARTS
lesson4-config-pod   1/1     Running   0
```

---

# 7. Verify Environment Variables

Check all environment variables:

```bash
oc exec lesson4-config-pod -- env
```

Expected values:

```text
APP_NAME=OpenShift-Learning
APP_ENV=development
APP_VERSION=1.0
```

Check an individual variable:

```bash
oc exec lesson4-config-pod -- sh -c 'echo $APP_NAME'
```

Expected:

```text
OpenShift-Learning
```

---

# 8. What is `configMapKeyRef`?

`configMapKeyRef` allows us to read a specific key from a ConfigMap.

Example:

```yaml
env:
  - name: APP_NAME
    valueFrom:
      configMapKeyRef:
        name: lesson4-config
        key: APP_NAME
```

Flow:

```text
ConfigMap
lesson4-config
      |
      +-- APP_NAME=OpenShift-Learning
                    |
                    v
                   Pod
                    |
                    v
        Environment Variable
                    |
                    v
APP_NAME=OpenShift-Learning
```

---

# 9. Using `envFrom`

Instead of referencing every ConfigMap key individually, all values can be loaded using `envFrom`.

### `lesson4-envfrom.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson4-envfrom-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          env
          sleep 3600
      envFrom:
        - configMapRef:
            name: lesson4-config
```

Apply:

```bash
oc apply -f lesson4-envfrom.yaml
```

Check:

```bash
oc get pod lesson4-envfrom-pod
```

View logs:

```bash
oc logs lesson4-envfrom-pod
```

The ConfigMap values should appear:

```text
APP_NAME=OpenShift-Learning
APP_ENV=development
APP_VERSION=1.0
```

---

# 10. `env` vs `envFrom`

Using `env`:

```yaml
env:
  - name: APP_NAME
    valueFrom:
      configMapKeyRef:
        name: lesson4-config
        key: APP_NAME
```

This allows us to select individual values.

Using `envFrom`:

```yaml
envFrom:
  - configMapRef:
      name: lesson4-config
```

This loads all compatible ConfigMap keys as environment variables.

Easy way to remember:

```text
env
 |
 +-- Select specific keys

envFrom
 |
 +-- Load all keys
```

---

# 11. ConfigMap as a File

ConfigMaps can also provide configuration files to containers.

Example application configuration:

```properties
app.name=OpenShift Learning
app.environment=development
app.version=1.0
```

Concept:

```text
ConfigMap
    |
    v
Volume
    |
    v
Container
    |
    v
/etc/app-config/application.properties
```

---

# 12. Mount ConfigMap as a Volume

Example Pod:

### `lesson4-config-volume.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson4-config-volume
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          cat /etc/app-config/application.properties
          sleep 3600
      volumeMounts:
        - name: app-config
          mountPath: /etc/app-config

  volumes:
    - name: app-config
      configMap:
        name: lesson4-app-config
```

Apply:

```bash
oc apply -f lesson4-config-volume.yaml
```

Check:

```bash
oc get pod lesson4-config-volume
```

Inspect mounted files:

```bash
oc exec lesson4-config-volume -- ls -l /etc/app-config
```

Read the configuration:

```bash
oc exec lesson4-config-volume -- cat /etc/app-config/application.properties
```

---

# 13. Updating a ConfigMap

A ConfigMap can be updated without rebuilding the container image.

Example:

```powershell
oc create configmap lesson4-config --from-literal=APP_NAME=OpenShift-Learning --from-literal=APP_ENV=production --from-literal=APP_VERSION=2.0 --dry-run=client -o yaml | oc apply -f -
```

Verify:

```bash
oc get configmap lesson4-config -o yaml
```

Updated configuration:

```text
APP_NAME=OpenShift-Learning
APP_ENV=production
APP_VERSION=2.0
```

---

# 14. Important ConfigMap Update Behaviour

If ConfigMap values are injected into a container as **environment variables**, changing the ConfigMap does not update the environment variables inside an already-running container.

The Pod/container needs to be recreated or restarted to receive the new environment values.

Concept:

```text
ConfigMap v1
     |
     v
Running Pod
APP_ENV=development

        ConfigMap Updated

ConfigMap v2
APP_ENV=production

Running Pod
APP_ENV=development
     |
     | Pod Restart/Recreation
     v
New Pod
APP_ENV=production
```

For a Deployment, a rollout restart can be used when appropriate:

```bash
oc rollout restart deployment/<deployment-name>
```

---

# 15. What is a Secret?

A **Secret** is used for sensitive application information.

Examples:

```text
Database Username
Database Password
API Token
TLS Certificate
Private Key
```

Example:

```text
DB_USERNAME=admin
DB_PASSWORD=********
```

---

# 16. Create a Secret

Create a generic Secret:

```bash
oc create secret generic lesson4-secret --from-literal=DB_USERNAME=admin --from-literal=DB_PASSWORD='P@ssw0rd123'
```

PowerShell:

```powershell
oc create secret generic lesson4-secret --from-literal=DB_USERNAME=admin --from-literal=DB_PASSWORD='P@ssw0rd123'
```

Expected:

```text
secret/lesson4-secret created
```

---

# 17. Inspect the Secret

List Secrets:

```bash
oc get secret
```

Inspect metadata and key sizes:

```bash
oc describe secret lesson4-secret
```

Example:

```text
Data
====
DB_PASSWORD:  11 bytes
DB_USERNAME:  5 bytes
```

View YAML:

```bash
oc get secret lesson4-secret -o yaml
```

The values under `data` are represented using Base64 encoding.

---

# 18. Base64 is NOT Encryption

This is an important security concept.

```text
Base64
   !=
Encryption
```

Base64 is an **encoding mechanism**, not an encryption mechanism.

Anyone who has permission to read the Secret data can potentially decode the Base64 values.

Therefore, access to Secrets should be controlled using mechanisms such as:

```text
RBAC
Least Privilege
Service Accounts
Secret Management Policies
```

---

# 19. Secret as Environment Variables

### `lesson4-secret-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson4-secret-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Secret variables loaded"
          sleep 3600
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: lesson4-secret
              key: DB_USERNAME

        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: lesson4-secret
              key: DB_PASSWORD
```

Apply:

```bash
oc apply -f lesson4-secret-pod.yaml
```

Check:

```bash
oc get pod lesson4-secret-pod
```

Expected:

```text
NAME                 READY   STATUS    RESTARTS
lesson4-secret-pod   1/1     Running   0
```

---

# 20. What is `secretKeyRef`?

`secretKeyRef` reads a specific key from a Secret.

Example:

```yaml
- name: DB_USERNAME
  valueFrom:
    secretKeyRef:
      name: lesson4-secret
      key: DB_USERNAME
```

Flow:

```text
Secret
lesson4-secret
       |
       +-- DB_USERNAME
       |
       +-- DB_PASSWORD
              |
              v
             Pod
              |
              v
      Environment Variables
```

---

# 21. Secret Using `envFrom`

All compatible Secret keys can also be loaded using `envFrom`.

```yaml
envFrom:
  - secretRef:
      name: lesson4-secret
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson4-secret-envfrom
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Secret loaded"
          sleep 3600
      envFrom:
        - secretRef:
            name: lesson4-secret
```

Apply:

```bash
oc apply -f lesson4-secret-envfrom.yaml
```

---

# 22. Mount Secret as Files

Secrets can also be mounted into containers as files.

### `lesson4-secret-volume.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson4-secret-volume
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Secret files mounted"
          ls -l /etc/secret
          sleep 3600
      volumeMounts:
        - name: db-secret
          mountPath: /etc/secret
          readOnly: true

  volumes:
    - name: db-secret
      secret:
        secretName: lesson4-secret
```

Apply:

```bash
oc apply -f lesson4-secret-volume.yaml
```

Check:

```bash
oc get pod lesson4-secret-volume
```

Inspect:

```bash
oc exec lesson4-secret-volume -- ls -l /etc/secret
```

Expected files:

```text
DB_PASSWORD
DB_USERNAME
```

Architecture:

```text
Secret
   |
   v
Volume
   |
   v
/etc/secret/
   |
   +-- DB_USERNAME
   |
   +-- DB_PASSWORD
```

---

# 23. ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---|---|---|
| Non-sensitive configuration | ✅ | Possible, but unnecessary |
| Passwords | ❌ | ✅ |
| API tokens | ❌ | ✅ |
| Database credentials | ❌ | ✅ |
| Environment variables | ✅ | ✅ |
| Mounted as files | ✅ | ✅ |
| `envFrom` | ✅ | ✅ |
| Individual key reference | `configMapKeyRef` | `secretKeyRef` |
| Intended for sensitive data | ❌ | ✅ |

Simple rule:

```text
ConfigMap
    |
    v
Non-sensitive configuration

Secret
    |
    v
Sensitive configuration
```

---

# 24. Real Troubleshooting – CreateContainerConfigError

During this lesson, the Secret Pod entered:

```text
CreateContainerConfigError
```

Command:

```bash
oc get pod lesson4-secret-pod
```

Output:

```text
NAME                 READY   STATUS
lesson4-secret-pod   0/1     CreateContainerConfigError
```

Instead of changing the image or restarting randomly, the Pod was inspected:

```bash
oc describe pod lesson4-secret-pod
```

The Events showed the exact problem:

```text
Error: couldn't find key DB_USERNAME in Secret
lesson4-secret
```

---

# 25. Root Cause

The Pod configuration expected:

```yaml
secretKeyRef:
  name: lesson4-secret
  key: DB_USERNAME
```

OpenShift therefore expected:

```text
Secret: lesson4-secret
        |
        +-- DB_USERNAME
        |
        +-- DB_PASSWORD
```

But the required `DB_USERNAME` key was missing.

Therefore:

```text
Pod Created
     |
     v
Read Secret Reference
     |
     v
Find lesson4-secret
     |
     v
Find DB_USERNAME
     |
     X
Key Missing
     |
     v
CreateContainerConfigError
```

The container itself had not started.

---

# 26. Troubleshooting CreateContainerConfigError

First check the Pod:

```bash
oc get pods
```

Then inspect it:

```bash
oc describe pod lesson4-secret-pod
```

Check the Secret:

```bash
oc get secret lesson4-secret
```

Inspect the Secret keys:

```bash
oc describe secret lesson4-secret
```

Then compare:

```text
Pod expects:
DB_USERNAME

Secret contains:
?

Pod expects:
DB_PASSWORD

Secret contains:
?
```

The key names must match exactly.

---

# 27. Fix the Missing Secret Key

The incorrect Secret was removed:

```bash
oc delete secret lesson4-secret
```

Then recreated correctly:

```bash
oc create secret generic lesson4-secret --from-literal=DB_USERNAME=admin --from-literal=DB_PASSWORD='P@ssw0rd123'
```

PowerShell:

```powershell
oc create secret generic lesson4-secret --from-literal=DB_USERNAME=admin --from-literal=DB_PASSWORD='P@ssw0rd123'
```

Verify:

```bash
oc describe secret lesson4-secret
```

The required keys should now exist:

```text
DB_USERNAME
DB_PASSWORD
```

Then recreate the standalone Pod if needed:

```bash
oc delete pod lesson4-secret-pod
oc apply -f lesson4-secret-pod.yaml
```

Verify:

```bash
oc get pod lesson4-secret-pod
```

Expected:

```text
lesson4-secret-pod   1/1   Running
```

---

# 28. Error Status Comparison

This lesson also helped distinguish common Pod errors.

| Status | Meaning |
|---|---|
| `Pending` | Pod has not successfully started/scheduled yet |
| `ImagePullBackOff` | Container image cannot be pulled |
| `CreateContainerConfigError` | Container configuration cannot be created |
| `CrashLoopBackOff` | Container starts but repeatedly crashes |
| `Running` | Pod/container is running |

Important troubleshooting principle:

```text
Do not guess based only on STATUS.

Use:

oc describe pod <pod-name>
oc logs <pod-name>
```

For `CreateContainerConfigError`, Events are especially useful because the container may not have started yet.

---

# 29. ConfigMap and Secret Architecture

A typical application can use both:

```text
                         Internet
                            |
                            v
                     OpenShift Route
                            |
                            v
                         Service
                            |
                            v
                       Deployment
                            |
                +-----------+-----------+
                |                       |
                v                       v
              Pod 1                   Pod 2
                |                       |
          +-----+-----+           +-----+-----+
          |           |           |           |
          v           v           v           v
      ConfigMap     Secret    ConfigMap     Secret
          |           |           |           |
          v           v           v           v
      App Config    DB Creds   App Config   DB Creds
```

Example:

### ConfigMap

```text
APP_NAME=OpenShift-Learning
APP_ENV=production
APP_VERSION=1.0
LOG_LEVEL=info
```

### Secret

```text
DB_USERNAME=admin
DB_PASSWORD=********
API_TOKEN=********
```

---

# 30. Important Commands Learned

## ConfigMaps

```bash
oc get configmap
oc describe configmap lesson4-config
oc get configmap lesson4-config -o yaml
```

Create:

```bash
oc create configmap lesson4-config --from-literal=APP_NAME=OpenShift-Learning
```

---

## Secrets

```bash
oc get secret
oc describe secret lesson4-secret
oc get secret lesson4-secret -o yaml
```

Create:

```bash
oc create secret generic lesson4-secret --from-literal=DB_USERNAME=admin --from-literal=DB_PASSWORD='P@ssw0rd123'
```

---

## Pods

```bash
oc get pods
oc describe pod <pod-name>
oc logs <pod-name>
```

---

## Check Environment Variables

```bash
oc exec <pod-name> -- env
```

---

## Check Mounted ConfigMap

```bash
oc exec lesson4-config-volume -- ls -l /etc/app-config
```

```bash
oc exec lesson4-config-volume -- cat /etc/app-config/application.properties
```

---

## Check Mounted Secret

```bash
oc exec lesson4-secret-volume -- ls -l /etc/secret
```

---

# 31. Troubleshooting Flow

When a Pod has a configuration error:

```text
Pod Not Running
      |
      v
oc get pods
      |
      v
Check STATUS
      |
      v
oc describe pod <pod>
      |
      v
Check Events
      |
      +----------------------+
      |                      |
      v                      v
ConfigMap Error          Secret Error
      |                      |
      v                      v
Check ConfigMap          Check Secret
      |                      |
      v                      v
Check Key Names          Check Key Names
      |                      |
      +----------+-----------+
                 |
                 v
             Fix Config
                 |
                 v
          Recreate/Restart
                 |
                 v
            Verify Pod
```

---

# 32. Hands-on Checklist

- [x] Learned ConfigMap basics
- [x] Created ConfigMap
- [x] Inspected ConfigMap
- [x] Used ConfigMap as environment variables
- [x] Used `configMapKeyRef`
- [x] Used ConfigMap with `envFrom`
- [x] Mounted ConfigMap as a file
- [x] Inspected mounted ConfigMap
- [x] Updated ConfigMap
- [x] Learned ConfigMap update behaviour
- [x] Learned Secret basics
- [x] Created Secret
- [x] Inspected Secret
- [x] Learned Base64 vs encryption
- [x] Used Secret as environment variables
- [x] Used `secretKeyRef`
- [x] Learned Secret with `envFrom`
- [x] Learned Secret volume mounts
- [x] Encountered `CreateContainerConfigError`
- [x] Used `oc describe pod` for troubleshooting
- [x] Identified missing `DB_USERNAME`
- [x] Fixed Secret configuration
- [x] Learned ConfigMap vs Secret

---

# 🧠 Key Takeaways

### ConfigMap

> Used to separate non-sensitive application configuration from the container image.

### Secret

> Used to store sensitive application information such as credentials, tokens and keys.

### `configMapKeyRef`

> Reads a specific value from a ConfigMap.

### `secretKeyRef`

> Reads a specific value from a Secret.

### `envFrom`

> Loads multiple ConfigMap or Secret keys into a container as environment variables.

### Volume Mount

> ConfigMaps and Secrets can be presented to applications as files.

### Base64

> Base64 is encoding, not encryption.

### CreateContainerConfigError

> Indicates that OpenShift/Kubernetes could not construct the container configuration, often because a referenced ConfigMap, Secret or key is missing or invalid.

---

# 🎤 Interview Questions

## 1. What is a ConfigMap?

A ConfigMap stores non-sensitive application configuration separately from the container image.

---

## 2. What is a Secret?

A Secret is used to store sensitive information such as passwords, tokens, certificates and credentials.

---

## 3. What is the difference between ConfigMap and Secret?

ConfigMaps are intended for non-sensitive configuration, while Secrets are intended for sensitive information.

---

## 4. How can a Pod consume a ConfigMap?

A Pod can consume a ConfigMap using:

- Environment variables
- `envFrom`
- `configMapKeyRef`
- Volume mounts

---

## 5. How can a Pod consume a Secret?

A Pod can consume a Secret using:

- Environment variables
- `envFrom`
- `secretKeyRef`
- Volume mounts

---

## 6. What is `configMapKeyRef`?

It allows a container environment variable to reference a specific key inside a ConfigMap.

---

## 7. What is `secretKeyRef`?

It allows a container environment variable to reference a specific key inside a Secret.

---

## 8. What is `envFrom`?

`envFrom` loads multiple keys from a ConfigMap or Secret into a container as environment variables.

---

## 9. Are Kubernetes/OpenShift Secrets encrypted because they use Base64?

No.

Base64 is encoding, not encryption.

---

## 10. What happens if a Secret key referenced by a Pod doesn't exist?

The container may fail to start with an error such as:

```text
CreateContainerConfigError
```

The Pod Events can show an error similar to:

```text
couldn't find key DB_USERNAME in Secret lesson4-secret
```

---

## 11. How do you troubleshoot `CreateContainerConfigError`?

Start with:

```bash
oc describe pod <pod-name>
```

Check the Events section.

Then verify referenced resources:

```bash
oc get configmap
oc get secret
oc describe configmap <configmap-name>
oc describe secret <secret-name>
```

Check whether the required ConfigMap/Secret and key names exist.

---

## 12. Does updating a ConfigMap automatically update environment variables in a running container?

No.

Environment variables are populated when the container starts. The Pod/container normally needs to be recreated or restarted to receive updated environment variable values.

---

## 13. Why shouldn't passwords be stored in ConfigMaps?

ConfigMaps are intended for non-sensitive configuration. Credentials and other sensitive values should use Secrets and appropriate access controls.

---

## 14. Can Secrets be mounted as files?

Yes.

Secrets can be mounted into a container through a volume.

---


# ✅ Lesson 4 Completed

The main concept learned in this lesson:

```text
                   Application Pod
                         |
             +-----------+-----------+
             |                       |
             v                       v
         ConfigMap                  Secret
             |                       |
             v                       v
     Non-sensitive Config      Sensitive Config
             |                       |
             +-----------+-----------+
                         |
                         v
                    Application
```

## Final Takeaway

**ConfigMaps separate application configuration from container images, while Secrets provide a mechanism for supplying sensitive configuration. Both can be consumed as environment variables or mounted files.**

The most valuable troubleshooting exercise in this lesson was diagnosing:

```text
CreateContainerConfigError
```

using:

```bash
oc describe pod lesson4-secret-pod
```

and identifying the actual root cause:

```text
couldn't find key DB_USERNAME in Secret lesson4-secret
```

Instead of randomly restarting the Pod, the error was traced to the missing Secret key and the configuration was corrected.

This demonstrates an important OpenShift troubleshooting approach:

```text
Observe Status
      |
      v
Inspect Events
      |
      v
Identify Root Cause
      |
      v
Fix Configuration
      |
      v
Verify Application
```
