# OpenShift Learning – Lesson 5

## Storage: Volumes, PersistentVolumes, PVCs & StorageClasses

In this lesson, I learned how to provide **persistent storage to applications running on OpenShift**.

I performed hands-on practice using the **Red Hat OpenShift Developer Sandbox**, including `emptyDir`, PersistentVolumeClaims (PVCs), StorageClasses, dynamic provisioning, access modes, persistent data, and troubleshooting a PVC stuck in `Pending`.

---

# 🎯 Learning Objectives

- Understand container storage
- Understand `emptyDir`
- Understand PersistentVolumes (PV)
- Understand PersistentVolumeClaims (PVC)
- Understand StorageClasses
- Understand dynamic provisioning
- Understand `ReadWriteOnce (RWO)`
- Understand `ReadWriteMany (RWX)`
- Understand `WaitForFirstConsumer`
- Create a PVC
- Mount a PVC into a Pod
- Write data to persistent storage
- Delete and recreate a Pod
- Verify data persistence
- Understand reclaim policies
- Troubleshoot a PVC stuck in `Pending`
- Understand CSI-based storage

---

# 1. Why Do We Need Persistent Storage?

Containers have their own filesystem.

For example:

```text
Pod
 |
 +-- Container
       |
       +-- /data
```

If an application writes:

```text
/data/file.txt
```

and the Pod is deleted, the container's temporary filesystem is lost.

```text
Old Pod
   |
   +-- file.txt
   |
   X Pod deleted
   |
   ▼
New Pod
   |
   +-- file.txt ❌
```

For applications that need data to survive Pod replacement, persistent storage is required.

---

# 2. OpenShift Storage Architecture

The main OpenShift storage architecture learned in this lesson:

```text
                    StorageClass
                         |
                         ▼
                Dynamic Provisioning
                         |
                         ▼
               PersistentVolume (PV)
                         |
                         ▼
            PersistentVolumeClaim (PVC)
                         |
                         ▼
                        Pod
                         |
                         ▼
                     Container
```

A developer normally creates a **PVC** to request storage.

The StorageClass and CSI provisioner can then dynamically create the underlying storage.

---

# 3. `emptyDir`

`emptyDir` provides temporary storage for a Pod.

Example:

### `lesson5-emptydir.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson5-emptydir
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Hello from emptyDir" > /data/test.txt
          echo "File created:"
          cat /data/test.txt
          sleep 3600
      volumeMounts:
        - name: temporary-storage
          mountPath: /data

  volumes:
    - name: temporary-storage
      emptyDir: {}
```

Apply:

```bash
oc apply -f lesson5-emptydir.yaml
```

Check:

```bash
oc get pod lesson5-emptydir
```

Expected:

```text
NAME               READY   STATUS
lesson5-emptydir   1/1     Running
```

Verify the file:

```bash
oc exec lesson5-emptydir -- cat /data/test.txt
```

Expected:

```text
Hello from emptyDir
```

---

# 4. `emptyDir` Behavior

The `emptyDir` volume exists for the lifetime of the Pod.

```text
Pod
 |
 +-- Container
 |
 +-- emptyDir
       |
       +-- /data/test.txt
```

When the Pod is deleted:

```text
Pod deleted
     |
     ▼
emptyDir deleted
     |
     ▼
Data lost
```

Therefore:

> `emptyDir` is temporary storage and should not be used for important persistent application data.

Typical uses:

- Temporary files
- Scratch space
- Cache
- Temporary processing
- Sharing data between containers in the same Pod

---

# 5. Check Available StorageClasses

The Developer Sandbox provides multiple StorageClasses.

Command:

```bash
oc get storageclass
```

The Sandbox showed:

```text
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION
efs-sc          efs.csi.aws.com         Delete          Immediate              false
gp2             kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true
gp2-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true
gp3 (default)   ebs.csi.aws.com         Delete          WaitForFirstConsumer   true
gp3-csi         ebs.csi.aws.com         Delete          WaitForFirstConsumer   true
```

The default StorageClass in this environment is:

```text
gp3
```

with:

```text
Provisioner:
ebs.csi.aws.com
```

and:

```text
VolumeBindingMode:
WaitForFirstConsumer
```

---

# 6. What is a StorageClass?

A StorageClass defines how storage should be dynamically provisioned.

Conceptually:

```text
PVC
 |
 | "I need 1Gi"
 ▼
StorageClass
 |
 ▼
CSI Provisioner
 |
 ▼
Actual Storage
 |
 ▼
PV
```

In this Sandbox environment:

```text
StorageClass
      |
      ▼
gp3
      |
      ▼
AWS EBS CSI Driver
      |
      ▼
AWS EBS Storage
```

The actual provisioner is:

```text
ebs.csi.aws.com
```

---

# 7. What is a PersistentVolumeClaim?

A **PersistentVolumeClaim (PVC)** is a request for persistent storage.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lesson5-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
oc apply -f lesson5-pvc.yaml
```

Check:

```bash
oc get pvc
```

---

# 8. PVC and PV

The relationship is:

```text
Application
     |
     ▼
    PVC
     |
     ▼
    PV
     |
     ▼
Actual Storage
```

### PVC

A request for storage.

### PV

The storage resource associated with the claim.

### StorageClass

Defines how storage can be dynamically provisioned.

---

# 9. PVC Status

Initially, a PVC may show:

```text
Pending
```

Once storage is successfully provisioned and associated:

```text
Bound
```

Example:

```text
NAME          STATUS   VOLUME
lesson5-pvc   Bound    pvc-xxxxxxxx
```

Meaning:

```text
PVC
 |
 +-- Bound
      |
      ▼
     PV
```

---

# 10. Important Discovery – `WaitForFirstConsumer`

The default `gp3` StorageClass in the Sandbox uses:

```text
VolumeBindingMode:
WaitForFirstConsumer
```

This is important.

When the PVC is created:

```text
PVC created
     |
     ▼
StorageClass = gp3
     |
     ▼
WaitForFirstConsumer
     |
     ▼
PVC remains Pending
```

This does **not necessarily mean there is an error**.

OpenShift waits until a Pod actually uses the PVC.

Once a Pod consumes the PVC:

```text
Pod uses PVC
     |
     ▼
StorageClass
     |
     ▼
AWS EBS CSI Driver
     |
     ▼
EBS volume provisioned
     |
     ▼
PV created
     |
     ▼
PVC becomes Bound
```

---

# 11. Create the PVC

### `lesson5-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lesson5-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
oc apply -f lesson5-pvc.yaml
```

Check:

```bash
oc get pvc lesson5-pvc
```

The initial result can be:

```text
NAME          STATUS    VOLUME   CAPACITY
lesson5-pvc   Pending
```

Because the StorageClass uses:

```text
WaitForFirstConsumer
```

---

# 12. What is `ReadWriteOnce`?

The PVC uses:

```yaml
accessModes:
  - ReadWriteOnce
```

`ReadWriteOnce` is commonly abbreviated as:

```text
RWO
```

Conceptually:

```text
Read + Write
      |
      ▼
Single Node
```

Other common access modes include:

```text
ReadWriteOnce (RWO)
ReadOnlyMany  (ROX)
ReadWriteMany (RWX)
```

---

# 13. RWO

### ReadWriteOnce

```text
RWO
 |
 +-- Read
 |
 +-- Write
 |
 +-- Single Node
```

This is commonly used with block storage such as AWS EBS.

---

# 14. RWX

### ReadWriteMany

```text
RWX
 |
 +-- Read
 |
 +-- Write
 |
 +-- Multiple Nodes
```

RWX requires a storage backend that supports shared read/write access.

For example, the Sandbox also provides an EFS-based StorageClass:

```text
efs-sc
```

with:

```text
efs.csi.aws.com
```

---

# 15. Use the PVC from a Pod

Create:

### `lesson5-pvc-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson5-storage-pod
spec:
  containers:
    - name: app
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Persistent data created by OpenShift Lesson 5" > /data/message.txt
          echo "File written to persistent storage."
          sleep 3600

      volumeMounts:
        - name: persistent-storage
          mountPath: /data

  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: lesson5-pvc
```

Apply:

```bash
oc apply -f lesson5-pvc-pod.yaml
```

---

# 16. Check the Pod

```bash
oc get pod lesson5-storage-pod
```

Expected:

```text
NAME                  READY   STATUS
lesson5-storage-pod   1/1     Running
```

Now check the PVC:

```bash
oc get pvc lesson5-pvc
```

The PVC should now become:

```text
Bound
```

because the Pod is consuming it.

---

# 17. Verify Persistent Data

Check the file:

```bash
oc exec lesson5-storage-pod -- cat /data/message.txt
```

Expected:

```text
Persistent data created by OpenShift Lesson 5
```

This confirms that the Pod successfully mounted the PVC.

---

# 18. Persistence Test

This was the most important hands-on exercise.

First, the Pod created:

```text
/data/message.txt
```

Then the Pod was deleted:

```bash
oc delete pod lesson5-storage-pod
```

The Pod was recreated:

```bash
oc apply -f lesson5-pvc-pod.yaml
```

After the new Pod became `Running`:

```bash
oc get pod lesson5-storage-pod
```

The file was checked again:

```bash
oc exec lesson5-storage-pod -- cat /data/message.txt
```

The data was still available:

```text
Persistent data created by OpenShift Lesson 5
```

---

# 19. Why Did the Data Survive?

The Pod was deleted, but the PVC and persistent storage remained.

```text
Old Pod
   |
   X Deleted
   |
   ▼
PVC
   |
   ▼
Persistent Storage
   |
   +-- message.txt
```

New Pod:

```text
New Pod
   |
   ▼
Same PVC
   |
   ▼
Same Persistent Storage
   |
   ▼
message.txt
```

Therefore:

> Pod lifecycle and persistent storage lifecycle are different.

---

# 20. Important Storage Architecture

```text
                         Application
                              |
                              ▼
                             Pod
                              |
                              ▼
                             PVC
                              |
                              ▼
                        StorageClass
                              |
                              ▼
                       CSI Provisioner
                              |
                              ▼
                       Persistent Storage
```

In this OpenShift Sandbox:

```text
PVC
 |
 ▼
gp3 StorageClass
 |
 ▼
AWS EBS CSI Driver
 |
 ▼
AWS EBS
```

---

# 21. CSI Driver

CSI stands for:

```text
Container Storage Interface
```

The CSI driver allows Kubernetes/OpenShift to communicate with external storage systems.

In this Sandbox:

```text
ebs.csi.aws.com
```

is the AWS EBS CSI provisioner.

Another available StorageClass was:

```text
efs-sc
```

using:

```text
efs.csi.aws.com
```

---

# 22. Reclaim Policy

The StorageClasses in the Sandbox showed:

```text
RECLAIMPOLICY
Delete
```

Common reclaim policies include:

```text
Delete
Retain
```

### Delete

When a dynamically provisioned PVC is deleted, the associated storage may also be deleted depending on the provisioner.

Conceptually:

```text
PVC deleted
    |
    ▼
PV deleted
    |
    ▼
Underlying storage deleted
```

### Retain

The storage can be retained for administrator intervention.

```text
PVC deleted
    |
    ▼
PV retained
    |
    ▼
Data may be recovered
```

---

# 23. StorageClass Details

Inspect the StorageClass:

```bash
oc describe storageclass gp3
```

Important fields include:

```text
Provisioner
Reclaim Policy
VolumeBindingMode
AllowVolumeExpansion
```

For this Sandbox:

```text
StorageClass: gp3
Provisioner: ebs.csi.aws.com
Reclaim Policy: Delete
VolumeBindingMode: WaitForFirstConsumer
AllowVolumeExpansion: true
```

---

# 24. PersistentVolume Permissions

During the lesson, the following command was attempted:

```bash
oc get pv
```

The Sandbox returned:

```text
Error from server (Forbidden):
persistentvolumes is forbidden
```

This happens because PersistentVolumes are **cluster-scoped resources** and the Developer Sandbox user does not have permission to list them.

However, the project-level PVC can be accessed:

```bash
oc get pvc
```

This demonstrates an important OpenShift security concept:

```text
Project-scoped resources
        |
        +-- Pods
        +-- PVCs
        +-- Services
        +-- Deployments
        +-- ConfigMaps
        +-- Secrets

Cluster-scoped resources
        |
        +-- PersistentVolumes
        +-- Nodes
        +-- StorageClasses
```

A normal developer account may have access to project resources without having cluster-admin permissions.

---

# 25. Troubleshooting PVC Pending

A PVC may show:

```text
Pending
```

Do not immediately assume that storage is broken.

First check:

```bash
oc get pvc
```

Then:

```bash
oc describe pvc lesson5-pvc
```

Look at:

```text
Events:
```

Then check the StorageClass:

```bash
oc get storageclass
```

And:

```bash
oc describe storageclass gp3
```

Important things to check:

```text
StorageClass
Provisioner
VolumeBindingMode
Access Mode
Storage Size
Events
```

---

# 26. PVC Troubleshooting Flow

```text
PVC Pending
     |
     ▼
oc get pvc
     |
     ▼
oc describe pvc
     |
     ▼
Check Events
     |
     ▼
Check StorageClass
     |
     +-----------------------------+
     |                             |
     ▼                             ▼
Immediate                  WaitForFirstConsumer
     |                             |
     ▼                             ▼
Provision storage          Is a Pod using PVC?
                                   |
                                   ▼
                                  Yes
                                   |
                                   ▼
                          Provision storage
                                   |
                                   ▼
                              PVC Bound
```

---

# 27. Intentional PVC Failure Test

A broken PVC was also created to understand troubleshooting.

### `lesson5-bad-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lesson5-bad-pvc
spec:
  storageClassName: does-not-exist
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
oc apply -f lesson5-bad-pvc.yaml
```

Check:

```bash
oc get pvc lesson5-bad-pvc
```

Expected:

```text
NAME              STATUS
lesson5-bad-pvc   Pending
```

Troubleshoot:

```bash
oc describe pvc lesson5-bad-pvc
```

The Events section provides the reason why dynamic provisioning cannot succeed.

After the exercise:

```bash
oc delete pvc lesson5-bad-pvc
```

---

# 28. Stateless vs Stateful Applications

### Stateless

Example:

```text
NGINX
 |
 +-- Pod
```

The application does not depend on local persistent data.

Pods can be replaced easily.

### Stateful

Example:

```text
PostgreSQL
    |
    +-- Pod
    |
    +-- PVC
         |
         +-- Persistent Data
```

If the Pod is replaced:

```text
Old PostgreSQL Pod
        |
        X
        |
        ▼
New PostgreSQL Pod
        |
        ▼
Same PVC
        |
        ▼
Same Data
```

This is why persistent storage is important for databases and other stateful applications.

---

# 29. Important Commands Learned

## StorageClasses

```bash
oc get storageclass
```

```bash
oc describe storageclass gp3
```

---

## PVCs

```bash
oc get pvc
```

```bash
oc describe pvc lesson5-pvc
```

```bash
oc get pvc lesson5-pvc -o yaml
```

---

## Pods

```bash
oc get pods
```

```bash
oc describe pod lesson5-storage-pod
```

```bash
oc logs lesson5-storage-pod
```

---

## Verify Persistent Data

```bash
oc exec lesson5-storage-pod -- cat /data/message.txt
```

---

## Delete Pod

```bash
oc delete pod lesson5-storage-pod
```

---

# 30. Important Limitation in Developer Sandbox

The Developer Sandbox is a shared managed OpenShift environment.

Therefore, some cluster-level operations are intentionally restricted.

For example:

```bash
oc get pv
```

may return:

```text
Forbidden
```

This does not prevent learning PVCs and persistent storage.

The important developer workflow is:

```text
Create PVC
     |
     ▼
Use PVC from Pod
     |
     ▼
Storage dynamically provisioned
     |
     ▼
Verify PVC
     |
     ▼
Use persistent data
```

---

# 31. Hands-on Checklist

- [x] Learned container storage
- [x] Created `emptyDir`
- [x] Wrote data to `emptyDir`
- [x] Learned that `emptyDir` is temporary
- [x] Listed available StorageClasses
- [x] Identified `gp3` as the default StorageClass
- [x] Identified AWS EBS CSI provisioner
- [x] Learned PersistentVolume
- [x] Learned PersistentVolumeClaim
- [x] Created a PVC
- [x] Observed PVC `Pending`
- [x] Learned `WaitForFirstConsumer`
- [x] Created a Pod consuming the PVC
- [x] Observed PVC transition to `Bound`
- [x] Mounted PVC into a Pod
- [x] Wrote data to persistent storage
- [x] Deleted the Pod
- [x] Recreated the Pod
- [x] Verified that the data persisted
- [x] Learned RWO
- [x] Learned RWX
- [x] Learned CSI
- [x] Learned reclaim policies
- [x] Troubleshot a `Pending` PVC
- [x] Understood Developer Sandbox cluster-level restrictions

---

# 🧠 Key Takeaways

### Volume

> Provides storage to a container.

### `emptyDir`

> Temporary storage associated with a Pod.

### PersistentVolume

> Represents persistent storage available to the cluster.

### PersistentVolumeClaim

> A request for persistent storage from an application.

### StorageClass

> Defines how storage is dynamically provisioned.

### Dynamic Provisioning

> Automatically creates storage when a suitable PVC is requested and consumed.

### CSI

> Container Storage Interface allows OpenShift to integrate with external storage systems.

### RWO

> ReadWriteOnce provides read/write access from a single node.

### RWX

> ReadWriteMany provides read/write access from multiple nodes when supported by the storage backend.

### `WaitForFirstConsumer`

> Delays volume provisioning/binding until a Pod actually consumes the PVC.

### Persistent Storage

> Allows application data to survive Pod deletion and recreation.

---

# 🎤 Interview Questions

## 1. What is a PersistentVolume?

A PersistentVolume represents persistent storage available to the OpenShift/Kubernetes cluster.

---

## 2. What is a PersistentVolumeClaim?

A PVC is a request from an application for persistent storage.

---

## 3. What is a StorageClass?

A StorageClass defines how storage should be dynamically provisioned.

---

## 4. What is dynamic provisioning?

Dynamic provisioning automatically creates the required persistent storage when a PVC requests it.

---

## 5. What is the difference between PV and PVC?

```text
PV
 |
 +-- Represents available/provisioned storage

PVC
 |
 +-- Requests storage
```

---

## 6. What is `emptyDir`?

`emptyDir` provides temporary storage for a Pod and its data is lost when the Pod is deleted.

---

## 7. What is the difference between `emptyDir` and PVC?

```text
emptyDir
 |
 +-- Temporary
 +-- Pod lifecycle

PVC
 |
 +-- Persistent
 +-- Independent of individual Pod lifecycle
```

---

## 8. What is `ReadWriteOnce`?

RWO allows a volume to be mounted with read/write access from a single node.

---

## 9. What is `ReadWriteMany`?

RWX allows a volume to be mounted with read/write access from multiple nodes, provided the storage backend supports it.

---

## 10. Why was our PVC initially `Pending`?

The default `gp3` StorageClass uses:

```text
WaitForFirstConsumer
```

Therefore, storage provisioning waits until a Pod actually consumes the PVC.

---

## 11. How do you troubleshoot a Pending PVC?

Use:

```bash
oc get pvc
oc describe pvc <pvc-name>
oc get storageclass
oc describe storageclass <storage-class-name>
```

Then inspect the PVC Events.

---

## 12. What is a CSI driver?

CSI stands for Container Storage Interface. It allows Kubernetes/OpenShift to integrate with external storage systems.

---

## 13. What CSI driver was available in this Sandbox?

The default `gp3` StorageClass uses:

```text
ebs.csi.aws.com
```

---

## 14. What happens to PVC data when a Pod is deleted?

The data remains available through the persistent storage as long as the PVC and underlying storage remain.

---

## 15. What is a reclaim policy?

A reclaim policy defines what happens to persistent storage when its associated PVC is deleted.

Common policies include:

```text
Delete
Retain
```

---


# ✅ Lesson 5 Completed

The main architecture learned in this lesson:

```text
                         Application
                              |
                              ▼
                             Pod
                              |
                              ▼
                             PVC
                              |
                              ▼
                        StorageClass
                              |
                              ▼
                       CSI Provisioner
                              |
                              ▼
                     Persistent Storage
```

For the OpenShift Developer Sandbox:

```text
PVC
 |
 ▼
gp3 StorageClass
 |
 ▼
AWS EBS CSI Driver
 |
 ▼
AWS EBS Storage
```

The most important practical demonstration was:

```text
Create PVC
    |
    ▼
PVC initially Pending
    |
    ▼
Create Pod using PVC
    |
    ▼
Storage provisioned
    |
    ▼
PVC becomes Bound
    |
    ▼
Write data
    |
    ▼
Delete Pod
    |
    ▼
Recreate Pod
    |
    ▼
Data still exists
```

## Final Takeaway

> **Pods are disposable, but application data should not be. PersistentVolumeClaims provide applications with persistent storage that can survive Pod replacement. StorageClasses and CSI drivers enable OpenShift to dynamically provision cloud storage.**

An important troubleshooting lesson from this exercise was that:

```text
PVC Pending
```

does not always mean an error.

Because the Sandbox `gp3` StorageClass uses:

```text
WaitForFirstConsumer
```

the PVC can remain `Pending` until a Pod actually consumes it.

Also, the Developer Sandbox restricts cluster-scoped operations such as:

```bash
oc get pv
```

while still allowing normal project-level PVC operations.

This demonstrates an important real-world OpenShift principle:

```text
Developer
   |
   +-- Works mainly inside Project/Namespace
   |
   +-- Uses PVC to request storage
   |
   ▼
Platform/Cluster
   |
   +-- Manages PV
   +-- Manages StorageClass
   +-- Manages CSI infrastructure
```
