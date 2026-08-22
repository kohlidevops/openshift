# OpenShift Learning – Lesson 1

## Lesson 1: OpenShift Fundamentals and First Application Deployment

This is my **OpenShift hands-on learning journey**.

In Lesson 1, I learned the basic OpenShift concepts, accessed an OpenShift Developer Sandbox, used the OpenShift Web Console and `oc` CLI, deployed my first application, created a Service, and exposed the application externally using an OpenShift Route.

---

## 🎯 Learning Objectives

By completing this lesson, I learned:

* What is OpenShift?
* OpenShift vs Kubernetes
* OpenShift Projects
* OpenShift Web Console
* OpenShift CLI (`oc`)
* Pods
* Deployments
* Services
* Routes
* How to deploy an application
* How to expose an application externally
* How to inspect OpenShift resources using CLI

---

# 1. What is OpenShift?

OpenShift is Red Hat's enterprise container application platform built on Kubernetes.

It provides Kubernetes capabilities along with additional platform features such as:

* Web Console
* `oc` CLI
* Projects
* Routes
* ImageStreams
* Builds
* S2I
* Security Context Constraints (SCC)
* Operators
* Enterprise application management

### Simple understanding

```text
OpenShift
    |
    +-- Kubernetes
    |
    +-- Web Console
    +-- Projects
    +-- Routes
    +-- Builds
    +-- ImageStreams
    +-- S2I
    +-- SCC
    +-- Operators
```

---

# 2. OpenShift vs Kubernetes

OpenShift is built on Kubernetes.

Kubernetes provides core container orchestration capabilities such as:

* Pods
* Deployments
* Services
* ConfigMaps
* Secrets
* Persistent Volumes
* RBAC

OpenShift adds additional enterprise platform capabilities and developer/admin tooling.

### Basic comparison

| Kubernetes           | OpenShift                        |
| -------------------- | -------------------------------- |
| `kubectl`            | `oc`                             |
| Namespace            | Project                          |
| Service              | Service                          |
| Ingress              | Route                            |
| Kubernetes resources | Kubernetes + OpenShift resources |
| Container builds     | BuildConfig / S2I                |
| Container images     | ImageStreams                     |
| Kubernetes security  | SCC + RBAC                       |
| Kubernetes UI        | OpenShift Web Console            |

---

# 3. OpenShift Developer Sandbox

For this learning exercise, I used the **Red Hat OpenShift Developer Sandbox**.

The Sandbox provides a hosted OpenShift environment that allows developers to practice OpenShift without installing and managing a local OpenShift cluster.

### My OpenShift environment

```text
OpenShift Developer Sandbox
        |
        +-- Project
              |
              +-- Application
              +-- Deployment
              +-- Pod
              +-- Service
              +-- Route
```

---

# 4. OpenShift Project

My current OpenShift project is:

```text
lakshminarayananredh-dev
```

A project provides an isolated workspace for applications and resources.

Example:

```text
OpenShift Cluster
       |
       +-- Project A
       |      |
       |      +-- Pods
       |      +-- Deployments
       |      +-- Services
       |      +-- Secrets
       |
       +-- Project B
              |
              +-- Pods
              +-- Deployments
```

---

# 5. OpenShift CLI – `oc`

The OpenShift command-line tool is called:

```bash
oc
```

It is similar to Kubernetes `kubectl`.

### Check OpenShift CLI version

```bash
oc version
```

### Check logged-in user

```bash
oc whoami
```

### Check current project

```bash
oc project
```

### List available projects

```bash
oc projects
```

### Display project status

```bash
oc status
```

---

# 6. Deploy My First Application

I deployed the ParksMap sample application using an existing container image.

### Command

```bash
oc new-app --image=quay.io/openshiftroadshow/parksmap:latest
```

This command created the required application resources.

---

# 7. Verify the Pod

After deploying the application:

```bash
oc get pods
```

Example:

```text
NAME                         READY   STATUS    RESTARTS   AGE
parksmap-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
```

The application runs inside a Kubernetes/OpenShift Pod.

---

# 8. Check the Deployment

```bash
oc get deployments
```

A Deployment manages the application Pods.

The basic relationship is:

```text
Deployment
     |
     +-- Pod
           |
           +-- Container
```

---

# 9. Check the Service

```bash
oc get svc
```

The Service provides internal network access to the application Pods.

Architecture:

```text
Client inside cluster
        |
        v
     Service
        |
        v
       Pod
```

---

# 10. Expose the Application

The application was running inside the OpenShift cluster, but it was not yet accessible externally.

I created an OpenShift Route:

```bash
oc expose service parksmap
```

Then verified the Route:

```bash
oc get route
```

The Route provided a hostname that I could open in a web browser.

---

# 11. Final Application Architecture

After exposing the application, the architecture became:

```text
                    Internet
                       |
                       v
                +-------------+
                | OpenShift   |
                |   Route     |
                +-------------+
                       |
                       v
                +-------------+
                |   Service   |
                +-------------+
                       |
                       v
                +-------------+
                | Deployment  |
                +-------------+
                       |
                       v
                +-------------+
                |     Pod     |
                +-------------+
                       |
                       v
                +-------------+
                | Container   |
                |  ParksMap   |
                +-------------+
```

---

# 12. Important Commands Learned

| Command              | Purpose                                  |
| -------------------- | ---------------------------------------- |
| `oc version`         | Display OpenShift CLI and server version |
| `oc whoami`          | Display current logged-in user           |
| `oc project`         | Display current project                  |
| `oc projects`        | List accessible projects                 |
| `oc status`          | Display project/application status       |
| `oc get pods`        | List Pods                                |
| `oc get deployments` | List Deployments                         |
| `oc get svc`         | List Services                            |
| `oc get route`       | List Routes                              |
| `oc new-app`         | Create an application                    |
| `oc expose service`  | Create a Route for a Service             |
| `oc logs`            | View container/application logs          |
| `oc describe pod`    | Display detailed Pod information         |

---

# 13. Troubleshooting Commands

I also learned that these commands are important when troubleshooting OpenShift applications.

### View Pods

```bash
oc get pods
```

### View Pod details

```bash
oc describe pod <pod-name>
```

### View application logs

```bash
oc logs <pod-name>
```

### View Services

```bash
oc get svc
```

### View Routes

```bash
oc get route
```

### View application status

```bash
oc status
```

---

# 14. Web Console

I also explored the OpenShift Web Console.

The console provides a graphical interface to manage applications and resources.

Important areas include:

* Projects
* Topology
* Pods
* Deployments
* Services
* Routes
* Logs
* Events
* ConfigMaps
* Secrets

### Developer Perspective

The Developer perspective is useful for application developers and DevOps engineers to deploy and manage workloads.

### Administrator Perspective

The Administrator perspective provides access to cluster-level administration and infrastructure-related resources.

---

# 15. Key Concepts Learned

### Pod

The smallest deployable unit in Kubernetes/OpenShift.

A Pod runs one or more containers.

```text
Pod
 |
 +-- Container
```

### Deployment

Manages application Pods and maintains the desired number of replicas.

```text
Deployment
    |
    +-- Pod
    +-- Pod
    +-- Pod
```

### Service

Provides stable internal networking to Pods.

```text
Service
   |
   +-- Pod
   +-- Pod
   +-- Pod
```

### Route

OpenShift Route exposes a Service externally using a hostname.

```text
Internet
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

### Project

Provides a logical workspace/isolation boundary for applications and resources.

---

# 16. Kubernetes Knowledge vs OpenShift Knowledge


```text
Kubernetes                 OpenShift
-----------                ----------
kubectl             --->   oc
Namespace           --->   Project
Ingress             --->   Route
Container Image     --->   ImageStream
Build               --->   BuildConfig / S2I
Kubernetes security --->   RBAC + SCC
```

---


# 17. What I Learned

The most important concept from Lesson 1 is:

```text
OpenShift Project
       |
       v
   Deployment
       |
       v
      Pod
       |
       v
   Container
       |
       ^
       |
    Service
       ^
       |
     Route
       ^
       |
    Internet
```

A simple way to remember it:

> **Deployment manages Pods, Service provides internal access, and Route provides external access.**

---



## Useful References

* Red Hat OpenShift Documentation: https://docs.redhat.com/en/documentation/openshift_container_platform/
* OpenShift Developer Sandbox: https://developers.redhat.com/developer-sandbox
* OpenShift CLI (`oc`): https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/cli_tools/openshift-cli-oc
