# OpenShift Learning – Lesson 13: Builds & Source-to-Image (S2I)

## 🎯 Objectives

In this lesson, I learned and practiced:

- OpenShift Builds
- BuildConfig
- Build vs BuildConfig
- Source-to-Image (S2I)
- Builder Images
- Binary Builds
- Node.js application deployment
- ImageStreams
- ImageStreamTags
- Automatic deployment after a successful build
- ImageChange triggers
- Build logs
- Build troubleshooting
- Source code updates and rebuilds
- End-to-end S2I application deployment

---

# 1. What is an OpenShift Build?

An OpenShift **Build** converts application source code into a container image.

Basic flow:

```text
Source Code
    |
    v
OpenShift Build
    |
    v
Container Image
```

The resulting image can then be deployed as a container.

---
# 2. Build vs Deployment

### Build

Creates the container image:

```text
Source Code
    |
    v
Build
    |
    v
Container Image
```

### Deployment

Runs the container image:

```text
Container Image
    |
    v
Deployment
    |
    v
Pod
```

Easy way to remember:

```text
Build      → Creates the image

Deployment → Runs the image
```

---

# 3. What is S2I?

S2I means:

```text
Source-to-Image
```

S2I allows OpenShift to build a container image directly from application source code using a builder image.

Basic flow:

```text
Application Source Code
          |
          v
         S2I
          |
          v
    Builder Image
          |
          v
    Container Image
```

---

# 4. What is a Builder Image?

A builder image contains the tools and runtime required to build an application.

Examples:

```text
Node.js Builder
Python Builder
Java Builder
PHP Builder
```

Conceptually:

```text
Builder Image
     |
     +---- Runtime
     +---- Build tools
     +---- S2I scripts
```

For this lesson, we used the OpenShift Node.js builder:

```text
openshift/nodejs:latest
```

---

# 5. Our Practice Application

We created a simple Node.js application:

```text
lesson13-nodejs
```

Directory:

```text
lesson13-nodejs/
├── package.json
├── package-lock.json
└── server.js
```

---

# 6. Node.js Application

Created `server.js`:

```javascript
const http = require("http");

const hostname = "0.0.0.0";
const port = process.env.PORT || 8080;

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    "Content-Type": "text/html"
  });

  res.end(`
    <html>
      <body>
        <h1>Lesson 13 - OpenShift S2I</h1>
        <p>Hello from my Node.js application!</p>
      </body>
    </html>
  `);
});

server.listen(port, hostname, () => {
  console.log(`Application running on port ${port}`);
});
```

---

# 7. package.json

Created `package.json`:

```json
{
  "name": "lesson13-nodejs",
  "version": "1.0.0",
  "description": "OpenShift S2I practice application",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  }
}
```

---

# 8. Test Node.js Locally

If Node.js is installed:

```powershell
npm install
```

Run:

```powershell
npm start
```

Application:

```text
http://localhost:8080
```

If Node.js is not installed locally, this step can be skipped because OpenShift's Node.js builder will provide the runtime during the S2I build.

---

# 9. Create the S2I Application

We created the application using:

```powershell
oc new-app nodejs:latest~. --name=lesson13-nodejs
```

This created a BuildConfig.

Check:

```powershell
oc get bc
```

---

# 10. Important Discovery: Binary Build

When we checked the BuildConfig:

```powershell
oc describe bc lesson13-nodejs
```

we found:

```text
Strategy:       Source
From Image:     ImageStreamTag openshift/nodejs:latest
Output to:      ImageStreamTag lesson13-nodejs:latest
Binary:         provided on build
Build Run Policy: Serial
Triggered by:   Config, ImageChange
```

The important part was:

```text
Binary: provided on build
```

This means the source code must be provided to OpenShift when starting the build.

---

# 11. Initial Build Problem

The first build failed.

We checked:

```powershell
oc logs build/lesson13-nodejs-1
```

The important error was:

```text
STEP 8/9: RUN /usr/libexec/s2i/assemble
---> Installing application source ...
mv: cannot stat '/tmp/src/*': No such file or directory
error: build error
```

The problem was:

```text
OpenShift Build
      |
      v
S2I Builder
      |
      v
/tmp/src/
      |
      v
No source files
```

The Node.js builder was working correctly, but the source files were not provided to the binary build.

---

# 12. Verify Local Source Files

Our local directory was:

```text
C:\Users\LAKSHMINARAYANAN S\Downloads\lesson13-nodejs
```

We verified:

```powershell
dir
```

The directory contained:

```text
package.json
package-lock.json
server.js
```

Therefore, the application source was present locally.

---

# 13. Correct Binary Build Command

Because the BuildConfig uses binary source input, we needed to explicitly provide the local source directory.

The successful command was:

```powershell
oc start-build lesson13-nodejs --from-dir="C:\Users\LAKSHMINARAYANAN S\Downloads\lesson13-nodejs" --follow
```

This sends the local application source to the OpenShift Build.

Conceptually:

```text
Windows Local Directory
        |
        | --from-dir
        v
OpenShift Binary Build
        |
        v
S2I
        |
        v
Container Image
```

---

# 14. Build Successful

The successful build showed:

```text
Push successful
```

This confirmed that the container image was successfully built and pushed to the configured ImageStream.

Check:

```powershell
oc get builds
```

Expected:

```text
NAME                STATUS
lesson13-nodejs-1   Failed
lesson13-nodejs-2   Failed
lesson13-nodejs-3   Failed
lesson13-nodejs-4   Complete
```

The exact build number may be different depending on how many attempts were made.

---

# 15. Check Build Logs

Check a build:

```powershell
oc logs build/lesson13-nodejs-4
```

Or follow the build while running:

```powershell
oc logs -f build/lesson13-nodejs-4
```

The build should complete successfully.

Useful command:

```powershell
oc get builds
```

---

# 16. Check BuildConfig

```powershell
oc describe bc lesson13-nodejs
```

Important fields:

```text
Strategy:       Source
From Image:     openshift/nodejs:latest
Output to:      lesson13-nodejs:latest
Binary:         provided on build
Build Run Policy: Serial
Triggered by:   Config, ImageChange
```

---

# 17. BuildConfig vs Build

This is an important concept.

### BuildConfig

Defines **how** the application should be built.

```text
BuildConfig
     |
     +---- Source
     +---- Strategy
     +---- Builder Image
     +---- Output
     +---- Triggers
```

### Build

Represents **one execution** of the BuildConfig.

For example:

```text
BuildConfig
    |
    +---- Build 1
    |
    +---- Build 2
    |
    +---- Build 3
    |
    +---- Build 4
```

Easy way to remember:

```text
BuildConfig → Instructions

Build       → One execution
```

---

# 18. Check ImageStream

After a successful build:

```powershell
oc get is
```

Check:

```powershell
oc describe is lesson13-nodejs
```

The BuildConfig showed:

```text
Output to: ImageStreamTag lesson13-nodejs:latest
```

So the build output was associated with:

```text
lesson13-nodejs:latest
```

---

# 19. Check ImageStreamTag

```powershell
oc get istag
```

Or:

```powershell
oc describe istag lesson13-nodejs:latest
```

The overall flow is:

```text
Source Code
    |
    v
BuildConfig
    |
    v
Build
    |
    v
Container Image
    |
    v
ImageStream
    |
    v
ImageStreamTag
```

---

# 20. Automatic Deployment

After the successful build, the application was deployed automatically.

This was an important part of the lesson.

The BuildConfig had:

```text
Triggered by: Config, ImageChange
```

The overall workflow became:

```text
Source Code
     |
     v
BuildConfig
     |
     v
S2I Build
     |
     v
Container Image
     |
     v
ImageStream
     |
     v
ImageChange Trigger
     |
     v
Deployment Rollout
     |
     v
New Pod
```

We verified that the application was deployed and running.

---

# 21. Check Deployment

```powershell
oc get deployment
```

Check Pods:

```powershell
oc get pods
```

Expected:

```text
lesson13-nodejs-xxxxx   1/1   Running
```

The exact Pod name will be different.

---

# 22. Check Rollout

Verify the Deployment rollout:

```powershell
oc rollout status deployment/lesson13-nodejs
```

Expected:

```text
deployment "lesson13-nodejs" successfully rolled out
```

---

# 23. Check Service

```powershell
oc get svc
```

The application Service should be available.

---

# 24. Create Route

Expose the Service:

```powershell
oc expose service lesson13-nodejs
```

Check:

```powershell
oc get route
```

Get the hostname:

```powershell
oc get route lesson13-nodejs -o jsonpath="{.spec.host}"
```

Open the Route hostname in a browser.

---

# 25. Application Update Practice

We changed the source code.

For example:

```text
Hello from my Node.js application!
```

was changed to:

```text
Hello from my updated OpenShift S2I application!
```

Save the updated `server.js`.

---

# 26. Trigger a New Build

Because this is a binary BuildConfig, provide the source directory again:

```powershell
oc start-build lesson13-nodejs --from-dir="C:\Users\LAKSHMINARAYANAN S\Downloads\lesson13-nodejs" --follow
```

Check:

```powershell
oc get builds
```

A new build should appear.

For example:

```text
lesson13-nodejs-4   Complete
lesson13-nodejs-5   Complete
```

The exact build number will depend on previous attempts.

---

# 27. Verify Automatic Deployment After New Build

After the new build succeeds:

```powershell
oc get pods
```

Check:

```powershell
oc rollout status deployment/lesson13-nodejs
```

The updated image should trigger a new Deployment rollout when the ImageStream/ImageChange trigger is configured as in this setup.

---

# 28. Verify in Browser

Open the existing Route in the browser.

The updated application message should now appear.

This confirmed:

```text
Source Code Changed
       |
       v
New Binary Build
       |
       v
S2I
       |
       v
New Image
       |
       v
ImageStream Updated
       |
       v
ImageChange Trigger
       |
       v
Deployment Rollout
       |
       v
Updated Pod
       |
       v
Browser Shows New Version
```

---

# 29. S2I vs Dockerfile

### Traditional Dockerfile approach

```text
Source Code
    |
    v
Dockerfile
    |
    v
docker build
    |
    v
Container Image
```

### S2I approach

```text
Source Code
    |
    v
Builder Image
    |
    v
S2I
    |
    v
Container Image
```

S2I simplifies the image-building process for supported application stacks.

---

# 30. Build Strategies

OpenShift supports different build strategies.

Common strategies include:

```text
Source-to-Image (S2I)
Docker Build
Custom Build
Pipeline Build
```

For this lesson, the main focus was:

```text
Source-to-Image (S2I)
```

---

# 31. Troubleshooting Build Failures

If:

```powershell
oc get builds
```

shows:

```text
Failed
```

Start with:

```powershell
oc describe build <build-name>
```

Then:

```powershell
oc logs build/<build-name>
```

Check BuildConfig:

```powershell
oc describe bc lesson13-nodejs
```

Check ImageStream:

```powershell
oc describe is lesson13-nodejs
```

Check Pods:

```powershell
oc get pods
```

Check Events:

```powershell
oc get events --sort-by=.lastTimestamp
```

---

# 32. Important Troubleshooting Lesson

If you see:

```text
mv: cannot stat '/tmp/src/*': No such file or directory
```

during:

```text
/usr/libexec/s2i/assemble
```

check whether the BuildConfig uses:

```text
Binary: provided on build
```

If it does, make sure the local source directory is provided.

Correct example:

```powershell
oc start-build lesson13-nodejs --from-dir="C:\path\to\lesson13-nodejs" --follow
```

The directory must contain the application source files.

For our application:

```text
lesson13-nodejs/
├── package.json
├── package-lock.json
└── server.js
```

---

# 33. Useful Troubleshooting Flow

```text
Build Failed
     |
     v
oc describe build <build-name>
     |
     v
oc logs build/<build-name>
     |
     v
Check BuildConfig
     |
     v
Check Source Input
     |
     v
Check Builder Image
     |
     v
Check ImageStream
     |
     v
Check Deployment
     |
     v
Check Pod
```

---

# 34. Common Problems

| Problem | Possible Cause |
|---|---|
| Build Failed | Application/build error |
| `/tmp/src/*` not found | Binary source was not provided |
| Builder image unavailable | Image/registry issue |
| Dependency installation failed | Invalid dependency/package |
| Source not found | Incorrect source directory |
| Image not available | Build output problem |
| Pod CrashLoopBackOff | Application runtime problem |
| Deployment not updated | ImageChange/Deployment trigger issue |

---

# 35. Real-World S2I Workflow

A typical OpenShift workflow:

```text
Developer
    |
    v
Git Repository
    |
    v
BuildConfig
    |
    v
S2I Build
    |
    v
Container Image
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

In a CI/CD environment, source changes can automatically trigger builds and deployments.

---

# 36. Hands-On Practice Completed

### Application

```text
lesson13-nodejs
```

### Files

```text
package.json
package-lock.json
server.js
```

### BuildConfig

```text
lesson13-nodejs
```

### Builder

```text
openshift/nodejs:latest
```

### Build Strategy

```text
Source
```

### Source Type

```text
Binary
```

### Output

```text
lesson13-nodejs:latest
```

### Build

Successfully completed after providing the local source directory.

### Deployment

Automatically updated after the successful build.

### Route

Application was successfully accessible from the browser.

### Code Update

Updated source code was successfully rebuilt and deployed.

---

# 37. Interview Questions

1. What is an OpenShift Build?
2. What is BuildConfig?
3. What is the difference between BuildConfig and Build?
4. What is S2I?
5. What is a builder image?
6. How does S2I work?
7. What is a Binary Build?
8. Why did our first build fail?
9. What does `Binary: provided on build` mean?
10. How do you provide local source code to a binary build?
11. What is `oc start-build --from-dir`?
12. What happens after a successful S2I build?
13. Where is the build output stored?
14. What is the relationship between BuildConfig and ImageStream?
15. What is an ImageChange trigger?
16. How did our updated source code reach the running Pod?
17. How do you check Build status?
18. How do you check Build logs?
19. How do you troubleshoot a failed Build?
20. What is the difference between S2I and Dockerfile?

---

# 38. Useful Commands

### BuildConfig

```powershell
oc get bc
```

```powershell
oc describe bc lesson13-nodejs
```

```powershell
oc get bc lesson13-nodejs -o yaml
```

### Builds

```powershell
oc get builds
```

```powershell
oc describe build <build-name>
```

```powershell
oc logs build/<build-name>
```

```powershell
oc logs -f build/<build-name>
```

### Start Binary Build

```powershell
oc start-build lesson13-nodejs --from-dir="C:\path\to\lesson13-nodejs" --follow
```

### ImageStreams

```powershell
oc get is
```

```powershell
oc describe is lesson13-nodejs
```

```powershell
oc get istag
```

### Deployment

```powershell
oc get deployment
```

```powershell
oc rollout status deployment/lesson13-nodejs
```

### Pods

```powershell
oc get pods
```

```powershell
oc describe pod <pod-name>
```

```powershell
oc logs <pod-name>
```

### Service and Route

```powershell
oc get svc
```

```powershell
oc get route
```

```powershell
oc get route lesson13-nodejs -o jsonpath="{.spec.host}"
```

### Events

```powershell
oc get events --sort-by=.lastTimestamp
```

---

# 🧹 Cleanup

Check resources first:

```powershell
oc get all
```

Delete the application resources if required:

```powershell
oc delete all -l app=lesson13-nodejs
```

Check BuildConfig:

```powershell
oc get bc
```

Delete it if required:

```powershell
oc delete bc lesson13-nodejs
```

Check ImageStream:

```powershell
oc get is
```

Delete it if required:

```powershell
oc delete is lesson13-nodejs
```

---

# 🧠 Final Memory Trick

```text
BuildConfig → Defines HOW to build

Build       → One execution of BuildConfig

S2I         → Source-to-Image

Builder     → Provides runtime/tools for building

Binary Build → Local source is provided during the build

ImageStream → Tracks the resulting image

ImageChange → Can trigger a Deployment update

Deployment  → Runs the image
```

The complete workflow we successfully practiced:

```text
Local Source Code
       |
       | --from-dir
       v
Binary Build
       |
       v
BuildConfig
       |
       v
S2I
       |
       v
Container Image
       |
       v
ImageStream
       |
       v
ImageChange Trigger
       |
       v
Deployment Rollout
       |
       v
New Pod
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

# 🏁 Lesson 13 Completed

## What I Successfully Practiced

- [x] Created a Node.js application
- [x] Created an OpenShift S2I application
- [x] Created and inspected a BuildConfig
- [x] Understood Source builds
- [x] Identified Binary source input
- [x] Troubleshot a failed binary build
- [x] Identified the `/tmp/src/*` error
- [x] Provided local source using `--from-dir`
- [x] Successfully completed an S2I build
- [x] Verified the ImageStream output
- [x] Verified the Deployment
- [x] Verified the running Pod
- [x] Verified the Service
- [x] Verified the Route
- [x] Changed application source code
- [x] Triggered a new binary build
- [x] Verified automatic deployment after the successful build
- [x] Verified the updated application in the browser

## ⭐ Key Lesson Learned

> **When an OpenShift BuildConfig uses `Binary: provided on build`, the local application source must be explicitly supplied when starting the build using `oc start-build --from-dir`.**

**Lesson 13 Topic: OpenShift Builds, Binary Builds & Source-to-Image (S2I)**
