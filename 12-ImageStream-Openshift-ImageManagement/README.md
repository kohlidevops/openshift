# OpenShift Learning – Lesson 12: ImageStreams & OpenShift Image Management

## 🎯 Objectives

In this lesson, I learned and practiced:

- Container Images
- ImageStreams
- Image vs ImageStream
- ImageStreamTags
- Importing external images
- Image version management
- Deploying applications using ImageStreams
- ImagePullSecrets
- ImagePullBackOff troubleshooting
- ImageStream troubleshooting
- Real-world image flow in OpenShift

---
## 1. What is a Container Image?

A container image contains everything required to run an application.

```text
Container Image
     |
     +---- Application
     +---- Libraries
     +---- Runtime
     +---- Configuration
```

Example:

```text
nginxinc/nginx-unprivileged:1.27
```

```text
Repository → nginxinc/nginx-unprivileged
Tag        → 1.27
```

---
## 2. What is an ImageStream?

An **ImageStream** is an OpenShift resource used to manage and track container images.

```text
External Registry
       |
       v
   ImageStream
       |
       v
 ImageStreamTag
       |
       v
 Deployment
       |
       v
     Pod
```

An ImageStream provides a way for OpenShift to track image references and versions.

---
## 3. Image vs ImageStream

| Image | ImageStream |
|---|---|
| Container image | OpenShift resource |
| Contains application layers | Tracks image references/versions |
| Stored in a registry | Exists as an OpenShift API resource |
| Used to run containers | Helps manage image versions |

Easy way to remember:

```text
Image       → Actual container image

ImageStream → OpenShift tracking/reference mechanism
```

---
## 4. Check Existing ImageStreams

```bash
oc get imagestreams
```

Or:

```bash
oc get is
```

Inspect an ImageStream:

```bash
oc describe is <image-stream-name>
```

---
## 5. Create an ImageStream

```bash
oc create imagestream lesson12-nginx
```

Check:

```bash
oc get imagestream
```

Or:

```bash
oc get is
```

---
## 6. Import an External Image

Import NGINX:

```bash
oc import-image lesson12-nginx:1.27 \
  --from=nginxinc/nginx-unprivileged:1.27 \
  --confirm
```

The flow is:

```text
External Registry
       |
       v
nginxinc/nginx-unprivileged:1.27
       |
       v
lesson12-nginx:1.27
```

---
## 7. Inspect ImageStream

```bash
oc describe is lesson12-nginx
```

Check ImageStream YAML:

```bash
oc get is lesson12-nginx -o yaml
```

---
## 8. ImageStreamTag

An ImageStreamTag identifies a specific tag/version in an ImageStream.

Example:

```text
lesson12-nginx:1.27
```

Structure:

```text
ImageStream : Tag
```

List ImageStreamTags:

```bash
oc get istag
```

Check a specific tag:

```bash
oc get istag lesson12-nginx:1.27
```

Or:

```bash
oc describe istag lesson12-nginx:1.27
```

---
## 9. Deploy Using an ImageStream

OpenShift can create an application from an ImageStream:

```bash
oc new-app lesson12-nginx:1.27 --name=lesson12-web
```

Check:

```bash
oc get deployment
oc get pods
```

Expected:

```text
lesson12-web-xxxxx   1/1   Running
lesson12-web-yyyyy   1/1   Running
```

---
## 10. ImageStream Deployment Flow

```text
ImageStream
     |
     v
ImageStreamTag
     |
     v
Deployment
     |
     v
Pod
     |
     v
Container
```

---
## 11. Create a Service

```bash
oc expose deployment lesson12-web --port=8080
```

Check:

```bash
oc get svc
```

---
## 12. Test the Application

Create a temporary client:

```bash
oc run lesson12-client --image=curlimages/curl --command -- sleep 3600
```

Test the Service:

```bash
oc exec lesson12-client -- curl http://lesson12-web:8080
```

---
## 13. Create a Route

Expose the Service:

```bash
oc expose service lesson12-web
```

Check:

```bash
oc get route
```

Get the hostname:

```bash
oc get route lesson12-web -o jsonpath="{.spec.host}"
```

Open the Route hostname in a browser.

---
## 14. ImageStream Versioning

An ImageStream can track different image versions:

```text
lesson12-nginx
     |
     +---- 1.25
     |
     +---- 1.26
     |
     +---- 1.27
```

Import another version:

```bash
oc import-image lesson12-nginx:1.26 --from=nginxinc/nginx-unprivileged:1.26 --confirm
```

Check:

```bash
oc describe is lesson12-nginx
```

List tags:

```bash
oc get istag
```

---
## 15. Image Version Selection

For example:

```text
Deployment
    |
    v
lesson12-nginx:1.27
```

Changing the application to:

```text
lesson12-nginx:1.26
```

selects another image version.

---
## 16. ImageStreamTag vs Docker Image Tag

Docker image:

```text
nginxinc/nginx-unprivileged:1.27
```

OpenShift ImageStreamTag:

```text
lesson12-nginx:1.27
```

The Docker image tag refers to an image in a registry.

The ImageStreamTag refers to a tag tracked by an ImageStream in an OpenShift project.

---
## 17. ImagePullSecrets

Private container registries may require authentication.

Examples:

```text
Private Docker Hub repository
Private GitHub Container Registry
Private enterprise registry
Private Quay registry
```

OpenShift can use an `imagePullSecret` to authenticate to the registry.

Conceptually:

```text
Deployment
    |
    v
Private Registry
    |
    X
Authentication Required
    |
    v
ImagePullSecret
    |
    v
Image Pull
```

Check existing Secrets:

```bash
oc get secrets
```

A Deployment can reference a registry Secret:

```yaml
imagePullSecrets:
  - name: my-registry-secret
```

---
## 18. ImagePullBackOff Troubleshooting

A common error is:

```text
ImagePullBackOff
```

Check:

```bash
oc get pods
```

Then:

```bash
oc describe pod <pod-name>
```

Check the Events section.

Common causes:

```text
Wrong image name
Wrong tag
Private registry authentication failure
Image does not exist
Registry unavailable
ImagePullSecret problem
```

---
## 19. ImageStream Troubleshooting

Check ImageStreams:

```bash
oc get is
```

Describe:

```bash
oc describe is lesson12-nginx
```

Check tags:

```bash
oc get istag
```

Check a specific tag:

```bash
oc describe istag lesson12-nginx:1.27
```

Check Deployment:

```bash
oc describe deployment lesson12-web
```

Check Pod:

```bash
oc describe pod <pod-name>
```

Check Events:

```bash
oc get events --sort-by=.lastTimestamp
```

---
## 20. Real-World OpenShift Image Flow

A typical OpenShift application flow:

```text
Developer
    |
    v
Git Repository
    |
    v
Build
    |
    v
Container Image
    |
    v
Container Registry
    |
    v
ImageStream
    |
    v
Deployment
    |
    v
Pods
    |
    v
Service
    |
    v
Route
    |
    v
Users
```

---
## Final architecture:

```text
External Registry
       |
       v
ImageStream
       |
       v
ImageStreamTag
       |
       v
Deployment
       |
       v
Pods
       |
       v
Service
       |
       v
Route
       |
       v
Browser
```

---
## 22. Interview Questions

1. What is an ImageStream?
2. What is the difference between an Image and an ImageStream?
3. What is an ImageStreamTag?
4. Why does OpenShift use ImageStreams?
5. How do you create an ImageStream?
6. How do you import an external image?
7. What is `oc import-image`?
8. How do you list ImageStreams?
9. How do you list ImageStreamTags?
10. How can you deploy an application using an ImageStream?
11. What is an ImagePullSecret?
12. When do you need an ImagePullSecret?
13. What causes `ImagePullBackOff`?
14. How do you troubleshoot image pull failures?
15. How can ImageStreams help with image version management?

---
## 23. Useful Commands

### ImageStreams

```bash
oc get imagestream
oc get is
oc create imagestream <name>
oc describe is <name>
```

### Import Image

```bash
oc import-image <is-name>:<tag> \
  --from=<registry>/<image>:<tag> \
  --confirm
```

### ImageStreamTags

```bash
oc get istag
oc describe istag <is-name>:<tag>
```

### Application

```bash
oc new-app <image-stream>:<tag>
oc get deployment
oc get pods
```

### Troubleshooting

```bash
oc describe pod <pod-name>
oc logs <pod-name>
oc get events --sort-by=.lastTimestamp
```

### Registry Secrets

```bash
oc get secrets
```

---
## 🧹 Cleanup

Delete the application:

```bash
oc delete deployment lesson12-web
```

Delete the Service:

```bash
oc delete service lesson12-web
```

Delete the Route:

```bash
oc delete route lesson12-web
```

Delete the ImageStream:

```bash
oc delete imagestream lesson12-nginx
```

Delete the temporary client:

```bash
oc delete pod lesson12-client
```

---
## ✅ Lesson 12 Completion Checklist

- [x] Understand container images
- [x] Understand ImageStreams
- [x] Understand Image vs ImageStream
- [x] Understand ImageStreamTags
- [x] Create an ImageStream
- [x] Import an external image
- [x] List ImageStreams
- [x] Inspect ImageStream details
- [x] List ImageStreamTags
- [x] Deploy using an ImageStream
- [x] Understand image versioning
- [x] Import multiple image versions
- [x] Understand ImagePullSecrets
- [x] Understand ImagePullBackOff
- [x] Troubleshoot image pull failures
- [x] Understand the ImageStream deployment workflow

---
## 🧠 Final Memory Trick

```text
Image
  ↓
Actual container image

ImageStream
  ↓
OpenShift resource that tracks images

ImageStreamTag
  ↓
Specific image version/tag
```

Main workflow:

```text
Registry
   |
   v
ImageStream
   |
   v
ImageStreamTag
   |
   v
Deployment
   |
   v
Pod
```

---
Key concept:

> An ImageStream is an OpenShift resource used to track and manage references to container images and their versions.

Complete application flow:

```text
Container Registry
       |
       v
   ImageStream
       |
       v
 ImageStreamTag
       |
       v
   Deployment
       |
       v
      Pods
       |
       v
    Service
       |
       v
     Route
       |
       v
     Users
```
