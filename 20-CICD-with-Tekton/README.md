# OpenShift Learning – Lesson 20: CI/CD with Tekton

## 🚀 Lesson 20: OpenShift CI/CD with Tekton – Tasks, Pipelines, PipelineRuns, TaskRuns & Parameters

> **Environment Note:** I am using an OpenShift Developer Sandbox with restricted permissions. This cluster contains **both Tekton Pipelines and Kubeflow Pipelines**, and both expose a resource named `pipelines`. Because of this, commands such as `oc get pipeline` or `oc get pipelines` can resolve to the wrong API resource. For Lesson 20, always use the explicit Tekton API resource names such as `pipelines.tekton.dev`, `tasks.tekton.dev`, `pipelineruns.tekton.dev`, and `taskruns.tekton.dev`.

---

# 🎯 Learning Objectives

By the end of this lesson, I should understand:

- What CI/CD means
- What Tekton is
- Tekton `Task`
- Tekton `TaskRun`
- Tekton `Pipeline`
- Tekton `PipelineRun`
- Pipeline parameters
- Task parameters
- `runAfter`
- Basic Pipeline execution
- How PipelineRuns create TaskRuns
- How TaskRuns create Pods
- Basic CI/CD troubleshooting
- How to inspect Tekton resources when multiple APIs have the same short name

---

# 1. What Is CI/CD?

CI/CD means:

```text
Continuous Integration
        +
Continuous Delivery / Deployment
```

A simple CI/CD workflow:

```text
Developer
   ↓
Git Push
   ↓
Build
   ↓
Test
   ↓
Container Image
   ↓
Deploy
   ↓
OpenShift
```

Without CI/CD, we may manually perform:

```text
git pull
npm install
npm test
docker build
docker push
oc apply
```

With CI/CD:

```text
Git Push
   ↓
Pipeline
   ↓
Build
   ↓
Test
   ↓
Deploy
```

---

# 2. What Is Tekton?

Tekton is a Kubernetes-native framework for creating CI/CD pipelines.

The important objects are:

```text
Task
  ↓
TaskRun
```

and:

```text
Pipeline
  ↓
PipelineRun
```

A Pipeline can contain multiple Tasks.

---

# 3. Tekton Architecture

The basic model:

```text
                 Pipeline
                    |
          +---------+---------+
          |                   |
        Task 1              Task 2
          |                   |
        Build                Test
```

When the Pipeline is executed:

```text
Pipeline
   ↓
PipelineRun
   ↓
TaskRuns
   ↓
Pods
```

Remember this relationship:

```text
Task
 ↓
TaskRun
 ↓
Pod
```

and:

```text
Pipeline
 ↓
PipelineRun
 ↓
TaskRuns
 ↓
Pods
```

---

# 4. Important: Your OpenShift Has Two Pipeline APIs

Run:

```cmd
oc api-resources | findstr /i pipeline
```

You may see:

```text
pipelines.kubeflow.org/v2beta1
```

and:

```text
pipelines.tekton.dev/v1
```

These are different resources.

Your cluster has:

```text
Kubeflow Pipeline
        +
Tekton Pipeline
```

For this lesson, we use:

```text
Tekton
```

Therefore, use the explicit Tekton API resources.

---

# 5. Why `oc get pipeline` Caused a Problem

We initially ran:

```cmd
oc get pipeline lesson20-pipeline
```

and OpenShift returned:

```text
Error from server (NotFound): pipelines.pipelines.kubeflow.org "lesson20-pipeline" not found
```

But when the Pipeline was created, OpenShift said:

```text
pipeline.tekton.dev/lesson20-pipeline created
```

This means the Pipeline was created successfully as a **Tekton Pipeline**.

The problem was that:

```cmd
oc get pipeline
```

resolved to:

```text
pipelines.pipelines.kubeflow.org
```

instead of:

```text
pipelines.tekton.dev
```

---

# 6. Correct Tekton Commands

## Tasks

Use:

```cmd
oc get tasks.tekton.dev
```

instead of:

```cmd
oc get tasks
```

## TaskRuns

Use:

```cmd
oc get taskruns.tekton.dev
```

instead of:

```cmd
oc get taskruns
```

## Pipelines

Use:

```cmd
oc get pipelines.tekton.dev
```

instead of:

```cmd
oc get pipelines
```

## PipelineRuns

Use:

```cmd
oc get pipelineruns.tekton.dev
```

instead of:

```cmd
oc get pipelineruns
```

This avoids ambiguity.

---

# 7. Check Tekton Resources

Run:

```cmd
oc get tasks.tekton.dev
```

Then:

```cmd
oc get taskruns.tekton.dev
```

Then:

```cmd
oc get pipelines.tekton.dev
```

Then:

```cmd
oc get pipelineruns.tekton.dev
```

---

# 8. Check a Specific Tekton Pipeline

Use:

```cmd
oc get pipelines.tekton.dev lesson20-pipeline
```

Describe it:

```cmd
oc describe pipelines.tekton.dev lesson20-pipeline
```

This guarantees that OpenShift uses the Tekton Pipeline API.

---

# 9. Check a Specific Task

Use:

```cmd
oc get tasks.tekton.dev lesson20-hello
```

Describe:

```cmd
oc describe tasks.tekton.dev lesson20-hello
```

---

# 10. Check a Specific TaskRun

Use:

```cmd
oc get taskruns.tekton.dev lesson20-hello-run
```

Describe:

```cmd
oc describe taskruns.tekton.dev lesson20-hello-run
```

---

# 11. Check a Specific PipelineRun

Use:

```cmd
oc get pipelineruns.tekton.dev lesson20-pipeline-run
```

Describe:

```cmd
oc describe pipelineruns.tekton.dev lesson20-pipeline-run
```

---

# 12. Create Your First Tekton Task

Create:

```text
lesson20-task.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: lesson20-hello
spec:
  steps:
    - name: hello
      image: busybox:1.36
      script: |
        #!/bin/sh
        echo "Hello from Tekton"
        echo "Lesson 20 CI/CD"
```

Apply:

```cmd
oc apply -f lesson20-task.yaml
```

Expected:

```text
task.tekton.dev/lesson20-hello created
```

---

# 13. Verify the Task

Use the explicit Tekton resource:

```cmd
oc get tasks.tekton.dev
```

You should see:

```text
lesson20-hello
```

Specific Task:

```cmd
oc get tasks.tekton.dev lesson20-hello
```

---

# 14. Understand the Task

Our Task contains:

```text
Task
 |
 +---- Step
         |
         +---- busybox
         |
         +---- echo
```

The Task is a reusable definition.

Think:

```text
Task
 ↓
Definition
```

It does not automatically execute just because we created it.

---

# 15. Create a TaskRun

A TaskRun executes a Task.

Create:

```text
lesson20-taskrun.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: lesson20-hello-run
spec:
  taskRef:
    name: lesson20-hello
```

Apply:

```cmd
oc apply -f lesson20-taskrun.yaml
```

---

# 16. Check the TaskRun

Use:

```cmd
oc get taskruns.tekton.dev
```

You should see something similar to:

```text
NAME
lesson20-hello-run
```

Check the specific TaskRun:

```cmd
oc get taskruns.tekton.dev lesson20-hello-run
```

---

# 17. Describe the TaskRun

Run:

```cmd
oc describe taskruns.tekton.dev lesson20-hello-run
```

Look for:

```text
Conditions
Status
Steps
Pod
```

---

# 18. Find the Task Pod

Run:

```cmd
oc get pods
```

Tekton creates a Pod to execute the TaskRun.

Conceptually:

```text
Task
 ↓
TaskRun
 ↓
Pod
 ↓
Container
 ↓
Command
```

---

# 19. View Task Logs

Find the Tekton Pod:

```cmd
oc get pods
```

Then:

```cmd
oc logs <task-pod-name>
```

You should see:

```text
Hello from Tekton
Lesson 20 CI/CD
```

---

# 20. Task vs TaskRun

Remember:

```text
Task
 ↓
What should happen?
```

```text
TaskRun
 ↓
Run it now
```

Easy memory:

```text
Task = Definition

TaskRun = Execution
```

---

# 21. Create a Pipeline

A Pipeline combines multiple Tasks.

Create:

```text
lesson20-pipeline.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: lesson20-pipeline
spec:
  tasks:
    - name: hello
      taskRef:
        name: lesson20-hello
```

Apply:

```cmd
oc apply -f lesson20-pipeline.yaml
```

---

# 22. Verify the Pipeline

**Important:** Do not use the ambiguous command:

```cmd
oc get pipelines
```

Use:

```cmd
oc get pipelines.tekton.dev
```

You should see:

```text
lesson20-pipeline
```

Check specifically:

```cmd
oc get pipelines.tekton.dev lesson20-pipeline
```

Describe:

```cmd
oc describe pipelines.tekton.dev lesson20-pipeline
```

---

# 23. Create a PipelineRun

A PipelineRun executes a Pipeline.

Create:

```text
lesson20-pipelinerun.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: lesson20-pipeline-run
spec:
  pipelineRef:
    name: lesson20-pipeline
```

Apply:

```cmd
oc apply -f lesson20-pipelinerun.yaml
```

---

# 24. Check the PipelineRun

Use:

```cmd
oc get pipelineruns.tekton.dev
```

Check specifically:

```cmd
oc get pipelineruns.tekton.dev lesson20-pipeline-run
```

You should eventually see the PipelineRun complete successfully.

---

# 25. Describe the PipelineRun

Run:

```cmd
oc describe pipelineruns.tekton.dev lesson20-pipeline-run
```

Look for:

```text
Pipeline
Tasks
Conditions
Status
```

---

# 26. Understand Pipeline Execution

The flow is:

```text
Pipeline
   ↓
PipelineRun
   ↓
TaskRun
   ↓
Pod
   ↓
Container
```

This is one of the most important concepts in Lesson 20.

---

# 27. Create a Second Task

Create:

```text
lesson20-test-task.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: lesson20-test
spec:
  steps:
    - name: test
      image: busybox:1.36
      script: |
        #!/bin/sh
        echo "Running application tests..."
        echo "Tests completed successfully"
```

Apply:

```cmd
oc apply -f lesson20-test-task.yaml
```

Verify:

```cmd
oc get tasks.tekton.dev
```

You should see:

```text
lesson20-hello
lesson20-test
```

---

# 28. Pipeline With Two Tasks

Update:

```text
lesson20-pipeline.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: lesson20-pipeline
spec:
  tasks:
    - name: hello
      taskRef:
        name: lesson20-hello

    - name: test
      runAfter:
        - hello
      taskRef:
        name: lesson20-test
```

Apply:

```cmd
oc apply -f lesson20-pipeline.yaml
```

Verify:

```cmd
oc get pipelines.tekton.dev
```

---

# 29. Understand `runAfter`

This:

```yaml
runAfter:
  - hello
```

means:

```text
hello
  ↓
test
```

So the Pipeline executes:

```text
Task 1
  ↓
Task 2
```

Without dependencies, independent Tasks may be able to execute in parallel.

For now, `runAfter` makes the Pipeline flow easier to understand.

---

# 30. Run the Updated Pipeline

Create:

```text
lesson20-pipeline-run-2.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: lesson20-pipeline-run-2
spec:
  pipelineRef:
    name: lesson20-pipeline
```

Apply:

```cmd
oc apply -f lesson20-pipeline-run-2.yaml
```

Check:

```cmd
oc get pipelineruns.tekton.dev
```

---

# 31. Observe TaskRuns

Run:

```cmd
oc get taskruns.tekton.dev
```

You should see TaskRuns created by the PipelineRun.

Conceptually:

```text
PipelineRun
     |
     +---- TaskRun: hello
     |
     +---- TaskRun: test
```

---

# 32. Pipeline Parameters

Parameters make Pipelines reusable.

Example:

```yaml
params:
  - name: MESSAGE
    type: string
    default: "Hello from Pipeline"
```

The Pipeline can pass the parameter to a Task.

Conceptually:

```text
PipelineRun parameter
       ↓
Pipeline parameter
       ↓
Task parameter
       ↓
Container command
```

---

# 33. Create Parameterized Task

Create:

```text
lesson20-param-task.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: lesson20-message
spec:
  params:
    - name: MESSAGE
      type: string
      default: "Hello from Tekton"
  steps:
    - name: message
      image: busybox:1.36
      script: |
        #!/bin/sh
        echo "$(params.MESSAGE)"
```

Apply:

```cmd
oc apply -f lesson20-param-task.yaml
```

Verify:

```cmd
oc get tasks.tekton.dev
```

---

# 34. Create Parameterized Pipeline

Create:

```text
lesson20-param-pipeline.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: lesson20-param-pipeline
spec:
  params:
    - name: MESSAGE
      type: string
      default: "Hello from Pipeline"
  tasks:
    - name: message
      taskRef:
        name: lesson20-message
      params:
        - name: MESSAGE
          value: $(params.MESSAGE)
```

Apply:

```cmd
oc apply -f lesson20-param-pipeline.yaml
```

Verify:

```cmd
oc get pipelines.tekton.dev
```

---

# 35. Run Parameterized Pipeline

Create:

```text
lesson20-param-run.yaml
```

Use:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: lesson20-param-run
spec:
  pipelineRef:
    name: lesson20-param-pipeline
  params:
    - name: MESSAGE
      value: "Hello from Lesson 20"
```

Apply:

```cmd
oc apply -f lesson20-param-run.yaml
```

Check:

```cmd
oc get pipelineruns.tekton.dev
```

---

# 36. Understand Parameter Flow

The value:

```text
Hello from Lesson 20
```

flows through:

```text
PipelineRun
     ↓
Pipeline parameter
     ↓
Task parameter
     ↓
Container command
```

This is important for real CI/CD pipelines.

---

# 37. CI/CD Pipeline Concept

A real application pipeline could eventually look like:

```text
Git Repository
      ↓
Clone Source
      ↓
Build
      ↓
Test
      ↓
Build Container Image
      ↓
Push Image
      ↓
Deploy to OpenShift
      ↓
Verify
```

We are not building the complete production pipeline yet.

For Lesson 20, first understand:

```text
Task
TaskRun
Pipeline
PipelineRun
Parameters
```

---

# 38. Real-World Pipeline

Eventually:

```text
                  Git
                   |
                   v
                Checkout
                   |
                   v
                 Build
                   |
                   v
                 Test
                   |
                   v
              Build Image
                   |
                   v
               Push Image
                   |
                   v
             Deploy OpenShift
                   |
                   v
                Verify
```

---

# 39. Workspaces

Tekton Workspaces provide shared storage between Tasks.

Conceptually:

```text
Task 1
  |
  v
Workspace
  |
  v
Task 2
```

For example:

```text
Git source
   ↓
Workspace
   ↓
Build Task
```

We will practice Workspaces after understanding the basic Pipeline model.

---

# 40. CI/CD Troubleshooting

If a PipelineRun fails:

```cmd
oc get pipelineruns.tekton.dev
```

Then:

```cmd
oc describe pipelineruns.tekton.dev <pipelinerun-name>
```

Check TaskRuns:

```cmd
oc get taskruns.tekton.dev
```

Then:

```cmd
oc describe taskruns.tekton.dev <taskrun-name>
```

Find the Pod:

```cmd
oc get pods
```

Then:

```cmd
oc logs <pod-name>
```

Troubleshooting flow:

```text
PipelineRun
     ↓
TaskRun
     ↓
Pod
     ↓
Container
     ↓
Logs
     ↓
Root Cause
```

---

# 41. Common Tekton Problems

## Problem 1 – Wrong Pipeline API

If:

```cmd
oc get pipeline lesson20-pipeline
```

returns:

```text
pipelines.pipelines.kubeflow.org
```

use:

```cmd
oc get pipelines.tekton.dev lesson20-pipeline
```

---

## Problem 2 – Wrong Task API

Use:

```cmd
oc get tasks.tekton.dev
```

instead of relying on:

```cmd
oc get tasks
```

---

## Problem 3 – Wrong TaskRun API

Use:

```cmd
oc get taskruns.tekton.dev
```

---

## Problem 4 – Wrong PipelineRun API

Use:

```cmd
oc get pipelineruns.tekton.dev
```

---

## Problem 5 – Task Not Found

Check:

```cmd
oc get tasks.tekton.dev
```

Make sure the Task referenced by the Pipeline exists.

---

## Problem 6 – Pipeline Not Found

Check:

```cmd
oc get pipelines.tekton.dev
```

---

## Problem 7 – TaskRun Failed

Check:

```cmd
oc get taskruns.tekton.dev
```

Then:

```cmd
oc describe taskruns.tekton.dev <taskrun-name>
```

---

## Problem 8 – Pod Failed

Check:

```cmd
oc get pods
```

Then:

```cmd
oc describe pod <pod-name>
```

Then:

```cmd
oc logs <pod-name>
```

---

# 42. Important Tekton Resource Relationship

Memorize this:

```text
Task
 ↓
TaskRun
```

```text
Pipeline
 ↓
PipelineRun
```

And:

```text
PipelineRun
     ↓
TaskRuns
     ↓
Pods
```

---


# 🧠 Final Memory Trick

Remember:

```text
Task
 ↓
One unit of work
```

```text
TaskRun
 ↓
One execution of a Task
```

```text
Pipeline
 ↓
Multiple Tasks
```

```text
PipelineRun
 ↓
One execution of a Pipeline
```

Complete flow:

```text
Pipeline
    |
    v
PipelineRun
    |
    +---- TaskRun
    |       |
    |       v
    |      Pod
    |
    +---- TaskRun
            |
            v
           Pod
```

CI/CD flow:

```text
Git
 ↓
Build
 ↓
Test
 ↓
Image
 ↓
Push
 ↓
Deploy
 ↓
Verify
```

---

# 🔧 Important Commands

## Check API Resources

```cmd
oc api-resources | findstr /i pipeline
```

## Tekton Tasks

```cmd
oc get tasks.tekton.dev
```

```cmd
oc get tasks.tekton.dev <task-name>
```

```cmd
oc describe tasks.tekton.dev <task-name>
```

## Tekton TaskRuns

```cmd
oc get taskruns.tekton.dev
```

```cmd
oc get taskruns.tekton.dev <taskrun-name>
```

```cmd
oc describe taskruns.tekton.dev <taskrun-name>
```

## Tekton Pipelines

```cmd
oc get pipelines.tekton.dev
```

```cmd
oc get pipelines.tekton.dev <pipeline-name>
```

```cmd
oc describe pipelines.tekton.dev <pipeline-name>
```

## Tekton PipelineRuns

```cmd
oc get pipelineruns.tekton.dev
```

```cmd
oc get pipelineruns.tekton.dev <pipelinerun-name>
```

```cmd
oc describe pipelineruns.tekton.dev <pipelinerun-name>
```

## Apply Resources

```cmd
oc apply -f lesson20-task.yaml
```

```cmd
oc apply -f lesson20-taskrun.yaml
```

```cmd
oc apply -f lesson20-pipeline.yaml
```

```cmd
oc apply -f lesson20-pipelinerun.yaml
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

---

# ⚠️ Commands to Avoid in This Environment

Because your cluster contains both Kubeflow and Tekton Pipeline APIs, avoid relying on these ambiguous commands for Lesson 20:

```cmd
oc get pipeline
```

```cmd
oc get pipelines
```

```cmd
oc get tasks
```

```cmd
oc get taskruns
```

```cmd
oc get pipelineruns
```

Instead use:

```cmd
oc get tasks.tekton.dev
```

```cmd
oc get taskruns.tekton.dev
```

```cmd
oc get pipelines.tekton.dev
```

```cmd
oc get pipelineruns.tekton.dev
```

This makes it explicit that we are working with **Tekton**.

---

# ⚠️ OpenShift Developer Sandbox Permission Check

Some Tekton resources may not be available to your user.

If you see:

```text
Error from server (Forbidden)
```

do not try to modify cluster-level permissions.

Check available APIs:

```cmd
oc api-resources | findstr /i tekton
```

Then check the specific resource.

For example:

```cmd
oc get pipelines.tekton.dev
```

This is also a real-world OpenShift troubleshooting lesson:

```text
Command
  ↓
Permission / API issue
  ↓
Identify correct API resource
  ↓
Use explicit resource
  ↓
Continue
```

---

# ✅ Lesson 20 Completion Checklist

- [ ] Understand CI/CD
- [ ] Understand Tekton
- [ ] Understand Tekton Task
- [ ] Create a Task
- [ ] Create a TaskRun
- [ ] Execute a Task
- [ ] Check TaskRun status
- [ ] Read Task logs
- [ ] Understand Pipeline
- [ ] Create a Pipeline
- [ ] Understand PipelineRun
- [ ] Create a PipelineRun
- [ ] Execute a Pipeline
- [ ] Check PipelineRun status
- [ ] Understand TaskRuns created by a PipelineRun
- [ ] Create multiple Tasks
- [ ] Use `runAfter`
- [ ] Understand Pipeline parameters
- [ ] Pass parameters from PipelineRun
- [ ] Understand basic Workspace concept
- [ ] Understand CI/CD pipeline flow
- [ ] Troubleshoot Task failures
- [ ] Troubleshoot Pipeline failures
- [ ] Understand Tekton API resources
- [ ] Understand why `pipelines.tekton.dev` is required in this environment
- [ ] Understand the difference between Tekton Pipelines and Kubeflow Pipelines

---

# 🏁 Lesson 20 Goal


> **Tekton provides Kubernetes-native CI/CD building blocks. A Task defines one unit of work, a TaskRun executes that Task, a Pipeline combines multiple Tasks, and a PipelineRun executes the Pipeline. Parameters make Pipelines reusable for different inputs.**

The core model:

```text
Task
  ↓
TaskRun
  ↓
Pod
```

```text
Pipeline
  ↓
PipelineRun
  ↓
TaskRuns
  ↓
Pods
```

And the future real-world CI/CD model:

```text
Git
 ↓
Build
 ↓
Test
 ↓
Build Image
 ↓
Push Image
 ↓
Deploy
 ↓
Verify
```

### Environment-specific reminder

```text
This OpenShift cluster
        |
        +---- Kubeflow Pipelines
        |       pipelines.kubeflow.org
        |
        +---- Tekton Pipelines
                pipelines.tekton.dev
```

For **Lesson 20**, always use:

```text
tasks.tekton.dev
taskruns.tekton.dev
pipelines.tekton.dev
pipelineruns.tekton.dev
```

**Lesson 20 Topic: OpenShift CI/CD with Tekton – Tasks, TaskRuns, Pipelines, PipelineRuns & Parameters**
