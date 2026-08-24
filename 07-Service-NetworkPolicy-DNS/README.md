# OpenShift Learning – Lesson 7: Networking Fundamentals

## 🎯 Objectives

In this lesson, I learned and practiced:

- Pod Networking
- Pod IP
- Services
- ClusterIP
- Service Selectors
- Endpoints
- EndpointSlices
- Service DNS
- `port` vs `targetPort`
- OpenShift Routes
- NetworkPolicy
- Service troubleshooting

## 1. Pod Networking

Check Pod IP addresses:

```bash
oc get pods -o wide
```

Pods receive their own IP addresses. Pod IPs can change when Pods are recreated, so applications should normally communicate through a Service instead of directly using Pod IPs.

## 2. Create Web Application

Created `lesson7-web.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lesson7-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lesson7-web
  template:
    metadata:
      labels:
        app: lesson7-web
    spec:
      containers:
        - name: web
          image: nginxinc/nginx-unprivileged:1.27
          ports:
            - containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: lesson7-web
spec:
  type: ClusterIP
  selector:
    app: lesson7-web
  ports:
    - port: 80
      targetPort: 8080
```

Apply:

```bash
oc apply -f lesson7-web.yaml
```

Verify:

```bash
oc get deployment lesson7-web
oc get pods -l app=lesson7-web -o wide
oc get svc lesson7-web
```

## 3. Services and ClusterIP

Check the Service:

```bash
oc get svc lesson7-web
```

Example:

```text
NAME          TYPE        CLUSTER-IP     PORT(S)
lesson7-web   ClusterIP   172.30.x.x     80/TCP
```

A `ClusterIP` Service provides a stable internal endpoint for applications running inside the OpenShift cluster.

```text
Client Pod
    |
    v
ClusterIP Service
    |
    +--------+
    |        |
    v        v
  Pod 1     Pod 2
```

## 4. Service Selector

The Service uses:

```yaml
selector:
  app: lesson7-web
```

The Pods use:

```text
app=lesson7-web
```

The Service uses the selector to find the backend Pods.

Check:

```bash
oc get pods --show-labels
```

## 5. Endpoints

Check the Service endpoints:

```bash
oc get endpoints lesson7-web
```

Example:

```text
NAME          ENDPOINTS
lesson7-web   10.130.x.x:8080,10.131.x.x:8080
```

If the Service shows:

```text
ENDPOINTS   <none>
```

check:

- Service selector
- Pod labels
- Pod status
- `targetPort`

## 6. EndpointSlices

Check EndpointSlices:

```bash
oc get endpointslice -l kubernetes.io/service-name=lesson7-web
```

EndpointSlices contain information about the backend endpoints associated with a Service.

## 7. Create Client Pod

Created `lesson7-client.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson7-client
  labels:
    role: client
spec:
  containers:
    - name: client
      image: curlimages/curl
      command:
        - sleep
        - "3600"
```

Apply:

```bash
oc apply -f lesson7-client.yaml
```

Check:

```bash
oc get pod lesson7-client
```

## 8. Test Service DNS

Enter the client Pod:

```bash
oc exec -it lesson7-client -- sh
```

Test the Service:

```bash
curl http://lesson7-web
```

The NGINX response was successfully returned.

Test the fully qualified Service DNS name:

```bash
curl http://lesson7-web.lakshminarayananredh-dev.svc.cluster.local
```

Service DNS format:

```text
<service>.<namespace>.svc.cluster.local
```

## 9. Internal vs External Access

Inside the OpenShift cluster:

```text
Client Pod
    |
    v
Service DNS
    |
    v
ClusterIP Service
    |
    v
Web Pods
```

The following works from inside a Pod:

```bash
curl http://lesson7-web
```

The following does not work from my Windows machine:

```bash
curl http://lesson7-web
```

Reason:

`lesson7-web` is a cluster-internal DNS name.

My Windows machine is outside the OpenShift cluster. The `oc` CLI connects my machine to the remote OpenShift Developer Sandbox; it does not make my Windows machine part of the cluster.

## 10. `port` vs `targetPort`

Service configuration:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Traffic flow:

```text
Client
  |
  | Service Port 80
  v
Service
  |
  | targetPort 8080
  v
Container Port 8080
```

- `port` = Service port
- `targetPort` = Application/container port

## 11. Service Troubleshooting

Deliberately changed the Service selector:

```bash
oc patch service lesson7-web --type=merge -p '{"spec":{"selector":{"app":"wrong-app"}}}'
```

Check the endpoints:

```bash
oc get endpoints lesson7-web
```

Result:

```text
ENDPOINTS   <none>
```

The Service could not find the Pods because the selector no longer matched the Pod labels.

Restore the correct selector:

```bash
oc patch service lesson7-web --type=merge -p '{"spec":{"selector":{"app":"lesson7-web"}}}'
```

Verify:

```bash
oc get endpoints lesson7-web
```

The Pod endpoints appeared again.

## 12. OpenShift Route

Expose the Service:

```bash
oc expose service lesson7-web
```

Check the Route:

```bash
oc get route lesson7-web
```

Get the Route hostname:

```bash
oc get route lesson7-web -o jsonpath="{.spec.host}"
```

External traffic flow:

```text
Windows PC
    |
    v
OpenShift Route
    |
    v
Service
    |
    v
Web Pods
```

The OpenShift Route provides external HTTP/HTTPS access to the application.

## 13. NetworkPolicy

Created `lesson7-networkpolicy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: lesson7-web-policy
spec:
  podSelector:
    matchLabels:
      app: lesson7-web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: client
      ports:
        - protocol: TCP
          port: 8080
```

Apply:

```bash
oc apply -f lesson7-networkpolicy.yaml
```

Check:

```bash
oc describe networkpolicy lesson7-web-policy
```

The policy targets:

```text
app=lesson7-web
```

and allows ingress from:

```text
role=client
```

on:

```text
TCP 8080
```

## 14. NetworkPolicy Testing

Created another client:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lesson7-bad-client
  labels:
    role: bad-client
```

Check the labels:

```bash
oc get pods --show-labels
```

Expected:

```text
lesson7-client       role=client
lesson7-bad-client   role=bad-client
lesson7-web          app=lesson7-web
```

Test the allowed client:

```bash
oc exec lesson7-client -- curl --max-time 5 http://lesson7-web
```

The allowed client successfully accessed the application.

Test the bad client:

```bash
oc exec lesson7-bad-client -- curl --max-time 5 http://lesson7-web
```

In the OpenShift Developer Sandbox, the bad client was also able to access the application even though the NetworkPolicy configuration was correct.

The Sandbox user does not have permission to inspect the cluster-level network configuration, so the underlying network implementation was not investigated further.

## 15. Useful Troubleshooting Commands

```bash
oc get pods -o wide
oc get pods --show-labels
oc get svc
oc describe svc lesson7-web
oc get endpoints lesson7-web
oc get endpointslice
oc get route
oc get networkpolicy
oc describe networkpolicy lesson7-web-policy
```

## 16. Key Learnings

- Pod IPs can change.
- Services provide stable endpoints.
- ClusterIP provides internal cluster communication.
- Service selectors use Pod labels.
- Endpoints show the backend Pods selected by a Service.
- EndpointSlices provide modern endpoint information.
- Service DNS works from inside the OpenShift cluster.
- `port` and `targetPort` can be different.
- OpenShift Routes provide external application access.
- NetworkPolicy controls Pod network traffic.
- `<none>` endpoints usually require checking Service selectors and Pod labels.
- My Windows machine is outside the OpenShift cluster.

## 17. Interview Questions

1. What is a Pod IP?
2. Why do we need a Service?
3. What is ClusterIP?
4. How does a Service find Pods?
5. What are Endpoints?
6. What is an EndpointSlice?
7. What is Service DNS?
8. Why does Service DNS work inside a Pod but not from my Windows machine?
9. What is the difference between `port` and `targetPort`?
10. What happens when a Service has no endpoints?
11. How does an OpenShift Route expose an application?
12. What is NetworkPolicy?
13. How would you troubleshoot a Service connectivity issue?
14. How would you troubleshoot a NetworkPolicy issue?

## 18. Cleanup

```bash
oc delete route lesson7-web
oc delete networkpolicy lesson7-web-policy
oc delete -f lesson7-web.yaml
oc delete -f lesson7-client.yaml
oc delete pod lesson7-bad-client
```

## ✅ Lesson 7 Completed

- [x] Pod Networking
- [x] Pod IP
- [x] Services
- [x] ClusterIP
- [x] Service Selectors
- [x] Endpoints
- [x] EndpointSlices
- [x] Service DNS
- [x] Internal Service Communication
- [x] `port` vs `targetPort`
- [x] Service Troubleshooting
- [x] OpenShift Route
- [x] External Application Access
- [x] NetworkPolicy
- [x] NetworkPolicy Troubleshooting

## 🏁 Final Architecture

```text
                    OpenShift Cluster
                 +---------------------+
                 |                     |
Client Pod ----->| Service ----> Pods  |
                 |                     |
                 +----------^----------+
                            |
                      OpenShift Route
                            ^
                            |
                       Windows PC
```
