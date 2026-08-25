# OpenShift Learning – Lesson 17: Stateful Applications

## 🚀 Lesson 17: StatefulSet, Stable Identity & Persistent Storage

> **Environment Note:** This lesson is designed for an OpenShift Developer Sandbox with restricted permissions. The focus is on beginner-friendly, hands-on StatefulSet and persistent-storage concepts.

---

## 🎯 Learning Objectives

By the end of this lesson, I should understand:

- Stateless vs Stateful applications
- What a StatefulSet is
- Deployment vs StatefulSet
- Stable Pod identity
- Predictable StatefulSet Pod names
- StatefulSet scaling
- StatefulSet Pod recreation
- Persistent storage with StatefulSet
- `volumeClaimTemplates`
- PersistentVolumeClaims created by StatefulSets
- Writing data to a PVC
- Verifying data after Pod recreation
- StatefulSet + Service relationship
- Basic StatefulSet troubleshooting
- Why StatefulSets are commonly used for databases

---

# 1. Stateless vs Stateful Applications

### Deployment

A Deployment is commonly used for stateless applications where Pods are interchangeable.

Example:

```text
lesson17-web-abc123
lesson17-web-def456
```

If one Pod disappears, another Pod can replace it.

```text
Deployment
    |
    +---- Pod A
    +---- Pod B
    +---- Pod C

Pods are generally interchangeable
```

### Stateful Application

A Stateful application needs stable identity and/or persistent data.

Examples:

```text
Database
Redis
Kafka
ZooKeeper
```

Conceptually:

```text
Stateless
    ↓
Pods are interchangeable

Stateful
    ↓
Stable identity
    +
Persistent data
```

---

# 2. What Is StatefulSet?

A `StatefulSet` is a Kubernetes/OpenShift workload resource designed for applications that need stable identity and persistent storage.

Think:

```text
Deployment
    ↓
Generic / stateless application

StatefulSet
    ↓
Stateful application
```

---

# 3. StatefulSet Pod Names

Deployments normally generate Pod names such as:

```text
lesson17-web-7d9f6c8d5
```

StatefulSets use predictable ordinal names:

```text
lesson17-db-0
lesson17-db-1
lesson17-db-2
```

The ordinal number provides stable identity:

```text
-0
-1
-2
```

---

# 4. Create Your First StatefulSet

Create:

```text
lesson17-statefulset.yaml
```

Use:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: lesson17-db
spec:
  serviceName: lesson17-db
  replicas: 1
  selector:
    matchLabels:
      app: lesson17-db
  template:
    metadata:
      labels:
        app: lesson17-db
    spec:
      containers:
        - name: db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Stateful application started"
              while true; do
                sleep 3600
              done
```

Apply:

```cmd
oc apply -f lesson17-statefulset.yaml
```

Check:

```cmd
oc get statefulsets
```

Expected:

```text
NAME          READY
lesson17-db   1/1
```

---

# 5. Check the StatefulSet Pod

Run:

```cmd
oc get pods
```

You should see:

```text
lesson17-db-0
```

Notice the predictable name:

```text
lesson17-db-0
```

---

# 6. Describe the StatefulSet

Run:

```cmd
oc describe statefulset lesson17-db
```

Look at:

```text
Replicas
Selector
Pod Template
Service Name
```

---

# 7. Delete the StatefulSet Pod

Delete:

```cmd
oc delete pod lesson17-db-0
```

Check:

```cmd
oc get pods
```

OpenShift/Kubernetes should recreate:

```text
lesson17-db-0
```

### Important Observation

The old Pod was deleted:

```text
lesson17-db-0
```

A new Pod was created with the same identity:

```text
lesson17-db-0
```

This demonstrates stable StatefulSet identity.

---

# 8. Scale the StatefulSet

Scale from 1 to 3 replicas:

```cmd
oc scale statefulset lesson17-db --replicas=3
```

Check:

```cmd
oc get pods
```

You should see:

```text
lesson17-db-0
lesson17-db-1
lesson17-db-2
```

---

# 9. StatefulSet Ordering

StatefulSets normally create Pods using ordinal identities.

Conceptually:

```text
lesson17-db-0
      ↓
lesson17-db-1
      ↓
lesson17-db-2
```

This is different from simply creating several interchangeable Pods.

---

# 10. Scale Back Down

Scale back to one replica:

```cmd
oc scale statefulset lesson17-db --replicas=1
```

Check:

```cmd
oc get pods
```

Eventually:

```text
lesson17-db-0
```

---

# 11. StatefulSet and Persistent Storage

The important relationship is:

```text
StatefulSet
     |
     v
Pod
     |
     v
PersistentVolumeClaim
     |
     v
PersistentVolume
     |
     v
Persistent Data
```

A Stateful application such as a database should not lose its data simply because its Pod is recreated.

---

# 12. `volumeClaimTemplates`

A StatefulSet can automatically create PVCs using:

```yaml
volumeClaimTemplates:
```

Example:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

Conceptually:

```text
StatefulSet
    |
    +---- Pod 0
    |       |
    |       +---- PVC data-lesson17-db-0
    |
    +---- Pod 1
            |
            +---- PVC data-lesson17-db-1
```

Each StatefulSet Pod can have its own persistent storage.

---

# 13. Create StatefulSet With Persistent Storage

Create:

```text
lesson17-storage.yaml
```

Use:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: lesson17-storage
spec:
  serviceName: lesson17-storage
  replicas: 1
  selector:
    matchLabels:
      app: lesson17-storage
  template:
    metadata:
      labels:
        app: lesson17-storage
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Lesson 17 Stateful Application"
              while true; do
                sleep 3600
              done
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

Apply:

```cmd
oc apply -f lesson17-storage.yaml
```

---

# 14. Check the StatefulSet

Run:

```cmd
oc get statefulset
```

Then:

```cmd
oc get pods
```

You should see:

```text
lesson17-storage-0
```

---

# 15. Check the PVC

Run:

```cmd
oc get pvc
```

You should see a PVC associated with the StatefulSet.

The name will normally look similar to:

```text
data-lesson17-storage-0
```

The important relationship is:

```text
StatefulSet
    ↓
Pod
    ↓
PVC
```

---

# 16. Check PVC Details

Run:

```cmd
oc describe pvc data-lesson17-storage-0
```

Look for:

```text
Status
Volume
Capacity
Access Modes
StorageClass
```

Normally, if dynamic provisioning succeeds:

```text
Status: Bound
```

---

# 17. Write Data to Persistent Storage

Get the Pod:

```cmd
oc get pods
```

Write data:

```cmd
oc exec lesson17-storage-0 -- sh -c "echo Hello-from-StatefulSet > /data/message.txt"
```

Read the data:

```cmd
oc exec lesson17-storage-0 -- cat /data/message.txt
```

Expected:

```text
Hello-from-StatefulSet
```

---

# 18. Delete the StatefulSet Pod

Delete:

```cmd
oc delete pod lesson17-storage-0
```

Wait for the new Pod:

```cmd
oc get pods
```

You should again get:

```text
lesson17-storage-0
```

Now check the data:

```cmd
oc exec lesson17-storage-0 -- cat /data/message.txt
```

Expected:

```text
Hello-from-StatefulSet
```

### What Did We Prove?

```text
Pod deleted
     ↓
New Pod created
     ↓
Same StatefulSet identity
     ↓
Same PVC
     ↓
Data still exists
```

This is the key practical concept of this lesson.

---

# 19. Pod vs PVC

Deleting the Pod:

```cmd
oc delete pod lesson17-storage-0
```

does not normally mean:

```text
Delete PVC
```

Therefore:

```text
Pod
 ↓
Deleted

PVC
 ↓
Still exists

Data
 ↓
Can remain available
```

---

# 20. StatefulSet + Service

Our StatefulSet contains:

```yaml
serviceName: lesson17-storage
```

StatefulSets use a Service as part of their stable network identity model.

For now, remember:

```text
StatefulSet
    +
Service
    ↓
Stable network identity
```

We will go deeper into DNS and service discovery later when needed.

---

# 21. Stateful Applications Examples

Common Stateful applications include:

```text
PostgreSQL
MySQL
MongoDB
Redis
Kafka
ZooKeeper
Elasticsearch
```

These applications commonly need:

```text
Stable identity
+
Persistent data
```

---

# 22. Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---|---|---|
| Typical use | Stateless apps | Stateful apps |
| Pod identity | Not stable | Stable |
| Pod names | Generated | Predictable |
| Ordering | Usually not important | Ordered identity |
| Persistent storage | Optional | Common |
| Database use | Usually not ideal | Common |
| Pod replacement | Interchangeable | Identity maintained |

Easy memory:

```text
Web application → Deployment

Database → StatefulSet
```

> This is a simplified rule. The actual choice depends on the application's requirements.

---

# 23. StatefulSet Troubleshooting

When a StatefulSet Pod is not starting:

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

Check PVCs:

```cmd
oc get pvc
```

Check PVC details:

```cmd
oc describe pvc <pvc-name>
```

Check Events:

```cmd
oc get events --sort-by=.lastTimestamp
```

### Troubleshooting Flow

```text
Pod problem
    ↓
oc get pods
    ↓
oc describe pod
    ↓
oc logs
    ↓
Check PVC
    ↓
Check Events
    ↓
Find root cause
```

---

# 24. Common StatefulSet Problems

## Problem 1 – PVC Pending

Possible flow:

```text
Pod Pending
     ↓
PVC Pending
     ↓
Storage provisioning problem
```

Check:

```cmd
oc get pvc
```

---

## Problem 2 – Pod Pending

Run:

```cmd
oc describe pod <pod-name>
```

Look at the Events section.

---

## Problem 3 – Permission Denied

Check:

```cmd
oc logs <pod-name>
```

The application may not have permission to write to the mounted directory.

---

## Problem 4 – Data Missing

Check:

```cmd
oc get pvc
```

Then verify that the correct PVC is mounted by inspecting:

```cmd
oc describe pod <pod-name>
```

---

# 🧠 Final Memory Trick

Remember:

```text
Deployment
    ↓
Stateless
    ↓
Pods are interchangeable
```

```text
StatefulSet
    ↓
Stable identity
    +
Persistent storage
    ↓
Stateful application
```

Most important concept:

```text
Pod
 ↓
Can disappear

PVC
 ↓
Can remain

Data
 ↓
Can survive Pod recreation
```

---

# 🔧 Important Commands

Create StatefulSet:

```cmd
oc apply -f lesson17-statefulset.yaml
```

List StatefulSets:

```cmd
oc get statefulsets
```

Describe StatefulSet:

```cmd
oc describe statefulset lesson17-db
```

List Pods:

```cmd
oc get pods
```

Scale StatefulSet:

```cmd
oc scale statefulset lesson17-db --replicas=3
```

Delete StatefulSet Pod:

```cmd
oc delete pod lesson17-db-0
```

List PVCs:

```cmd
oc get pvc
```

Describe PVC:

```cmd
oc describe pvc <pvc-name>
```

Read application logs:

```cmd
oc logs <pod-name>
```

Check Pod details:

```cmd
oc describe pod <pod-name>
```

Check Events:

```cmd
oc get events --sort-by=.lastTimestamp
```

Write data:

```cmd
oc exec lesson17-storage-0 -- sh -c "echo Hello-from-StatefulSet > /data/message.txt"
```

Read data:

```cmd
oc exec lesson17-storage-0 -- cat /data/message.txt
```

---

# ✅ Lesson 17 Completion Checklist

- [ ] Understand stateless vs stateful applications
- [ ] Understand StatefulSet
- [ ] Understand Deployment vs StatefulSet
- [ ] Create a StatefulSet
- [ ] Understand predictable Pod names
- [ ] Scale a StatefulSet
- [ ] Understand StatefulSet Pod ordering
- [ ] Delete a StatefulSet Pod
- [ ] Observe Pod recreation
- [ ] Understand persistent storage with StatefulSet
- [ ] Understand `volumeClaimTemplates`
- [ ] Create a StatefulSet with PVC
- [ ] Check PVC status
- [ ] Write data to PVC
- [ ] Delete the Pod
- [ ] Verify data survives Pod recreation
- [ ] Understand StatefulSet + Service relationship
- [ ] Understand basic StatefulSet troubleshooting
- [ ] Understand why databases commonly use StatefulSet

---

# 🏁 Lesson 17 Goal


> **A StatefulSet is used for applications that need stable identity and/or persistent storage. When a StatefulSet Pod is deleted, OpenShift recreates the Pod with its stable identity, while the associated persistent storage can preserve application data.**

The core model is:

```text
StatefulSet
     |
     +---- Stable Pod Identity
     |
     +---- Persistent Storage
     |
     +---- Persistent Application Data
```

**Lesson 17 Topic: Stateful Applications – StatefulSet, Stable Identity & Persistent Storage**
