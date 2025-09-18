---
title: "Custom Resource and Controller"
type: docs
---

## Custom Resource
Everything a user interacts with in Kubernetes is a **resource**, including Pods, DaemonSets, and Deployments.
Similarly, Custom Resources (CRs) are just another type of resource, but instead of being predefined by Kubernetes, they are defined by users through Custom Resource Definitions (CRDs).

Once defined, CRDs behave just like native Kubernetes resources, and controllers can watch them and trigger reconciliation.
Specifically, spec is to descrive the desires state to looks like, status represent the current reconcile state.

**Exception**: If the object does not require reconciliation (e.g., static configuration data), we can omit the spec and status fields.


## Controller
Controllers watch them and trigger **reconciliation** to ensure the actual state matches the desired state (defined in spec), with the current state reflected in status.


### Reconcile
![Reconcile flow](/images/reconcile-process.png)

A loop to ensure the current state match the desired state, controller **doesn't care about event history**, only current state.

### Control Loop in Action
- Deployment level (maintaining 3 replicas)
- Pod level (ensuring resource limits and configurations match specs).
- if delete, controller enforces the desired state by ensuring its complete removal from the Kubernetes environment

```yaml
Indicate the spec section from replicas: 3 .
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app-image:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          env:
            - name: ENVIRONMENT
              value: "production"
          resources:
            limits:
              cpu: "500m"
              memory: "256Mi"
            requests:
              cpu: "250m"
              memory: "128Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
      restartPolicy: Always
```

### Owner Reference
Creates parent-child relationships between resources. Deleting the parent automatically deletes all children. Example: Delete Deployment → ReplicaSets and Pods are removed.

### Finalizer
Define the how a resource is delete, here is the flow.
1. Add Finalizer → Resource marked can't delete yet
2. User deletes → Resource enters terminating state
3. Controller cleans up → Removes any **dependencies defined by user**
4. Remove finalizer → Kubernetes completes deletion