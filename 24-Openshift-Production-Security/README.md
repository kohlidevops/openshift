# OpenShift Learning – Lesson 24: Production Security

## 🚀 Lesson 24: Production Security

> **Environment:** OpenShift Developer Sandbox  
> **Approach:** Namespace/project-level hands-on practice  
> **Dockerfile:** Not required  
> **Argo CD:** Not required  
> **Cluster-admin access:** Not required  
> **NGINX Image:** `nginxinc/nginx-unprivileged:1.25`

---

# 🎯 Learning Objectives

By the end of this lesson, I should understand and practice:

- Secrets management
- ConfigMap vs Secret
- Secret as environment variables
- Secret as a volume
- OpenShift SCC
- SCC vs `securityContext`
- Non-root containers
- `allowPrivilegeEscalation`
- Linux capabilities
- Least privilege
- TLS
- HTTPS Routes
- Edge TLS termination
- Container image security
- Production security best practices
- Security troubleshooting

---

# 1. Production Security Architecture

We will build:

```text
                         Browser
                            |
                          HTTPS
                            |
                            v
                   +----------------+
                   | OpenShift Route|
                   |   Edge TLS     |
                   +----------------+
                            |
                            v
                   +----------------+
                   |    Service     |
                   +----------------+
                            |
                            v
                   +----------------+
                   |      Pod       |
                   |                |
                   |  Non-root      |
                   |  Container     |
                   |                |
                   |  No privilege  |
                   |  escalation    |
                   |  Drop caps     |
                   +----------------+
                       |        |
                       |        |
                       v        v
                  ConfigMap   Secret
                  non-secret   sensitive
```

---

# 2. Why Production Security Matters

A production application must protect:

```text
Credentials
Containers
Images
Network traffic
Application access
Infrastructure
```

Our security model:

```text
Secure Image
     +
Non-root Container
     +
SCC
     +
SecurityContext
     +
Secrets
     +
TLS
     +
Least Privilege
     =
More Secure Application
```

---

# Part 1 – ConfigMap

## 3. Create ConfigMap

Create:

```cmd
oc create configmap lesson24-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info --from-literal=APP_VERSION=1.0
```

Verify:

```cmd
oc get configmap lesson24-config
```

Describe:

```cmd
oc describe configmap lesson24-config
```

Expected configuration:

```text
APP_ENV=production
LOG_LEVEL=info
APP_VERSION=1.0
```

---

# Part 2 – Secrets Management

## 4. Create Secret

Create:

```cmd
oc create secret generic lesson24-secret --from-literal=USERNAME=admin --from-literal=PASSWORD='MyStrongPassword123!'
```

Verify:

```cmd
oc get secret lesson24-secret
```

Describe:

```cmd
oc describe secret lesson24-secret
```

The actual secret values should not be displayed by `oc describe secret`.

> **Important:** Do not use the example password in a real production environment.

---

# 5. Secret Data

Run:

```cmd
oc get secret lesson24-secret -o yaml
```

You will see encoded data.

Remember:

```text
Base64 ≠ Encryption
```

Base64 is an encoding mechanism, not a security mechanism.

---

# 6. ConfigMap vs Secret

## ConfigMap

Used for non-sensitive configuration:

```text
APP_ENV
LOG_LEVEL
APP_VERSION
API_URL
```

## Secret

Used for sensitive configuration:

```text
PASSWORD
TOKEN
API_KEY
DATABASE_CREDENTIAL
```

Remember:

```text
ConfigMap
    ↓
Non-sensitive

Secret
    ↓
Sensitive
```

---

# Part 3 – Secure Deployment

## 7. Create Secure Deployment

Create:

```text
lesson24-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson24-app
  labels:
    app: lesson24
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lesson24
  template:
    metadata:
      labels:
        app: lesson24
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: lesson24-config
          env:
            - name: APP_USERNAME
              valueFrom:
                secretKeyRef:
                  name: lesson24-secret
                  key: USERNAME
            - name: APP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: lesson24-secret
                  key: PASSWORD
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
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
oc apply -f lesson24-deployment.yaml
```

---

# 8. Verify Deployment

```cmd
oc get deployment lesson24-app
```

Expected:

```text
1/1
```

Check Pods:

```cmd
oc get pods -l app=lesson24
```

---

# 9. Check Logs

```cmd
oc logs <pod-name>
```

The password should not appear in application logs.

Security principle:

```text
Secrets
   ↓
Never intentionally print
   ↓
Application logs
```

---

# 10. Verify ConfigMap Environment Variables

Get the Pod:

```cmd
oc get pods -l app=lesson24
```

Then:

```cmd
oc exec <pod-name> -- env
```

Look for:

```text
APP_ENV=production
LOG_LEVEL=info
APP_VERSION=1.0
APP_USERNAME=admin
```

Do not unnecessarily display or copy the password.

---

# Part 4 – Secret as a Volume

## 11. Create Secret Volume Deployment

Create:

```text
lesson24-secret-volume.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson24-secret-volume
  labels:
    app: lesson24-secret-volume
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lesson24-secret-volume
  template:
    metadata:
      labels:
        app: lesson24-secret-volume
    spec:
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.25
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: app-secret
              mountPath: /tmp/app-secret
              readOnly: true
      volumes:
        - name: app-secret
          secret:
            secretName: lesson24-secret
```

Apply:

```cmd
oc apply -f lesson24-secret-volume.yaml
```

---

# 12. Verify Secret Volume

Get the Pod:

```cmd
oc get pods -l app=lesson24-secret-volume
```

Check the mounted files:

```cmd
oc exec <pod-name> -- ls -l /tmp/app-secret
```

You should see:

```text
USERNAME
PASSWORD
```

Do not print the contents unnecessarily.

---

# 13. Why ReadOnly?

The Secret volume uses:

```yaml
readOnly: true
```

This follows the principle:

```text
Give the application
only the access it needs.
```

The application should not modify the Secret.

---

# Part 5 – Container Security

## 14. What Is a Non-root Container?

A root container has high privileges.

A non-root container has fewer privileges.

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
Fewer privileges
 ↓
Smaller attack surface
```

Our application uses:

```text
nginxinc/nginx-unprivileged:1.25
```

---

# 15. Verify Container User

Run:

```cmd
oc exec <pod-name> -- id
```

Check whether the process is running as root.

Expected concept:

```text
Non-root
```

The exact UID can be assigned by OpenShift and may vary.

---

# 16. Why We Don't Hard-Code a UID

Do not blindly configure:

```yaml
runAsUser: 1001
```

OpenShift may assign a permitted UID through the applicable SCC.

Therefore:

```text
OpenShift
    ↓
SCC
    ↓
Allowed UID
    ↓
Container
```

The exact UID may differ between environments.

---

# 17. `allowPrivilegeEscalation`

Our container uses:

```yaml
allowPrivilegeEscalation: false
```

This prevents processes inside the container from gaining additional privileges through privilege escalation mechanisms.

Conceptually:

```text
Container
    ↓
Process
    ↓
No privilege escalation
```

---

# 18. Linux Capabilities

We configured:

```yaml
capabilities:
  drop:
    - ALL
```

Linux capabilities provide specific privileges.

Instead of allowing unnecessary privileges:

```text
Drop ALL
```

This follows:

> **Least privilege**

Only grant capabilities when the application actually requires them.

---

# Part 6 – SCC

## 19. What Is SCC?

SCC means:

> **Security Context Constraints**

SCC controls security-related characteristics of Pods in OpenShift.

It can control:

```text
Run as root
Run as non-root
Privileged containers
Linux capabilities
Host networking
Host filesystem access
Volume types
UID ranges
```

Conceptually:

```text
Pod
 ↓
SCC
 ↓
Security Restrictions
 ↓
Container
```

---

# 20. SCC vs securityContext

These are different concepts.

### securityContext

Defined as part of the workload:

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

### SCC

Controlled by OpenShift:

```text
OpenShift
    ↓
SCC
    ↓
Pod security restrictions
```

Think:

```text
securityContext
      +
SCC
      ↓
Container Security
```

---

# 21. Sandbox SCC Limitation

We will NOT attempt:

```cmd
oc create scc ...
```

or:

```cmd
oc adm policy add-scc-to-user ...
```

or cluster-wide SCC modifications.

These require elevated permissions and may return:

```text
Forbidden
```

in the Developer Sandbox.

Our goal is:

```text
Understand SCC
       +
Observe its effect
       +
Build secure workloads
```

not administer cluster-wide SCC policies.

---

# 22. Inspect Pod Security

Run:

```cmd
oc get pod <pod-name> -o yaml
```

Look for:

```text
securityContext
```

Also:

```cmd
oc describe pod <pod-name>
```

Review security-related information.

If a command returns:

```text
Forbidden
```

do not try to bypass the permission boundary.

---

# Part 7 – Service

## 23. Create Service

Create:

```text
lesson24-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: lesson24-service
spec:
  selector:
    app: lesson24
  ports:
    - name: http
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

Apply:

```cmd
oc apply -f lesson24-service.yaml
```

Verify:

```cmd
oc get service lesson24-service
```

---

# Part 8 – TLS

## 24. Create HTTPS Route

Create:

```text
lesson24-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: lesson24-route
spec:
  to:
    kind: Service
    name: lesson24-service
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Apply:

```cmd
oc apply -f lesson24-route.yaml
```

---

# 25. Get Route

```cmd
oc get route lesson24-route
```

You will receive a hostname.

Open:

```text
https://<route-hostname>
```

Use:

```text
HTTPS
```

instead of HTTP.

---

# 26. Edge TLS

Our architecture:

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

> **Edge TLS termination**

TLS terminates at the OpenShift router.

---

# 27. OpenShift TLS Options

Common Route TLS approaches:

```text
Edge
Passthrough
Re-encrypt
```

### Edge

```text
Browser
   |
 HTTPS
   ↓
Router
   |
 HTTP
   ↓
Application
```

### Passthrough

```text
Browser
   |
 HTTPS
   ↓
Router
   |
 HTTPS
   ↓
Application
```

### Re-encrypt

```text
Browser
   |
 HTTPS
   ↓
Router
   |
 HTTPS
   ↓
Application
```

For this lesson:

```text
Edge TLS
```

is sufficient.

---

# Part 9 – Image Security

## 28. Why Image Security Matters

Security starts with the image.

```text
Container Image
       ↓
Container Runtime
       ↓
OpenShift
       ↓
Pod
```

A vulnerable image can introduce security problems before the application even starts.

---

# 29. Image Security Questions

Before using an image, ask:

```text
Who maintains the image?

Is the registry trusted?

Is the image maintained?

What base image is used?

Does it run as root?

Does it contain unnecessary packages?

Are vulnerabilities known?

Is the image version controlled?
```

---

# 30. Avoid `latest`

Avoid:

```text
nginxinc/nginx-unprivileged:latest
```

for production workloads.

We use:

```text
nginxinc/nginx-unprivileged:1.25
```

because a fixed version provides more predictable deployments.

For stronger immutability, production environments can use an image digest:

```text
image@sha256:<digest>
```

---

# 31. Check Image Used

```cmd
oc get pod <pod-name> -o jsonpath="{.spec.containers[*].image}"
```

Expected:

```text
nginxinc/nginx-unprivileged:1.25
```

---

# Part 10 – Security Testing

## 32. Check SecurityContext

```cmd
oc get pod <pod-name> -o yaml
```

Verify:

```text
allowPrivilegeEscalation: false
```

and:

```text
capabilities:
  drop:
    - ALL
```

---

# 33. Test Filesystem Permissions

Try:

```cmd
oc exec <pod-name> -- sh -c "touch /etc/test-file"
```

A protected filesystem location should not be writable by the non-root application.

This demonstrates:

```text
Non-root
+
Filesystem Permissions
=
Reduced Attack Surface
```

---

# 34. Verify User

Run:

```cmd
oc exec <pod-name> -- id
```

Understand the UID.

Important:

```text
The exact UID may vary.
```

OpenShift can assign a UID according to its security policy.

---

# Part 11 – Production Security Best Practices

# 35. Secrets

```text
☐ Don't hard-code passwords
☐ Don't commit secrets to Git
☐ Use OpenShift Secrets
☐ Limit Secret access
☐ Don't print secrets in logs
☐ Rotate credentials
```

---

# 36. Containers

```text
☐ Run as non-root
☐ Disable privilege escalation
☐ Drop unnecessary capabilities
☐ Use minimal images
☐ Avoid privileged containers
☐ Use read-only mounts where possible
```

---

# 37. Images

```text
☐ Use trusted registries
☐ Use maintained images
☐ Scan images
☐ Avoid unnecessary packages
☐ Avoid latest in production
☐ Prefer immutable image references
```

---

# 38. Network

```text
☐ Use HTTPS
☐ Use TLS
☐ Expose only required services
☐ Use NetworkPolicy where appropriate
☐ Avoid unnecessary public Routes
```

---

# 39. OpenShift

```text
☐ Understand SCC
☐ Follow least privilege
☐ Avoid cluster-admin
☐ Avoid privileged workloads
☐ Understand ServiceAccount permissions
☐ Understand namespace boundaries
```

---

# Part 12 – Troubleshooting

## 40. Check Pods

```cmd
oc get pods
```

---

## 41. Describe Pod

```cmd
oc describe pod <pod-name>
```

---

## 42. Check Logs

```cmd
oc logs <pod-name>
```

---

## 43. Check Previous Container Logs

```cmd
oc logs <pod-name> --previous
```

---

## 44. Check Events

```cmd
oc get events --sort-by=.lastTimestamp
```

---

## 45. Check Deployment

```cmd
oc describe deployment lesson24-app
```

---

## 46. Check Rollout

```cmd
oc rollout status deployment/lesson24-app
```

---

# Part 13 – Hands-On Challenges

## Challenge 1 – Secret

Create:

```text
lesson24-secret
```

with:

```text
USERNAME
PASSWORD
```

Verify the Secret exists.

---

## Challenge 2 – ConfigMap

Create:

```text
lesson24-config
```

with:

```text
APP_ENV
LOG_LEVEL
APP_VERSION
```

---

## Challenge 3 – Secure Deployment

Deploy:

```text
nginxinc/nginx-unprivileged:1.25
```

with:

```text
ConfigMap
Secret
allowPrivilegeEscalation=false
drop ALL capabilities
```

---

## Challenge 4 – Verify Non-root

Run:

```cmd
oc exec <pod-name> -- id
```

Explain why the application should not need root.

---

## Challenge 5 – Secret Volume

Mount the Secret at:

```text
/tmp/app-secret
```

and make the mount:

```text
readOnly
```

---

## Challenge 6 – TLS

Create:

```text
lesson24-route
```

using:

```text
Edge TLS
```

and:

```text
insecureEdgeTerminationPolicy: Redirect
```

Verify the application using HTTPS.

---

## Challenge 7 – Image Security

Answer:

```text
Why are we using nginxinc/nginx-unprivileged instead of the standard nginx image?
```

---

## Challenge 8 – SCC

Answer:

```text
What is SCC?

What does SCC control?

How is SCC different from securityContext?

Why can't we modify SCC in the Sandbox?
```

---

## Challenge 9 – Least Privilege

Explain:

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

## Challenge 10 – Production Checklist

Explain how you would secure a production OpenShift application using:

```text
Secret
ConfigMap
SCC
SecurityContext
Non-root
TLS
Trusted image
Least privilege
```

---

# Part 14 – Final Security Architecture

```text
                         PRODUCTION SECURITY
                                  |
          +-----------------------+-----------------------+
          |                       |                       |
          v                       v                       v
       Secrets                Containers              Network
          |                       |                       |
    No hard-code              Non-root                  TLS
    Least access              No escalation             HTTPS
    Rotation                  Drop caps                 Limited exposure
          |
          v
      Credentials
```

Container security:

```text
                 Container Image
                       |
                       v
                 Trusted Source
                       |
                       v
                  Fixed Version
                       |
                       v
                    Non-root
                       |
                       v
                  OpenShift SCC
                       |
                       v
                SecurityContext
                       |
                       v
                  Secure Pod
```

---

# 🔗 Connection With Previous Lessons

```text
Lesson 14
Secrets
   ↓
Lesson 15
Non-root / Security
   ↓
Lesson 17
Networking
   ↓
Lesson 21
CI/CD
   ↓
Lesson 22
GitOps
   ↓
Lesson 23
Environment Management
   ↓
Lesson 24
Production Security
```

Lesson 24 combines many of the concepts learned previously.

---

# 🧠 Key Concepts to Remember

## Secret

```text
Sensitive configuration
```

Examples:

```text
Password
Token
API Key
Database Credential
```

---

## ConfigMap

```text
Non-sensitive configuration
```

Examples:

```text
Environment
Log Level
Application Version
API URL
```

---

## SCC

```text
OpenShift security policy
```

Controls security characteristics of Pods.

---

## SecurityContext

```text
Container/workload security settings
```

Example:

```yaml
allowPrivilegeEscalation: false
```

---

## Non-root Container

```text
Application
   ↓
Non-root Process
   ↓
Reduced Privileges
   ↓
Smaller Attack Surface
```

---

## TLS

```text
HTTP
  ↓
HTTPS
  ↓
Encrypted Communication
```

---

## Image Security

```text
Trusted Image
      +
Maintained Image
      +
Known Version
      +
Non-root
      +
Vulnerability Scanning
```

---

# 🏁 Final Production Security Model

Remember:

```text
Secure Image
     +
Non-root
     +
SCC
     +
SecurityContext
     +
Secrets
     +
TLS
     +
Least Privilege
     =
More Secure OpenShift Application
```

---

# ✅ Lesson 24 Completion Checklist

- [ ] Understand Secrets
- [ ] Create a Secret
- [ ] Understand ConfigMap vs Secret
- [ ] Consume Secret as environment variables
- [ ] Consume Secret as a volume
- [ ] Understand Base64 is not encryption
- [ ] Understand SCC
- [ ] Understand SCC vs securityContext
- [ ] Understand Sandbox SCC limitations
- [ ] Run a non-root container
- [ ] Verify container UID
- [ ] Understand `allowPrivilegeEscalation`
- [ ] Drop unnecessary Linux capabilities
- [ ] Understand least privilege
- [ ] Use an unprivileged NGINX image
- [ ] Understand image security
- [ ] Avoid `latest` in production
- [ ] Understand immutable image references
- [ ] Create an HTTPS Route
- [ ] Understand Edge TLS
- [ ] Understand TLS termination
- [ ] Understand Secret volumes
- [ ] Practice filesystem permission testing
- [ ] Practice security troubleshooting
- [ ] Understand production security best practices
- [ ] Avoid NGINX non-root permission problems encountered in previous lessons

---

# 🎯 Lesson 24 Goal


> **Production security in OpenShift means protecting credentials, restricting container privileges, using trusted images, enforcing least privilege, securing network communication with TLS, and understanding how OpenShift SCC and workload security settings work together.**

The final model:

```text
Secure Image
      ↓
Non-root Container
      ↓
SCC
      ↓
SecurityContext
      ↓
Secrets
      ↓
TLS
      ↓
Least Privilege
      ↓
Secure OpenShift Application
```

**Lesson 24 Topic: Production Security – Secrets → SCC → Non-root Containers → SecurityContext → TLS → Image Security → Least Privilege → Production Best Practices**
