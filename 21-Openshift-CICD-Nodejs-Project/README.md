# OpenShift Learning – Lesson 21: OpenShift CI/CD Project

## 🚀 Lesson 21: Git → S2I Build → Image → Deploy → Test → Verify

> **Environment:** OpenShift Developer Sandbox  
> **Approach:** OpenShift-native CI/CD  
> **Build:** Source-to-Image (S2I)  
> **Application:** Node.js  
> **Container build:** OpenShift S2I  
> **Image:** OpenShift ImageStream  
> **Deployment:** OpenShift Deployment  
> **Source:** GitHub  
> **Dockerfile:** Not required

---

# 🎯 Learning Objectives

By the end of this lesson, I should understand:

- Git-based application deployment
- OpenShift BuildConfig
- Source-to-Image (S2I)
- Node.js S2I builder image
- OpenShift ImageStreams
- ImageStreamTags
- Build triggers
- ImageChange triggers
- Deployment automation
- Application testing
- Deployment verification
- OpenShift Events
- Build logs
- Deployment troubleshooting
- GitHub webhook concept
- Developer Sandbox webhook limitations
- End-to-end OpenShift CI/CD

---

# 🚫 What We Will NOT Use

This is an **OpenShift-only CI/CD project**.

We will NOT use:

```text
Dockerfile
docker build
docker push
Docker Hub
Jenkins
GitHub Actions
External CI/CD platforms
External container registries
Tekton
```

We will use:

```text
GitHub
   ↓
OpenShift BuildConfig
   ↓
S2I
   ↓
ImageStream
   ↓
Deployment
   ↓
Service
   ↓
Route
```

---

# 1. Project Architecture

The OpenShift-native workflow:

```text
GitHub
   ↓
OpenShift BuildConfig
   ↓
Node.js S2I
   ↓
ImageStream
   ↓
ImageChange
   ↓
Deployment
   ↓
Pod
   ↓
Service
   ↓
Route
   ↓
Browser
```

The CI/CD concept:

```text
Git
 ↓
Build
 ↓
Image
 ↓
Deploy
 ↓
Test
 ↓
Verify
```

---

# 2. Lesson 13 → Lesson 21

In Lesson 13, I practiced:

```text
Node.js
   ↓
S2I
   ↓
BuildConfig
   ↓
Image
   ↓
Deployment
```

Lesson 21 extends this:

```text
GitHub
   ↓
BuildConfig
   ↓
S2I
   ↓
ImageStream
   ↓
ImageChange
   ↓
Deployment
   ↓
Service
   ↓
Route
   ↓
Testing
   ↓
Verification
   ↓
Troubleshooting
```

---

# 3. Application Structure

Create:

```text
lesson21-openshift-cicd/
├── package.json
├── package-lock.json
├── server.js
└── test.js
```

There is **no Dockerfile**.

OpenShift's Node.js S2I builder will create the application image.

---

# 4. Create package.json

```json
{
  "name": "lesson21-openshift-cicd",
  "version": "1.0.0",
  "description": "OpenShift CI/CD Lesson 21 application",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "test": "node test.js"
  },
  "dependencies": {
    "express": "^5.1.0"
  }
}
```

---

# 5. Create server.js

```javascript
const express = require("express");

const app = express();

const PORT = process.env.PORT || 8080;

app.get("/", (req, res) => {
  res.send("Hello from Lesson 21 - OpenShift CI/CD!");
});

app.get("/health", (req, res) => {
  res.status(200).send("OK");
});

app.get("/version", (req, res) => {
  res.send("Lesson 21 - Version 1");
});

app.listen(PORT, "0.0.0.0", () => {
  console.log(`Application running on port ${PORT}`);
});
```

---

# 6. Create test.js

```javascript
const http = require("http");

const options = {
  hostname: "127.0.0.1",
  port: 8080,
  path: "/health",
  method: "GET"
};

const request = http.request(options, (response) => {
  if (response.statusCode === 200) {
    console.log("TEST PASSED");
    process.exit(0);
  } else {
    console.error("TEST FAILED");
    process.exit(1);
  }
});

request.on("error", () => {
  console.error("TEST FAILED");
  process.exit(1);
});

request.end();
```

---

# 7. Test the Application Locally

Install dependencies:

```cmd
npm install
```

Start the application:

```cmd
npm start
```

Open another CMD window.

Run:

```cmd
npm test
```

Expected:

```text
TEST PASSED
```

Stop the application:

```text
Ctrl + C
```

---

# 8. Push Application to GitHub

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
git commit -m "Initial Lesson 21 OpenShift CI/CD application"
```

Create a GitHub repository:

```text
lesson21-openshift-cicd
```

Add remote:

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

Verify that GitHub contains:

```text
package.json
package-lock.json
server.js
test.js
```

---

# 9. Verify OpenShift Project

Check the current project:

```cmd
oc project
```

Expected:

```text
lakshminarayananredh-dev
```

Check BuildConfigs:

```cmd
oc get bc
```

Check Builds:

```cmd
oc get builds
```

Check ImageStreams:

```cmd
oc get is
```

Check Deployments:

```cmd
oc get deployment
```

Check Pods:

```cmd
oc get pods
```

---

# 10. Check Node.js S2I Builder

Check OpenShift builder ImageStreams:

```cmd
oc get is -n openshift
```

Look for:

```text
nodejs
```

You can also use:

```cmd
oc get is -n openshift | findstr /i nodejs
```

We will use:

```text
nodejs:latest
```

as the S2I builder.

---

# 11. Create the OpenShift Application

Use the GitHub repository:

```cmd
oc new-app nodejs:latest~https://github.com/YOUR_USERNAME/lesson21-openshift-cicd.git --name=lesson21-app
```

Replace:

```text
YOUR_USERNAME
```

with your GitHub username.

---

# 12. Check Resources Created by OpenShift

BuildConfig:

```cmd
oc get bc
```

ImageStream:

```cmd
oc get is
```

Deployment:

```cmd
oc get deployment
```

Service:

```cmd
oc get service
```

Pods:

```cmd
oc get pods
```

The basic architecture is:

```text
GitHub
   ↓
BuildConfig
   ↓
S2I
   ↓
ImageStream
   ↓
Deployment
   ↓
Service
```

---

# 13. Start the Build

If the build did not automatically start:

```cmd
oc start-build lesson21-app
```

Check:

```cmd
oc get builds
```

Expected eventually:

```text
Complete
```

---

# 14. Watch the Build

Get the build name:

```cmd
oc get builds
```

Then:

```cmd
oc logs -f build/<build-name>
```

Example:

```cmd
oc logs -f build/lesson21-app-1
```

Look for:

```text
Installing application source
Installing dependencies
Build complete
```

---

# 15. Understand S2I

S2I means:

```text
Source-to-Image
```

The process:

```text
Node.js Builder Image
        +
Application Source
        ↓
       S2I
        ↓
Application Container Image
```

No Dockerfile is required.

OpenShift handles the image creation.

---

# 16. Understand BuildConfig

Run:

```cmd
oc describe bc lesson21-app
```

Look at:

```text
Strategy
From Image
Output to
Triggers
```

You should see information similar to:

```text
Strategy: Source
From Image: nodejs
Output to: lesson21-app:latest
```

The BuildConfig controls how OpenShift builds the application.

---

# 17. Check ImageStream

Run:

```cmd
oc get is
```

Then:

```cmd
oc describe is lesson21-app
```

You should see an image tag such as:

```text
latest
```

Conceptually:

```text
BuildConfig
    ↓
S2I
    ↓
lesson21-app:latest
    ↓
ImageStream
```

---

# 18. Check Deployment

Run:

```cmd
oc get deployment
```

Then:

```cmd
oc get pods
```

Expected:

```text
lesson21-app-xxxxx   1/1   Running
```

---

# 19. Expose the Application

Check the Service:

```cmd
oc get service
```

If `lesson21-app` exists:

```cmd
oc expose service lesson21-app
```

Check the Route:

```cmd
oc get route
```

Get the Route:

```cmd
oc get route lesson21-app
```

Open the Route URL in your browser.

Expected:

```text
Hello from Lesson 21 - OpenShift CI/CD!
```

---

# 20. Test the Application

Open:

```text
/
```

Expected:

```text
Hello from Lesson 21 - OpenShift CI/CD!
```

Open:

```text
/health
```

Expected:

```text
OK
```

Open:

```text
/version
```

Expected:

```text
Lesson 21 - Version 1
```

---

# 21. Understand the First CI/CD Flow

At this point:

```text
GitHub
   ↓
BuildConfig
   ↓
S2I
   ↓
ImageStream
   ↓
Deployment
   ↓
Pod
   ↓
Service
   ↓
Route
   ↓
Browser
```

This is the basic OpenShift-native CI/CD workflow.

---

# 22. ImageChange Trigger

Inspect the BuildConfig:

```cmd
oc describe bc lesson21-app
```

Look at:

```text
Triggers
```

You may see:

```text
Config
ImageChange
GitHub
Generic
```

The important deployment trigger is:

```text
ImageChange
```

Conceptually:

```text
New Image
   ↓
ImageStream
   ↓
ImageChange Trigger
   ↓
Deployment
```

---

# 23. Update Application to Version 2

Change `server.js`:

```javascript
app.get("/version", (req, res) => {
  res.send("Lesson 21 - Version 2");
});
```

Commit:

```cmd
git add .
```

```cmd
git commit -m "Update application to version 2"
```

Push:

```cmd
git push
```

---

# 24. Developer Sandbox GitHub Webhook Limitation

We attempted to configure the GitHub webhook.

OpenShift provided:

```text
Webhook Generic:
https://api.rm1.0a51.p1.openshiftapps.com:6443/apis/build.openshift.io/v1/namespaces/lakshminarayananredh-dev/buildconfigs/lesson21-app/webhooks/<secret>/generic
```

and:

```text
Webhook GitHub:
https://api.rm1.0a51.p1.openshiftapps.com:6443/apis/build.openshift.io/v1/namespaces/lakshminarayananredh-dev/buildconfigs/lesson21-app/webhooks/<secret>/github
```

GitHub returned:

```text
403 Forbidden
```

We checked:

```cmd
oc auth can-i create buildconfigs/webhooks --as=system:unauthenticated -n lakshminarayananredh-dev
```

Result:

```text
no
```

---

# 25. Why the GitHub Webhook Returns 403

The problem is **not the webhook secret**.

The Developer Sandbox user does not have permission to grant the required webhook access to:

```text
system:unauthenticated
```

Therefore:

```text
GitHub
   ↓
Webhook
   ↓
OpenShift API
   ↓
403 Forbidden
```

This is a **Developer Sandbox permission limitation**, not an application configuration mistake.

---

# 26. Do Not Waste Time Fixing the Webhook

Do not:

```text
Create random secrets
Change the webhook URL repeatedly
Change the payload format
Reconfigure the application
```

We have already confirmed the permission problem.

For this learning environment, use:

```cmd
oc start-build lesson21-app
```

as the practical BuildConfig trigger.

---

# 27. Manual Build Trigger

After pushing a Git commit:

```cmd
git push
```

Trigger the OpenShift build:

```cmd
oc start-build lesson21-app
```

Check:

```cmd
oc get builds
```

Watch:

```cmd
oc logs -f build/<build-name>
```

The flow becomes:

```text
GitHub
   ↓
Source Updated
   ↓
oc start-build
   ↓
OpenShift BuildConfig
   ↓
S2I
   ↓
Image
   ↓
Deployment
```

---

# 28. Verify Version 2

After the build completes:

```cmd
oc get pods
```

Check rollout:

```cmd
oc rollout status deployment/lesson21-app
```

Open:

```text
/version
```

Expected:

```text
Lesson 21 - Version 2
```

---

# 29. Automated Testing Concept

The desired CI/CD flow is:

```text
Source
   ↓
Build
   ↓
Test
   ↓
Deploy
```

Important rule:

```text
Test FAILED
     ↓
Do NOT release bad application
```

For this lesson, we will practice application health testing after deployment.

---

# 30. Check Running Pod

Run:

```cmd
oc get pods
```

Find:

```text
lesson21-app-xxxxx
```

Then:

```cmd
oc describe pod <pod-name>
```

Check:

```text
Status
Containers
Ready
Events
```

---

# 31. Application Health Test

Run:

```cmd
oc exec <pod-name> -- node -e "require('http').get('http://127.0.0.1:8080/health',r=>process.exit(r.statusCode===200?0:1))"
```

If successful, the command exits successfully.

Conceptually:

```text
Pod
 ↓
Application
 ↓
/health
 ↓
HTTP 200
 ↓
TEST PASSED
```

---

# 32. Check Deployment Health

Run:

```cmd
oc rollout status deployment/lesson21-app
```

Expected:

```text
deployment "lesson21-app" successfully rolled out
```

Then:

```cmd
oc get pods
```

Expected:

```text
1/1 Running
```

---

# 33. Check Service

Run:

```cmd
oc get service
```

Find:

```text
lesson21-app
```

The Service provides internal access:

```text
Service
   ↓
Pod
```

---

# 34. Check Route

Run:

```cmd
oc get route
```

Then:

```cmd
oc get route lesson21-app
```

The Route provides external access:

```text
Internet
   ↓
Route
   ↓
Service
   ↓
Pod
```

---

# 35. Build Failure Practice

Intentionally introduce a JavaScript syntax error.

Commit:

```cmd
git add .
```

```cmd
git commit -m "Test failed build"
```

Push:

```cmd
git push
```

Start the build:

```cmd
oc start-build lesson21-app
```

Check:

```cmd
oc get builds
```

---

# 36. Troubleshoot Failed Build

If the build fails:

```cmd
oc logs build/<failed-build-name>
```

Then:

```cmd
oc describe build/<failed-build-name>
```

Look for:

```text
Error
Failed
npm
S2I
```

Troubleshooting flow:

```text
Build
 ↓
Failed
 ↓
Build Logs
 ↓
Root Cause
 ↓
Fix Source
 ↓
New Build
```

---

# 37. Fix the Application

Fix the source code.

Commit:

```cmd
git add .
```

```cmd
git commit -m "Fix application build"
```

Push:

```cmd
git push
```

Start:

```cmd
oc start-build lesson21-app
```

Check:

```cmd
oc get builds
```

Expected:

```text
Complete
```

---

# 38. Deployment Troubleshooting

Check:

```cmd
oc get deployment
```

Then:

```cmd
oc describe deployment lesson21-app
```

Check Pods:

```cmd
oc get pods
```

If a Pod fails:

```cmd
oc describe pod <pod-name>
```

Then:

```cmd
oc logs <pod-name>
```

---

# 39. OpenShift Events

Events are important for troubleshooting.

Run:

```cmd
oc get events --sort-by=.lastTimestamp
```

Look for:

```text
Scheduled
Pulling
Pulled
Created
Started
BackOff
Failed
```

---

# 40. Build Troubleshooting Flow

When BuildConfig fails:

```text
oc get builds
        ↓
Find failed build
        ↓
oc describe build/<name>
        ↓
oc logs build/<name>
        ↓
Find root cause
```

---

# 41. Deployment Troubleshooting Flow

When the application doesn't work:

```text
oc get deployment
        ↓
oc get pods
        ↓
oc describe pod
        ↓
oc logs pod
        ↓
oc get events
        ↓
Find root cause
```

---

# 42. CI/CD Troubleshooting Flow

Complete troubleshooting:

```text
Git
 ↓
BuildConfig
 ↓
Build
 ↓
ImageStream
 ↓
Deployment
 ↓
Pod
 ↓
Service
 ↓
Route
```

Troubleshoot from left to right.

---

# 43. Version 3 Practice

Change:

```javascript
app.get("/version", (req, res) => {
  res.send("Lesson 21 - Version 3");
});
```

Commit:

```cmd
git add .
```

```cmd
git commit -m "Release version 3"
```

Push:

```cmd
git push
```

Start:

```cmd
oc start-build lesson21-app
```

Check:

```cmd
oc get builds
```

After successful build:

```cmd
oc rollout status deployment/lesson21-app
```

Open:

```text
/version
```

Expected:

```text
Lesson 21 - Version 3
```

---

# 44. Notification Practice

For the Developer Sandbox, we will not introduce external Slack, Teams or email systems.

Instead, use OpenShift build status and Events.

Check:

```cmd
oc get builds
```

Successful build:

```text
Complete
```

Failed build:

```text
Failed
```

Build details:

```cmd
oc describe build/<build-name>
```

Events:

```cmd
oc get events --sort-by=.lastTimestamp
```

This gives practical CI/CD status and troubleshooting experience without introducing another platform.

---

# 45. Final Project Flow

The actual Developer Sandbox workflow is:

```text
                    GitHub
                       |
                       | git push
                       v
                Source Updated
                       |
                       v
              oc start-build
                       |
                       v
              OpenShift BuildConfig
                       |
                       v
                  Node.js S2I
                       |
                       v
                  ImageStream
                       |
                       v
                  Deployment
                       |
                       v
                     Pod
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

Testing:

```text
Deployment
   ↓
Pod
   ↓
/health
   ↓
HTTP 200
   ↓
TEST PASSED
```

Troubleshooting:

```text
Build / Deployment
       ↓
     Events
       ↓
      Logs
       ↓
   Root Cause
```

---

# 46. Lesson 21 vs Lesson 13

## Lesson 13

```text
Node.js
 ↓
S2I
 ↓
BuildConfig
 ↓
Image
 ↓
Deployment
```

## Lesson 21

```text
GitHub
 ↓
BuildConfig
 ↓
S2I
 ↓
ImageStream
 ↓
ImageChange
 ↓
Deployment
 ↓
Service
 ↓
Route
 ↓
Testing
 ↓
Verification
 ↓
Troubleshooting
```

Lesson 21 is a practical extension of Lesson 13.

---


# 🧠 Final Memory Trick

Remember:

```text
G → B → S → I → D → T → V
```

Where:

```text
G = Git
B = BuildConfig
S = S2I
I = ImageStream
D = Deployment
T = Test
V = Verify
```

Actual Developer Sandbox workflow:

```text
GitHub
   ↓
git push
   ↓
oc start-build
   ↓
BuildConfig
   ↓
S2I
   ↓
ImageStream
   ↓
Deployment
   ↓
Pod
   ↓
Service
   ↓
Route
   ↓
Test
   ↓
Verify
```

---

# 🔧 Important Commands

## BuildConfig

```cmd
oc get bc
```

```cmd
oc describe bc lesson21-app
```

```cmd
oc start-build lesson21-app
```

## Builds

```cmd
oc get builds
```

```cmd
oc logs -f build/<build-name>
```

```cmd
oc describe build/<build-name>
```

## ImageStreams

```cmd
oc get is
```

```cmd
oc describe is lesson21-app
```

## Deployment

```cmd
oc get deployment
```

```cmd
oc describe deployment lesson21-app
```

```cmd
oc rollout status deployment/lesson21-app
```

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

## Service

```cmd
oc get service
```

## Route

```cmd
oc get route
```

```cmd
oc get route lesson21-app
```

## Events

```cmd
oc get events --sort-by=.lastTimestamp
```

## Webhook Permission Check

```cmd
oc auth can-i create buildconfigs/webhooks --as=system:unauthenticated -n lakshminarayananredh-dev
```

Expected in this Developer Sandbox:

```text
no
```

---

# ⚠️ Developer Sandbox Limitation

For this environment:

```text
GitHub
   ↓
GitHub Webhook
   ↓
OpenShift
   ↓
403 Forbidden
```

We confirmed:

```cmd
oc auth can-i create buildconfigs/webhooks --as=system:unauthenticated -n lakshminarayananredh-dev
```

returns:

```text
no
```

Therefore, this is a **Developer Sandbox permission limitation**.

Do not treat it as a failed application configuration.

For practical Lesson 21 exercises, use:

```cmd
oc start-build lesson21-app
```

The following OpenShift concepts remain fully available:

```text
BuildConfig
S2I
Build
ImageStream
ImageStreamTag
ImageChange
Deployment
Service
Route
Testing
Events
Logs
Troubleshooting
```

---

# ✅ Lesson 21 Completion Checklist

- [ ] Create Node.js application
- [ ] Create GitHub repository
- [ ] Push source code to GitHub
- [ ] Understand S2I
- [ ] Understand why Dockerfile is not required
- [ ] Create OpenShift application using Node.js S2I
- [ ] Understand BuildConfig
- [ ] Start an OpenShift build
- [ ] Monitor build logs
- [ ] Understand ImageStream
- [ ] Understand ImageStreamTag
- [ ] Understand ImageChange
- [ ] Deploy application
- [ ] Create Service
- [ ] Create Route
- [ ] Access application from browser
- [ ] Test `/health`
- [ ] Test `/version`
- [ ] Update application source
- [ ] Build a new application version
- [ ] Verify deployment
- [ ] Intentionally create a failed build
- [ ] Troubleshoot build logs
- [ ] Troubleshoot Pod logs
- [ ] Check OpenShift Events
- [ ] Check deployment rollout
- [ ] Understand GitHub webhook
- [ ] Understand Developer Sandbox 403 limitation
- [ ] Use `oc start-build` as the practical trigger
- [ ] Complete Git → Build → Image → Deploy → Test → Verify workflow

---

# 🏁 Lesson 21 Goal


> **OpenShift can build and deploy a Node.js application directly from source using S2I without requiring a Dockerfile. BuildConfig controls the build, S2I converts source code into a container image, ImageStream tracks the image, and deployment automation runs the application. Services and Routes expose the application, while Builds, Events and logs are used for testing and troubleshooting.**

Final OpenShift-native workflow:

```text
                    GitHub
                       |
                       v
                Source Code
                       |
                       v
             OpenShift BuildConfig
                       |
                       v
                  Node.js S2I
                       |
                       v
                  ImageStream
                       |
                       v
                  Deployment
                       |
                       v
                     Pod
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

Developer Sandbox practical trigger:

```text
GitHub
   ↓
git push
   ↓
oc start-build lesson21-app
   ↓
S2I Build
   ↓
ImageStream
   ↓
Deployment
   ↓
Test
   ↓
Verify
```

**Lesson 21 Topic: OpenShift CI/CD Project – Git → S2I Build → ImageStream → Deployment → Testing → Verification → Troubleshooting**
