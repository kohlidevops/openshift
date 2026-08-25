# OpenShift Learning – Lesson 19: Templates & Parameters

## 🚀 Lesson 19: OpenShift Templates, Parameters & Reusable Deployments

> **Environment Note:** I am using an OpenShift Developer Sandbox with restricted/non-root container execution. During earlier lessons, the standard `nginx:1.25` configuration caused `CrashLoopBackOff` because nginx tried to write to `/var/cache/nginx` and `/etc/nginx`. Therefore, this lesson uses a custom nginx configuration through a ConfigMap, port `8080`, and `/tmp` for writable temporary files. **Do not use the old plain-nginx Template from the earlier version of this lesson.**

---

## 🎯 Learning Objectives

By the end of this lesson, I should understand:

- What an OpenShift Template is
- Why Templates are useful
- What Template parameters are
- Parameter substitution
- `${PARAMETER}` syntax
- `oc process`
- Processing a Template
- Applying processed resources
- Default parameter values
- Overriding parameter values
- Using one Template for multiple environments
- Using ConfigMaps inside a Template
- Creating Deployments from Templates
- Basic Template troubleshooting
- Difference between normal YAML and a Template

---

# 1. What Is an OpenShift Template?

An OpenShift Template is a reusable definition of application resources.

Instead of creating separate YAML files for every environment, we can create one Template and provide different parameter values.

Conceptually:

```text
Template
   |
   +---- ConfigMap
   |
   +---- Deployment
   |
   +---- Parameters
```

---

# 2. Why Use Templates?

Suppose the same application needs to be deployed to:

```text
Development
QA
Production
```

Some values may change:

```text
Application name
Image
Replica count
```

Instead of creating three different YAML files:

```text
lesson19-dev.yaml
lesson19-qa.yaml
lesson19-prod.yaml
```

we can use:

```text
lesson19-template.yaml
```

with parameters.

Conceptually:

```text
                 Template
                    |
       +------------+------------+
       |            |            |
      DEV           QA          PROD
       |            |            |
    Parameters  Parameters  Parameters
```

---

# 3. Template Parameters

A parameter is a variable inside a Template.

Example:

```yaml
parameters:
  - name: APP_NAME
    description: Application name
    value: lesson19-app
```

Then inside the Template:

```yaml
name: ${APP_NAME}
```

The parameter connects the two.

```text
APP_NAME
    ↓
lesson19-app

${APP_NAME}
    ↓
lesson19-app
```

---

# 4. How `${APP_NAME}` Is Replaced

Suppose the Template contains:

```yaml
parameters:
  - name: APP_NAME
    value: lesson19-app
```

and:

```yaml
metadata:
  name: ${APP_NAME}
```

When we run:

```cmd
oc process lesson19-template
```

OpenShift processes the Template:

```text
APP_NAME
   ↓
Default value = lesson19-app
   ↓
Find ${APP_NAME}
   ↓
Replace ${APP_NAME}
   ↓
lesson19-app
```

So the generated YAML contains:

```yaml
metadata:
  name: lesson19-app
```

---

# 5. Override a Parameter

The default value can be overridden.

For example:

```cmd
oc process lesson19-template -p APP_NAME=my-web-app
```

Now:

```text
${APP_NAME}
```

becomes:

```text
my-web-app
```

instead of:

```text
lesson19-app
```

Easy memory:

```text
Default:
APP_NAME = lesson19-app

-p APP_NAME=my-web-app

Result:
APP_NAME = my-web-app
```

---

# 6. OpenShift-Compatible nginx Configuration

We previously faced this problem:

```text
CrashLoopBackOff
```

with:

```text
nginx:1.25
```

The important error was:

```text
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

We also saw:

```text
sed: couldn't open temporary file /etc/nginx/... Permission denied
```

### Why?

OpenShift runs containers under restricted/non-root security settings.

The standard nginx configuration expects writable locations such as:

```text
/var/cache/nginx
/etc/nginx
```

Those locations are not writable by the restricted runtime user.

Therefore, **do not modify `/etc/nginx/nginx.conf` using `sed` during container startup.**

Instead, this lesson uses:

```text
ConfigMap
    ↓
Custom nginx.conf
    ↓
Mount as nginx.conf
    ↓
Listen on 8080
    ↓
Temporary files → /tmp
    ↓
Logs → stdout/stderr
```

---

# 7. Create the Correct Lesson 19 Template

Create:

```text
lesson19-template.yaml
```

Use this complete Template:

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: lesson19-template
parameters:
  - name: APP_NAME
    description: Application name
    value: lesson19-app

  - name: IMAGE
    description: Container image
    value: nginx:1.25

  - name: REPLICAS
    description: Number of application replicas
    value: "1"

objects:

  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ${APP_NAME}-nginx-config
    data:
      nginx.conf: |
        worker_processes auto;
        pid /tmp/nginx.pid;

        events {
          worker_connections 1024;
        }

        http {
          access_log /dev/stdout;
          error_log /dev/stderr;

          client_body_temp_path /tmp/client_temp;
          proxy_temp_path /tmp/proxy_temp;
          fastcgi_temp_path /tmp/fastcgi_temp;
          uwsgi_temp_path /tmp/uwsgi_temp;
          scgi_temp_path /tmp/scgi_temp;

          server {
            listen 8080;
            server_name _;

            location / {
              root /usr/share/nginx/html;
              index index.html;
            }
          }
        }

  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${APP_NAME}
    spec:
      replicas: ${{REPLICAS}}
      selector:
        matchLabels:
          app: ${APP_NAME}
      template:
        metadata:
          labels:
            app: ${APP_NAME}
        spec:
          containers:
            - name: web
              image: ${IMAGE}
              command:
                - nginx
                - -c
                - /etc/nginx/nginx.conf
                - -g
                - daemon off;
              ports:
                - containerPort: 8080
              volumeMounts:
                - name: nginx-config
                  mountPath: /etc/nginx/nginx.conf
                  subPath: nginx.conf
                - name: nginx-tmp
                  mountPath: /tmp
          volumes:
            - name: nginx-config
              configMap:
                name: ${APP_NAME}-nginx-config
            - name: nginx-tmp
              emptyDir: {}
```

### Why this YAML works with OpenShift

```text
nginx.conf
    ↓
Provided by ConfigMap
    ↓
No sed modification required
```

```text
nginx temporary files
    ↓
/tmp
    ↓
Writable
```

```text
nginx
    ↓
Port 8080
    ↓
Non-root compatible
```

```text
logs
    ↓
stdout/stderr
    ↓
oc logs
```

---

# 8. Apply the Template

Run:

```cmd
oc apply -f lesson19-template.yaml
```

Check:

```cmd
oc get templates
```

Expected:

```text
NAME
lesson19-template
```

---

# 9. Describe the Template

Run:

```cmd
oc describe template lesson19-template
```

Look for:

```text
Parameters
Objects
```

You should see:

```text
APP_NAME
IMAGE
REPLICAS
```

---

# 10. Process the Template

This is one of the most important commands in Lesson 19.

Run:

```cmd
oc process lesson19-template
```

This processes the Template using the default parameter values.

Conceptually:

```text
Template
    +
Parameters
    ↓
oc process
    ↓
Generated YAML
```

---

# 11. View Generated YAML

Run:

```cmd
oc process lesson19-template -o yaml
```

You should see generated resources.

The parameter:

```text
${APP_NAME}
```

should become:

```text
lesson19-app
```

The parameter:

```text
${IMAGE}
```

should become:

```text
nginx:1.25
```

The parameter:

```text
${REPLICAS}
```

should become:

```text
1
```

---

# 12. Important: `oc process` Does Not Deploy

This command:

```cmd
oc process lesson19-template
```

only generates the resources.

It does not create the Deployment.

Think:

```text
oc process
    ↓
Generate YAML
```

To create the resources:

```cmd
oc process lesson19-template | oc apply -f -
```

This means:

```text
Template
    ↓
oc process
    ↓
Generated YAML
    ↓
oc apply
    ↓
OpenShift resources
```

---

# 13. Deploy Using Default Parameters

Run:

```cmd
oc process lesson19-template | oc apply -f -
```

Check:

```cmd
oc get deployment
```

You should see:

```text
lesson19-app
```

Check Pods:

```cmd
oc get pods
```

Expected:

```text
lesson19-app-xxxxx   1/1   Running
```

---

# 14. Verify the Application

Check the Pod:

```cmd
oc get pods
```

Then:

```cmd
oc describe pod <pod-name>
```

Look for:

```text
Container Port: 8080
```

Check logs:

```cmd
oc logs <pod-name>
```

You should not see:

```text
/var/cache/nginx/client_temp failed
```

You should not see:

```text
Permission denied
```

---

# 15. Verify Deployment

Run:

```cmd
oc get deployment lesson19-app
```

Expected:

```text
NAME           READY   UP-TO-DATE   AVAILABLE
lesson19-app   1/1     1            1
```

Check rollout:

```cmd
oc rollout status deployment/lesson19-app
```

Expected:

```text
deployment "lesson19-app" successfully rolled out
```

---

# 16. Process With Custom APP_NAME

Run:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-web
```

Look at the generated YAML.

You should see:

```text
lesson19-web
```

instead of:

```text
lesson19-app
```

---

# 17. Process With Custom Image

Run:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-web -p IMAGE=nginx:1.27
```

The generated resources should contain:

```text
Application:
lesson19-web

Image:
nginx:1.27
```

---

# 18. Process With Custom Replica Count

Run:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-scaled -p IMAGE=nginx:1.27 -p REPLICAS=3
```

Look at the generated YAML.

You should find:

```yaml
replicas: 3
```

---

# 19. Deploy Custom Parameters

Run:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-scaled -p IMAGE=nginx:1.27 -p REPLICAS=3 | oc apply -f -
```

Check:

```cmd
oc get deployment
```

Then:

```cmd
oc get pods
```

You should have:

```text
lesson19-scaled-xxxxx   1/1   Running
lesson19-scaled-xxxxx   1/1   Running
lesson19-scaled-xxxxx   1/1   Running
```

---

# 20. Multiple Applications From One Template

Now we can use the same Template multiple times.

### Development

```cmd
oc process lesson19-template -p APP_NAME=lesson19-dev -p IMAGE=nginx:1.25 -p REPLICAS=1 | oc apply -f -
```

### QA

```cmd
oc process lesson19-template -p APP_NAME=lesson19-qa -p IMAGE=nginx:1.27 -p REPLICAS=2 | oc apply -f -
```

### Production

```cmd
oc process lesson19-template -p APP_NAME=lesson19-prod -p IMAGE=nginx:1.27 -p REPLICAS=3 | oc apply -f -
```

Now:

```text
Same Template
      |
      +---- DEV
      |      1 replica
      |
      +---- QA
      |      2 replicas
      |
      +---- PROD
             3 replicas
```

---

# 21. Check All Applications

Run:

```cmd
oc get deployment
```

You should see something similar to:

```text
lesson19-app
lesson19-dev
lesson19-qa
lesson19-prod
```

Check Pods:

```cmd
oc get pods
```

---

# 22. Template Parameters

Our Template contains:

```yaml
parameters:
  - name: APP_NAME
    value: lesson19-app

  - name: IMAGE
    value: nginx:1.25

  - name: REPLICAS
    value: "1"
```

These are the variables.

Inside the Template:

```text
${APP_NAME}
${IMAGE}
${REPLICAS}
```

OpenShift substitutes the values during:

```cmd
oc process
```

---

# 23. Parameter Substitution Example

Template:

```yaml
metadata:
  name: ${APP_NAME}
```

Default:

```text
APP_NAME = lesson19-app
```

Result:

```yaml
metadata:
  name: lesson19-app
```

Custom:

```cmd
oc process lesson19-template -p APP_NAME=my-application
```

Result:

```yaml
metadata:
  name: my-application
```

---

# 24. Normal YAML vs Template

### Normal YAML

```yaml
metadata:
  name: lesson19-app
```

The value is fixed.

### Template

```yaml
metadata:
  name: ${APP_NAME}
```

The value is configurable.

Easy memory:

```text
Normal YAML
    ↓
Fixed values

Template
    ↓
Reusable variables
```

---

# 25. Check Template Parameters

Run:

```cmd
oc describe template lesson19-template
```

Look for:

```text
Parameters
```

You should see:

```text
APP_NAME
IMAGE
REPLICAS
```

---

# 26. Get the Template YAML

Run:

```cmd
oc get template lesson19-template -o yaml
```

This allows you to inspect the Template stored in OpenShift.

---

# 27. Inspect Before Deploying

A good practice is to process the Template first:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-test -p IMAGE=nginx:1.27 -p REPLICAS=2
```

Review the generated YAML.

Then deploy:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-test -p IMAGE=nginx:1.27 -p REPLICAS=2 | oc apply -f -
```

Think:

```text
process
   ↓
Review
   ↓
apply
```

---

# 28. Template Troubleshooting

If processing fails:

```cmd
oc process lesson19-template
```

Check the Template:

```cmd
oc describe template lesson19-template
```

Get the YAML:

```cmd
oc get template lesson19-template -o yaml
```

Look for:

```text
Parameter name
Parameter usage
YAML indentation
Object definitions
```

---

# 29. Common Template Problems

## Problem 1 – Parameter Not Defined

You use:

```text
${APP_NAME}
```

but don't define:

```yaml
parameters:
  - name: APP_NAME
```

Make sure the parameter exists.

---

## Problem 2 – Wrong Parameter Name

Defined:

```text
APP_NAME
```

but used:

```text
${APPLICATION_NAME}
```

These are different names.

---

## Problem 3 – YAML Indentation

YAML indentation matters.

Always test:

```cmd
oc process lesson19-template
```

before applying.

---

## Problem 4 – Resource Already Exists

If you reuse the same `APP_NAME`, OpenShift may update an existing resource.

Check:

```cmd
oc get deployment
```

Use a different application name for practice when needed.

---

# 30. nginx Troubleshooting

If you see:

```text
CrashLoopBackOff
```

check:

```cmd
oc get pods
```

Then:

```cmd
oc logs <pod-name>
```

If the Pod restarted:

```cmd
oc logs <pod-name> --previous
```

Check Events:

```cmd
oc describe pod <pod-name>
```

Look for:

```text
Permission denied
```

### If you see this:

```text
/var/cache/nginx/client_temp
```

then you are probably using the **old/broken nginx Deployment or Template**.

The corrected Lesson 19 Template must contain:

```text
ConfigMap
+
custom nginx.conf
+
8080
+
/tmp
```

Do not fix it using:

```text
sed
```

against `/etc/nginx/nginx.conf`.

---

# 31. Verify the Correct nginx Configuration

After deployment, run:

```cmd
oc describe pod <pod-name>
```

Check that:

```text
Port: 8080
```

and the Pod has the ConfigMap mounted.

You can also check:

```cmd
oc get configmap
```

You should see:

```text
lesson19-app-nginx-config
```

or the corresponding ConfigMap for your `APP_NAME`.

---


# 🧠 Final Memory Trick

Remember:

```text
Template
    ↓
Reusable Definition
```

```text
Parameter
    ↓
Variable
```

```text
oc process
    ↓
Template + Parameters
    ↓
Generated YAML
```

```text
oc process
    |
    v
oc apply
    |
    v
OpenShift Resources
```

Complete flow:

```text
             Template
                 |
                 +---- Parameters
                 |
                 +---- ConfigMap
                 |
                 +---- Deployment
                 |
                 v
             oc process
                 |
                 v
           Generated YAML
                 |
                 v
              oc apply
                 |
                 v
            Application
```

OpenShift-compatible nginx flow:

```text
ConfigMap
    ↓
nginx.conf
    ↓
Port 8080
    ↓
/tmp
    ↓
Non-root compatible nginx
```

---

# 🔧 Important Commands

List Templates:

```cmd
oc get templates
```

Apply Template definition:

```cmd
oc apply -f lesson19-template.yaml
```

Describe Template:

```cmd
oc describe template lesson19-template
```

Get Template YAML:

```cmd
oc get template lesson19-template -o yaml
```

Process Template:

```cmd
oc process lesson19-template
```

Process as YAML:

```cmd
oc process lesson19-template -o yaml
```

Process with one parameter:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-web
```

Process with multiple parameters:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-web -p IMAGE=nginx:1.27 -p REPLICAS=2
```

Process and apply:

```cmd
oc process lesson19-template | oc apply -f -
```

Process custom values and apply:

```cmd
oc process lesson19-template -p APP_NAME=lesson19-custom -p IMAGE=nginx:1.27 -p REPLICAS=2 | oc apply -f -
```

Check Deployments:

```cmd
oc get deployment
```

Check Pods:

```cmd
oc get pods
```

Check ConfigMaps:

```cmd
oc get configmap
```

Check rollout:

```cmd
oc rollout status deployment/<deployment-name>
```

Check logs:

```cmd
oc logs <pod-name>
```

Check previous logs:

```cmd
oc logs <pod-name> --previous
```

Check Pod details:

```cmd
oc describe pod <pod-name>
```

---

# ✅ Lesson 19 Completion Checklist

- [ ] Understand OpenShift Templates
- [ ] Understand why Templates are useful
- [ ] Understand Template parameters
- [ ] Understand `${PARAMETER}` substitution
- [ ] Understand default parameter values
- [ ] Override parameter values
- [ ] Create a Template
- [ ] Apply a Template definition
- [ ] List Templates
- [ ] Describe a Template
- [ ] Process a Template
- [ ] Generate YAML using `oc process`
- [ ] Process and apply resources
- [ ] Create a Deployment from a Template
- [ ] Create multiple applications from one Template
- [ ] Use `APP_NAME` parameter
- [ ] Use `IMAGE` parameter
- [ ] Use `REPLICAS` parameter
- [ ] Understand ConfigMap inside a Template
- [ ] Understand OpenShift-compatible nginx configuration
- [ ] Understand why the original nginx configuration failed
- [ ] Use port 8080
- [ ] Use `/tmp` for writable nginx temporary files
- [ ] Verify application Pods are Running
- [ ] Troubleshoot Template processing
- [ ] Troubleshoot Pod logs and Events
- [ ] Understand Template vs normal YAML
- [ ] Understand how Templates can support multiple environments

---

# 🏁 Lesson 19 Goal


> **An OpenShift Template is a reusable definition of application resources. Parameters allow me to change values such as application name, image and replica count without creating separate YAML files. `oc process` substitutes the parameter values and generates deployable resource definitions.**

The core model:

```text
OpenShift Template
       |
       +---- Parameters
       |
       +---- ConfigMap
       |
       +---- Deployment
       |
       v
   oc process
       |
       v
Generated Resources
       |
       v
   oc apply
       |
       v
Running Application
```

**Lesson 19 Topic: OpenShift Templates & Parameters – Reusable and OpenShift-Compatible Application Deployment**
