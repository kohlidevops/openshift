# OpenShift Learning – Lesson 11: Jobs & CronJobs

## 🎯 Objectives

In this lesson, I learned and practiced:

- Jobs
- Jobs vs Deployments
- Job lifecycle
- Successful and failed Jobs
- `backoffLimit`
- `completions`
- `parallelism`
- CronJobs
- Cron schedule syntax
- CronJob → Job → Pod workflow
- Suspending and resuming CronJobs
- Job troubleshooting
- CronJob troubleshooting
- Real-world batch workload use cases

---
## 1. What is a Job?

A **Job** is used to run a task that should eventually complete successfully.

Examples:

- Database migration
- Database backup
- Data processing
- Report generation
- Cleanup tasks
- Batch processing
- One-time initialization

```text
Job
 |
 v
Pod
 |
 v
Task
 |
 v
Completed
```

Unlike a Deployment, the Pod created by a Job does not need to remain running continuously.

---
## 2. Job vs Deployment

| Deployment | Job |
|---|---|
| Runs an application continuously | Runs a task and completes |
| Keeps Pods running | Pod finishes after the task |
| Used for web applications/APIs | Used for batch/one-time workloads |
| Maintains desired replicas | Tracks successful completions |

Easy way to remember:

```text
Deployment → Keep application running

Job → Complete a task
```

---
## 3. Create First Job

Created `lesson11-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: lesson11-job
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "Hello from OpenShift Job"; sleep 10; echo "Job completed successfully"
```

Apply:

```bash
oc apply -f lesson11-job.yaml
```

Check:

```bash
oc get jobs
```

Expected:

```text
NAME           COMPLETIONS   DURATION   AGE
lesson11-job   1/1           10s        ...
```

`1/1` means:

```text
1 successful completion out of 1 required completion
```

---
## 4. Check the Job Pod

```bash
oc get pods
```

The Pod may show:

```text
NAME                 READY   STATUS
lesson11-job-xxxxx   0/1     Completed
```

`Completed` is expected for a successful Job.

```text
Running   → Task is still running

Completed → Task finished successfully
```

---
## 5. Check Job Details

```bash
oc describe job lesson11-job
```

Check:

- Completions
- Parallelism
- Pod Statuses
- Events

You can also check:

```bash
oc get job lesson11-job -o yaml
```

---
## 6. Check Job Logs

Find the Job Pod:

```bash
oc get pods
```

Then:

```bash
oc logs <job-pod-name>
```

Expected:

```text
Hello from OpenShift Job
Job completed successfully
```

---
## 7. Job Lifecycle

```text
Job
 |
 v
Pod Created
 |
 v
Container Runs
 |
 v
Task Completes
 |
 v
Pod = Completed
 |
 v
Job = 1/1
```

---
## 8. Failed Job

Created `lesson11-failed-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: lesson11-failed-job
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "Starting job"; exit 1
```

Apply:

```bash
oc apply -f lesson11-failed-job.yaml
```

Check:

```bash
oc get jobs
```

Check Pods:

```bash
oc get pods
```

Describe:

```bash
oc describe job lesson11-failed-job
```

---
## 9. Understanding `backoffLimit`

Example:

```yaml
backoffLimit: 2
```

This controls retries after Job failures.

Conceptually:

```text
Job
 |
 +--> Attempt 1 → Failed
 |
 +--> Attempt 2 → Failed
 |
 +--> Attempt 3 → Failed
 |
 v
Job Failed
```

Check:

```bash
oc describe job lesson11-failed-job
```

---
## 10. `completions`

`completions` defines how many successful executions are required.

Example:

```yaml
completions: 3
```

This means:

```text
Required successful completions = 3
```

Created `lesson11-completions.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: lesson11-completions
spec:
  completions: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "Processing task"; sleep 5
```

Apply:

```bash
oc apply -f lesson11-completions.yaml
```

Check:

```bash
oc get jobs
```

Eventually:

```text
COMPLETIONS
3/3
```

---
## 11. `parallelism`

`parallelism` defines how many Pods can run at the same time.

Example:

```yaml
completions: 4
parallelism: 2
```

This means:

```text
Required successful completions = 4

Maximum simultaneous Pods = 2
```

Created `lesson11-parallel.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: lesson11-parallel
spec:
  completions: 4
  parallelism: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: job
          image: busybox:1.36
          command:
            - sh
            - -c
            - echo "Processing task"; sleep 10
```

Apply:

```bash
oc apply -f lesson11-parallel.yaml
```

Watch Pods:

```bash
oc get pods -w
```

Check:

```bash
oc get job lesson11-parallel
```

Eventually:

```text
4/4
```

---
## 12. `completions` vs `parallelism`

```text
completions
     |
     v
How many successful tasks are required?

parallelism
     |
     v
How many tasks can run at the same time?
```

Example:

```yaml
completions: 6
parallelism: 2
```

Means:

```text
6 successful completions are required

Maximum 2 Pods can run simultaneously
```

---
## 13. What is a CronJob?

A CronJob creates Jobs according to a schedule.

```text
CronJob
   |
   | Schedule
   v
Job
   |
   v
Pod
   |
   v
Task
```

Easy way to remember:

```text
Job     → Run a task

CronJob → Schedule a task
```

---
## 14. Create First CronJob

Created `lesson11-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: lesson11-cronjob
spec:
  schedule: "*/2 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: cronjob
              image: busybox:1.36
              command:
                - sh
                - -c
                - echo "Hello from CronJob"; date
```

The schedule:

```text
*/2 * * * *
```

means:

```text
Every 2 minutes
```

Apply:

```bash
oc apply -f lesson11-cronjob.yaml
```

---
## 15. Check CronJob

```bash
oc get cronjobs
```

Example:

```text
NAME               SCHEDULE      SUSPEND   ACTIVE
lesson11-cronjob   */2 * * * *   False     0
```

Wait for the schedule to execute.

Then:

```bash
oc get jobs
```

A Job should be created.

Then:

```bash
oc get pods
```

The Job creates a Pod.

---
## 16. CronJob Workflow

```text
CronJob
   |
   | Every 2 minutes
   v
Job
   |
   v
Pod
   |
   v
Container
   |
   v
Task Completed
```

After the next scheduled execution:

```text
CronJob
   |
   v
Another Job
   |
   v
Another Pod
```

---
## 17. Cron Schedule Syntax

Cron has five fields:

```text
┌──────── minute
│ ┌────── hour
│ │ ┌──── day of month
│ │ │ ┌── month
│ │ │ │ ┌ day of week
│ │ │ │ │
* * * * *
```

Examples:

### Every minute

```text
* * * * *
```

### Every 5 minutes

```text
*/5 * * * *
```

### Every hour

```text
0 * * * *
```

### Every day at midnight

```text
0 0 * * *
```

### Every day at 2 AM

```text
0 2 * * *
```

### Every Monday at 9 AM

```text
0 9 * * 1
```

---
## 18. Check CronJob History

```bash
oc get jobs
```

You may see multiple Jobs:

```text
lesson11-cronjob-xxxxx
lesson11-cronjob-yyyyy
lesson11-cronjob-zzzzz
```

Each scheduled execution creates a Job.

```text
One CronJob
     |
     +---- Job 1
     |      |
     |      +---- Pod
     |
     +---- Job 2
     |      |
     |      +---- Pod
     |
     +---- Job 3
            |
            +---- Pod
```

---
## 19. Check CronJob Details

```bash
oc describe cronjob lesson11-cronjob
```

Check:

- Schedule
- Last Schedule Time
- Active Jobs
- Events

---
## 20. Suspend a CronJob

Temporarily stop scheduled executions:

```bash
oc patch cronjob lesson11-cronjob -p '{"spec":{"suspend":true}}'
```

Check:

```bash
oc get cronjob lesson11-cronjob
```

Expected:

```text
SUSPEND
true
```

---
## 21. Resume a CronJob

```bash
oc patch cronjob lesson11-cronjob -p '{"spec":{"suspend":false}}'
```

Check:

```bash
oc get cronjob lesson11-cronjob
```

The CronJob should resume its schedule.

---
## 22. Job Troubleshooting

If:

```bash
oc get jobs
```

shows:

```text
lesson11-job   0/1
```

Troubleshoot in this order:

### Step 1 – Describe Job

```bash
oc describe job lesson11-job
```

### Step 2 – Find the Pod

```bash
oc get pods
```

### Step 3 – Describe Pod

```bash
oc describe pod <pod-name>
```

### Step 4 – Check Logs

```bash
oc logs <pod-name>
```

### Step 5 – Check Events

```bash
oc get events --sort-by=.lastTimestamp
```

Troubleshooting flow:

```text
Job
 |
 v
Describe Job
 |
 v
Find Pod
 |
 v
Describe Pod
 |
 v
Check Logs
 |
 v
Check Events
```

---
## 23. CronJob Troubleshooting

If a CronJob is not creating Jobs:

### Check CronJob

```bash
oc get cronjob
```

### Describe CronJob

```bash
oc describe cronjob lesson11-cronjob
```

Check:

- Schedule
- Suspend status
- Last Schedule Time
- Events

### Check Jobs

```bash
oc get jobs
```

### Check Pods

```bash
oc get pods
```

---
## 24. Real-World Use Cases

### Jobs

```text
Database migration
Data processing
One-time initialization
Batch processing
Report generation
Cleanup tasks
```

### CronJobs

```text
Database backups
Log cleanup
Scheduled reports
Periodic data synchronization
Recurring batch processing
Cleanup operations
```

Example:

```text
Every night at 1 AM
       |
       v
CronJob
       |
       v
Database Backup Job
       |
       v
Backup Pod
       |
       v
Backup Completed
```

---
## 25. Important Concepts

### Job

```text
Run a task until it completes successfully.
```

### CronJob

```text
Run Jobs according to a schedule.
```

### Completions

```text
Number of successful completions required.
```

### Parallelism

```text
Maximum number of Pods running simultaneously.
```

### BackoffLimit

```text
Number of retries allowed after failure.
```

---
## 26. Hands-On Challenge

Create:

```text
lesson11-challenge
```

### Job

Requirements:

- Use `busybox:1.36`
- Print a message
- Sleep for 5 seconds
- Complete successfully
- Configure `backoffLimit`
- Verify the Pod becomes `Completed`

Verify:

```bash
oc get jobs
oc get pods
```

### Multiple Completions

Configure:

```text
completions: 4
parallelism: 2
```

Verify:

```bash
oc get jobs
oc get pods
```

### CronJob

Create a CronJob that runs every 2 minutes.

Verify:

```bash
oc get cronjobs
oc get jobs
oc get pods
```

Wait for at least two scheduled executions.

---
## 27. Interview Questions

1. What is a Job in OpenShift/Kubernetes?
2. What is the difference between a Job and Deployment?
3. What happens when a Job completes?
4. What is `backoffLimit`?
5. What is `completions`?
6. What is `parallelism`?
7. What is the difference between `completions` and `parallelism`?
8. What is a CronJob?
9. What is the difference between Job and CronJob?
10. Explain Cron schedule syntax.
11. How do you check whether a Job completed?
12. How do you troubleshoot a failed Job?
13. How do you troubleshoot a CronJob that is not creating Jobs?
14. How can you temporarily stop a CronJob?
15. Give some real-world use cases for Jobs and CronJobs.

---
## 28. Useful Commands

### Jobs

```bash
oc get jobs
oc describe job <job-name>
oc get job <job-name> -o yaml
```

### Pods

```bash
oc get pods
oc describe pod <pod-name>
oc logs <pod-name>
```

### CronJobs

```bash
oc get cronjobs
oc describe cronjob <cronjob-name>
oc get cronjob <cronjob-name> -o yaml
```

### Events

```bash
oc get events --sort-by=.lastTimestamp
```

### Suspend / Resume

```bash
oc patch cronjob <cronjob-name> -p '{"spec":{"suspend":true}}'
```

```bash
oc patch cronjob <cronjob-name> -p '{"spec":{"suspend":false}}'
```

## 🧹 Cleanup

Delete Jobs:

```bash
oc delete job lesson11-job
oc delete job lesson11-failed-job
oc delete job lesson11-completions
oc delete job lesson11-parallel
```

Delete CronJob:

```bash
oc delete cronjob lesson11-cronjob
```

---
## ✅ Lesson 11 Completion Checklist

- [x] Understand what a Job is
- [x] Understand Job vs Deployment
- [x] Create a Job
- [x] Check Job completion
- [x] Find the Pod created by a Job
- [x] Check Job logs
- [x] Understand failed Jobs
- [x] Understand `backoffLimit`
- [x] Understand `completions`
- [x] Understand `parallelism`
- [x] Create a CronJob
- [x] Understand Cron schedule syntax
- [x] See Jobs created by a CronJob
- [x] Suspend a CronJob
- [x] Resume a CronJob
- [x] Troubleshoot a failed Job
- [x] Troubleshoot a CronJob
- [x] Understand real-world Job/CronJob use cases

## 🧠 Final Memory Trick

```text
Deployment → Keep application running

Job → Run a task and finish

CronJob → Run Jobs on a schedule

completions → How many successful tasks?

parallelism → How many can run at once?

backoffLimit → How many retries after failure?
```


Final workflow:

```text
CronJob
   |
   | Schedule
   v
Job
   |
   | Creates
   v
Pod
   |
   | Runs
   v
Task
   |
   v
Completed / Failed
```
---
