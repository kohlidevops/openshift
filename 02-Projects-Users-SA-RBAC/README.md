# OpenShift Learning – Lesson 2: Projects, Users and RBAC

## 🚀 Lesson 2: Projects, Users, ServiceAccounts and RBAC

In this lesson, I learned how OpenShift manages **projects, users, groups, ServiceAccounts and permissions using RBAC**.

I also performed hands-on exercises in the **Red Hat OpenShift Developer Sandbox** to understand how identities receive permissions and how those permissions can be tested and removed.

The lesson covers the OpenShift course topics around projects, users, ServiceAccounts, groups, authentication/authorization and RBAC.

---

# 🎯 Learning Objectives

By completing this lesson, I learned:

* OpenShift Projects
* Projects vs Kubernetes Namespaces
* OpenShift Users
* Authentication
* Authorization
* Groups
* ServiceAccounts
* Roles
* RoleBindings
* ClusterRoles
* ClusterRoleBindings
* RBAC verbs
* `oc auth can-i`
* Least privilege
* Namespace/project-scoped permissions
* Cluster-scoped permissions
* How applications use ServiceAccounts
* How to grant and remove permissions

---

# 1. OpenShift Project

An OpenShift Project provides a logical workspace and isolation boundary for applications and resources.

My OpenShift Sandbox project is:

```text
lakshminarayananredh-dev
```

Check the current project:

```bash
oc project
```

List accessible projects:

```bash
oc projects
```

Display project information:

```bash
oc get project
```

---

# 2. Project vs Namespace

Kubernetes uses:

```text
Namespace
```

OpenShift provides:

```text
Project
```

A Project is an OpenShift abstraction around a Kubernetes namespace with additional OpenShift project management and access-control capabilities.

Conceptually:

```text
OpenShift Cluster
       |
       +-- Project A
       |     |
       |     +-- Pods
       |     +-- Services
       |     +-- Deployments
       |     +-- Secrets
       |
       +-- Project B
             |
             +-- Pods
             +-- Services
             +-- Deployments
```

A project helps isolate workloads between teams, applications or environments.

---

# 3. Project Commands

### Display current project

```bash
oc project
```

### Switch project

```bash
oc project lakshminarayananredh-dev
```

### List projects

```bash
oc projects
```

### Get project information

```bash
oc get project
```

### View project YAML

```bash
oc get project lakshminarayananredh-dev -o yaml
```

> **Developer Sandbox limitation:** The Sandbox account does not allow arbitrary project creation using `oc new-project`. Attempting to create another project resulted in:

```text
Error from server (Forbidden):
You may not request a new project via this API.
```

Therefore, all Lesson 2 hands-on exercises were performed inside the existing Sandbox project.

---

# 4. Users

A User represents a human identity that authenticates to OpenShift.

Check the currently authenticated user:

```bash
oc whoami
```

Example concept:

```text
User
 |
 +-- Authentication
 |
 +-- Authorization
 |
 +-- Resource Access
```

OpenShift uses authentication to identify a user and authorization to determine what the user is allowed to do.

---

# 5. Authentication vs Authorization

This is an important DevOps and security concept.

### Authentication

Authentication answers:

> **Who are you?**

Example:

```text
User
  |
  ▼
Login
  |
  ▼
OpenShift
  |
  ▼
Authenticated Identity
```

### Authorization

Authorization answers:

> **What are you allowed to do?**

Example:

```text
Authenticated User
       |
       ▼
Authorization
       |
       +-- Can view Pods?       YES
       +-- Can create Pods?     YES/NO
       +-- Can modify RBAC?    YES/NO
```

### Easy way to remember

```text
Authentication = Who are you?
Authorization  = What can you do?
```

---

# 6. Useful User Commands

### Current user

```bash
oc whoami
```

### OpenShift API server

```bash
oc whoami --show-server
```

### OpenShift Console URL

```bash
oc whoami --show-console
```

### View user objects

```bash
oc get users
```

> In a Developer Sandbox, cluster-level user information may be restricted and can return a `Forbidden` error. `oc whoami` can still be used to identify the currently authenticated user.

---

# 7. Groups

A Group is a collection of users.

Example:

```text
Developers
   |
   +-- developer1
   +-- developer2
   +-- developer3
```

Instead of assigning permissions individually:

```text
developer1 → permissions
developer2 → permissions
developer3 → permissions
```

permissions can be assigned to the group:

```text
Developers Group
       |
       ▼
Developer Permissions
```

This is useful in enterprise environments because permissions can be managed centrally.

List groups:

```bash
oc get groups
```

> Cluster-level group management may be restricted in the Developer Sandbox.

---

# 8. ServiceAccounts

A ServiceAccount represents an identity for an application or workload rather than a human user.

### Human

```text
Human
  |
  ▼
User
```

### Application

```text
Application
  |
  ▼
ServiceAccount
```

For example:

```text
Jenkins
   |
   ▼
ServiceAccount
   |
   ▼
OpenShift API
```

Using a dedicated ServiceAccount is preferable to giving an application a human user's credentials.

---

# 9. View ServiceAccounts

List ServiceAccounts:

```bash
oc get serviceaccounts
```

Inspect a ServiceAccount:

```bash
oc describe serviceaccount devops-sa
```

---

# 10. Create a ServiceAccount

Create an application ServiceAccount:

```bash
oc create serviceaccount app-sa
```

Verify:

```bash
oc get serviceaccounts
```

Example:

```text
NAME
----
builder
default
deployer
app-sa
devops-sa
```

The exact list may vary depending on the OpenShift environment.

---

# 11. Test ServiceAccount Permissions

A newly created ServiceAccount does not automatically receive the permissions we want for our application.

Test whether `app-sa` can read Pods:

```bash
oc auth can-i get pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
no
```

Test whether it can create Deployments:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
no
```

This demonstrates that permissions should be explicitly granted.

---

# 12. RBAC

RBAC stands for:

> **Role-Based Access Control**

RBAC controls which users, groups and ServiceAccounts can perform specific actions on OpenShift/Kubernetes resources.

Basic model:

```text
Identity
   |
   ▼
Role
   |
   ▼
RoleBinding
   |
   ▼
Resource Permissions
```

Identities can include:

```text
User
Group
ServiceAccount
```

---

# 13. Role

A Role defines permissions within a specific project/namespace.

Example:

```text
Role: pod-reader

Permissions:
    get pods
    list pods
    watch pods
```

A Role defines **what can be done**, but does not by itself assign those permissions to an identity.

---

# 14. Role Commands

List Roles:

```bash
oc get roles
```

Inspect a Role:

```bash
oc describe role pod-reader
```

---

# 15. Create a Custom Role

I created a custom Role called `pod-reader`.

### `pod-reader-role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

Apply the Role:

```bash
oc apply -f pod-reader-role.yaml
```

Verify:

```bash
oc get roles
```

Inspect:

```bash
oc describe role pod-reader
```

The Role provides:

```text
get pods
list pods
watch pods
```

It does not provide:

```text
create pods
delete pods
update pods
```

---

# 16. RoleBinding

A RoleBinding connects an identity to a Role.

Think:

```text
WHO
 |
 +-- User
 +-- Group
 +-- ServiceAccount
 |
 ▼
RoleBinding
 |
 ▼
Role
 |
 ▼
Permissions
```

Example:

```text
app-sa
   |
   ▼
RoleBinding
   |
   ▼
pod-reader
```

Now `app-sa` receives the permissions defined by `pod-reader`.

---

# 17. Create a RoleBinding

Bind the `pod-reader` Role to `app-sa`:

```bash
oc create rolebinding app-pod-reader \
  --role=pod-reader \
  --serviceaccount=lakshminarayananredh-dev:app-sa
```

For PowerShell, it can also be written as one line:

```powershell
oc create rolebinding app-pod-reader --role=pod-reader --serviceaccount=lakshminarayananredh-dev:app-sa
```

---

# 18. Verify RoleBinding

List RoleBindings:

```bash
oc get rolebindings
```

Inspect the specific RoleBinding:

```bash
oc describe rolebinding app-pod-reader
```

The relationship is:

```text
app-sa
   |
   ▼
RoleBinding
   |
   ▼
pod-reader
   |
   +-- get pods
   +-- list pods
   +-- watch pods
```

---

# 19. Test the Granted Permissions

Test `get`:

```bash
oc auth can-i get pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
yes
```

Test `list`:

```bash
oc auth can-i list pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
yes
```

Test `watch`:

```bash
oc auth can-i watch pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
yes
```

---

# 20. Test Permissions That Are NOT Granted

Try to create a Pod:

```bash
oc auth can-i create pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
no
```

Try to delete a Pod:

```bash
oc auth can-i delete pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
no
```

Try to create a Deployment:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
no
```

This demonstrates **least privilege**.

---

# 21. Expand the Custom Role

The Role can be modified to include Services.

Updated `pod-reader-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch

  - apiGroups: [""]
    resources:
      - services
    verbs:
      - get
      - list
      - watch
```

Apply:

```bash
oc apply -f pod-reader-role.yaml
```

Test:

```bash
oc auth can-i get services \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

Expected:

```text
yes
```

But:

```bash
oc auth can-i create services \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

should return:

```text
no
```

---

# 22. Built-in `view` Role

OpenShift provides predefined roles such as:

```text
view
edit
admin
```

Conceptually:

```text
view
 |
 +-- Read resources

edit
 |
 +-- Read
 +-- Create
 +-- Update
 +-- Delete many application resources

admin
 |
 +-- Project-level administration
```

Inspect available ClusterRoles:

```bash
oc get clusterroles
```

Filter common roles:

```powershell
oc get clusterroles | findstr "view edit admin"
```

---

# 23. Grant `view` Role

Grant the `view` role to `devops-sa`:

```bash
oc policy add-role-to-user view -z devops-sa
```

The `-z` option identifies the subject as a ServiceAccount.

Test:

```bash
oc auth can-i get pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

Expected:

```text
yes
```

Test:

```bash
oc auth can-i get deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

Expected:

```text
yes
```

Test whether it can create Deployments:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

Expected:

```text
no
```

---

# 24. Grant `edit` Role

Remove `view`:

```bash
oc policy remove-role-from-user view -z devops-sa
```

Grant `edit`:

```bash
oc policy add-role-to-user edit -z devops-sa
```

Test:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

Expected:

```text
yes
```

Test:

```bash
oc auth can-i delete deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

Expected:

```text
yes
```

This demonstrates the difference between read-only and application-editing permissions.

---

# 25. Grant `admin` Role

Remove `edit` first:

```bash
oc policy remove-role-from-user edit -z devops-sa
```

Grant `admin`:

```bash
oc policy add-role-to-user admin -z devops-sa
```

Test:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

Expected:

```text
yes
```

Test RoleBinding-related permissions:

```bash
oc auth can-i create rolebindings \
  --as=system:serviceaccount:lakshminarayananredh-dev:devops-sa
```

This demonstrates a higher level of project administration.

### Important

Do not leave unnecessary administrative permissions assigned.

Remove the role after testing:

```bash
oc policy remove-role-from-user admin -z devops-sa
```

---

# 26. ClusterRole

A ClusterRole defines a set of permissions that can be used at cluster scope or can also be bound within a namespace.

Examples of common built-in ClusterRoles:

```text
view
edit
admin
cluster-admin
```

Check:

```bash
oc get clusterroles
```

Inspect:

```bash
oc describe clusterrole view
```

```bash
oc describe clusterrole edit
```

```bash
oc describe clusterrole admin
```

---

# 27. Role vs ClusterRole

### Role

```text
Role
 |
 +-- Namespace/Project scoped
```

Example:

```text
Project: myapp-dev
      |
      └── Role
```

### ClusterRole

```text
ClusterRole
 |
 +-- Cluster-level role definition
```

Example:

```text
OpenShift Cluster
       |
       └── ClusterRole
```

A ClusterRole can be bound with a RoleBinding to grant its permissions within a particular namespace/project, or with a ClusterRoleBinding for cluster-wide access.

---

# 28. ClusterRoleBinding

A ClusterRoleBinding connects a ClusterRole to a User, Group or ServiceAccount at cluster scope.

Conceptually:

```text
User / Group / ServiceAccount
             |
             ▼
    ClusterRoleBinding
             |
             ▼
        ClusterRole
             |
             ▼
    Cluster-wide permissions
```

For example:

```text
cluster-admin
      +
ClusterRoleBinding
      +
Administrator
```

This is extremely powerful and should be used carefully.

> **Developer Sandbox limitation:** Creating or modifying cluster-wide RBAC objects may be restricted. The important objective here is to understand the architecture and inspect the existing ClusterRoles.

---

# 29. RBAC Verbs

RBAC permissions are defined using verbs.

Common verbs:

```text
get
list
watch
create
update
patch
delete
deletecollection
```

### Meaning

| Verb     | Meaning                     |
| -------- | --------------------------- |
| `get`    | Read one resource           |
| `list`   | Read multiple resources     |
| `watch`  | Watch changes               |
| `create` | Create a resource           |
| `update` | Replace/update a resource   |
| `patch`  | Partially update a resource |
| `delete` | Delete a resource           |

Example:

```yaml
verbs:
  - get
  - list
  - watch
```

means read-only access to the specified resources.

---

# 30. `oc auth can-i`

One of the most useful RBAC troubleshooting commands is:

```bash
oc auth can-i
```

Basic syntax:

```bash
oc auth can-i <verb> <resource>
```

Example:

```bash
oc auth can-i get pods
```

For a ServiceAccount:

```bash
oc auth can-i get pods \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

You can also list all permissions available to an identity:

```bash
oc auth can-i --list \
  --as=system:serviceaccount:lakshminarayananredh-dev:app-sa
```

This is very useful when troubleshooting:

```text
403 Forbidden
Permission denied
Cannot create resource
Cannot read resource
Cannot modify resource
```

---

# 31. Least Privilege

The RBAC lab demonstrates the security principle of **least privilege**.

Instead of giving an application:

```text
cluster-admin
```

we should give it only what it needs.

Example:

```text
Application
     |
     ▼
ServiceAccount
     |
     ▼
Role
     |
     +-- get pods
     +-- list pods
     +-- watch pods
```

Instead of:

```text
Application
     |
     ▼
cluster-admin ❌
```

The first approach is much safer.

---

# 32. Real-World Enterprise RBAC Model

A typical enterprise environment may have:

### Developers

```text
Developers
   |
   ▼
Project-level permissions
   |
   +-- Deploy applications
   +-- Update applications
   +-- View logs
   +-- View Pods
```

But they may not be allowed to:

```text
Modify RBAC
Modify cluster configuration
Manage cluster-wide resources
```

### DevOps Team

```text
DevOps
   |
   ▼
Higher project-level permissions
   |
   +-- Deploy
   +-- Troubleshoot
   +-- Manage application resources
   +-- Manage project RBAC
```

### Applications

```text
Application
   |
   ▼
ServiceAccount
   |
   ▼
Minimum required permissions
```

### Cluster Administrators

```text
Cluster Administrator
        |
        ▼
Cluster-wide administration
```

---

# 33. Complete RBAC Architecture

The complete model can be remembered as:

```text
                         OpenShift
                            |
                            ▼
                   Authentication
                            |
                    "Who are you?"
                            |
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
        User              Group        ServiceAccount
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                    Authorization
                            |
                   "What can you do?"
                            |
             ┌──────────────┴──────────────┐
             ▼                             ▼
           Role                       ClusterRole
             │                             │
             ▼                             ▼
       RoleBinding                  ClusterRoleBinding
             │                             │
             ▼                             ▼
       Project Scope                Cluster Scope
```

---

# 34. Cleanup

Remove permissions from `devops-sa`:

```bash
oc policy remove-role-from-user admin -z devops-sa
```

```bash
oc policy remove-role-from-user edit -z devops-sa
```

```bash
oc policy remove-role-from-user view -z devops-sa
```

If OpenShift reports that a role is not currently bound, that is fine.

Delete the test RoleBinding:

```bash
oc delete rolebinding app-pod-reader
```

Delete the custom Role:

```bash
oc delete role pod-reader
```

Delete the test ServiceAccounts:

```bash
oc delete serviceaccount app-sa
```

```bash
oc delete serviceaccount devops-sa
```

Verify:

```bash
oc get serviceaccounts
```

Your Sandbox's original ServiceAccounts should remain.

---


# 🧠 Key Takeaways

### Project

> Logical workspace and isolation boundary for OpenShift resources.

### User

> Identity representing a human.

### Group

> Collection of users.

### ServiceAccount

> Identity used by applications/workloads.

### Authentication

> **Who are you?**

### Authorization

> **What are you allowed to do?**

### Role

> Defines permissions within a project/namespace.

### RoleBinding

> Assigns a Role to a User, Group or ServiceAccount.

### ClusterRole

> Defines reusable permissions at cluster scope; it can also be used with namespace-scoped RoleBindings.

### ClusterRoleBinding

> Assigns a ClusterRole at cluster scope.

### RBAC

> Controls access to OpenShift/Kubernetes resources based on roles and identities.

### Least Privilege

> Give an identity only the permissions it actually needs.

---

# 🎤 Interview Questions

After completing this lesson, I should be able to answer:

### 1. What is an OpenShift Project?

A logical workspace/isolation boundary for applications and resources.

### 2. Project vs Namespace?

A Project is an OpenShift abstraction around a Kubernetes namespace with additional OpenShift project management and access-control capabilities.

### 3. Authentication vs Authorization?

Authentication identifies the user; authorization determines what the authenticated identity can do.

### 4. User vs ServiceAccount?

A User normally represents a human, while a ServiceAccount provides an identity for applications/workloads.

### 5. What is RBAC?

Role-Based Access Control controls which identities can perform specific actions on resources.

### 6. Role vs RoleBinding?

A Role defines permissions. A RoleBinding assigns those permissions to a User, Group or ServiceAccount.

### 7. Role vs ClusterRole?

A Role is namespace/project scoped. A ClusterRole defines a reusable role at cluster scope and can also be bound within a namespace.

### 8. RoleBinding vs ClusterRoleBinding?

A RoleBinding grants permissions within a namespace/project. A ClusterRoleBinding grants a ClusterRole at cluster scope.

### 9. What are RBAC verbs?

Actions such as `get`, `list`, `watch`, `create`, `update`, `patch` and `delete`.

### 10. Why use ServiceAccounts?

To provide applications with dedicated identities and controlled permissions without using human credentials.

### 11. How do you check whether an identity has permission?

Use:

```bash
oc auth can-i
```

Example:

```bash
oc auth can-i get pods --as=system:serviceaccount:<project>:<serviceaccount>
```

### 12. What is least privilege?

Giving a user, group or application only the minimum permissions required to perform its job.


---

The key lesson from this module is:

> **OpenShift uses Users, Groups and ServiceAccounts as identities, while RBAC uses Roles and RoleBindings to control what those identities can do.**

---

## Useful References

* Red Hat OpenShift Documentation: https://docs.redhat.com/en/documentation/openshift_container_platform/
* OpenShift CLI: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/cli_tools/openshift-cli-oc
* OpenShift RBAC: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/
