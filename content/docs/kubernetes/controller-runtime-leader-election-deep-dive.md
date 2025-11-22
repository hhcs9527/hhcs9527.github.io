---
title: "Deep Dive: Leader Election in Controller-Runtime"
type: docs
---

## The Questions You'll Ask

When building Kubernetes controllers with high availability, you'll inevitably hit these questions:

- Does only the leader's reconciler execute its `Start()` method?
- Are all reconcilers' `Reconcile()` methods blocked until leader election completes?
- When leadership switches from controller-1 to controller-2, what happens to controller-1?
- How does the transition happen without causing duplicate work?

This article dives into the mechanism behind the scenes using cert-manager as a concrete example. Want quick answers? Jump to [FAQ](#FAQ).

## Real-world story

Our team wanted to deploy controllers in HA mode with multiple replicas for periodic tasks.

Our leader raised these questions:

- Does every replica get blocked until leader election completes?
- What happens when a leader loses its lease?
- Can we ensure the task will only be executed once?

We couldn't find clear answers in the docs, so we read the source code to understand how
controller-runtime actually handles this. This article documents what we learned.

## Why Leader Election Matters

In production Kubernetes clusters, controllers run with multiple replicas for high availability.
Without coordination, all replicas would attempt to reconcile the same resource simultaneously, causing:

- **Race conditions** - Multiple instances competing to update resources
- **Data inconsistency** - Resource updates applied in unpredictable order
- **Duplicate API calls** - Wasted resources and potential cascading failures

Leader election solves this by ensuring **only one replica (the leader) actively runs reconciliation logic while others remain hot standbys.**

## Manager and Runnable Interface

### Manager

The central orchestrator responsible for:

1. **Registration**: Components register via `mgr.Add()` or builder patterns
2. **Queueing**: Registered runnables enter a `startQueue` awaiting leader election
3. **Execution**: After winning leadership, the Manager triggers `Start()` on all queued components

### Patterns for Controller Registration

#### Pattern 1: Builder-Based Registration

The most common pattern for controllers watching Custom Resources:

```go
func (r *RegistryReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&storagev1alpha1.Resource{}).
        Complete(r)
}
```

**Example**: [Kubewarden's registry controller](https://github.com/kubewarden/sbomscanner/blob/main/internal/controller/registry_controller.go#L92) uses `SetupWithManager` to register the custom controller and ensure the registry exists.

#### Pattern 2: Manual Registration

For specialized components like cert managers that rotate certificates in a time loop, manual registration provides fine-grained control over leader election behavior:

```go
func (r *CertReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return mgr.Add(r) // Direct registration
}

func (r *CertReconciler) Start(ctx context.Context) error {
    // Custom startup logic - runs only on leader
    return r.controller.Start(ctx)
}
```

**Example**: [Kubewarden's certController](https://github.com/kubewarden/kubewarden-controller/blob/57d46e64b05bdbcf1f7849e25af56037d09f273c/internal/controller/cert_controller.go#L60) uses `Start()` for certificate rotation.

### Runnable Interface

The Runnable interface provides a way to execute code periodically without reconciliation:

- Provides an interface to run custom logic independently from reconcilers
- By implementing the `Start()` function, the registry executes it automatically
- **LeaderElection Runnable**: A subtype of Runnable that allows you to specify whether leader election is required, preventing duplicate execution across instances

## FAQ
### Q1 : Does only the leader's reconciler execute its `Start()` method?
**A**: Only after the leader's `Start()` method executes. The flow is:

```
Leader elected
  → Start() begins watching CRs
  → CR events arrive
  → Reconcile() called for each event
```
Non-leader pods never call `Reconcile()` because their `Start()` never executes.

Use Case: Periodically update Cert CR implemented by [Kubewarden CertController](https://github.com/kubewarden/kubewarden-controller/blob/85f71061da955ef55714a26769e06554b0a681bb/internal/controller/cert_controller.go#L34).

### Q2 : Are all reconcilers' `Reconcile()` methods blocked until leader election completes?
**A**: Yes, here is the flow
```
Pod starts
  → Manager.Start() called
  → Leader election begins (blocking)
  → Non-leaders: Wait indefinitely
  → Leader wins
    → Calls Start() on all registered runnables
    → Controller.Start() begins watching
    → Events trigger Reconcile()
```
### Q3 : When leadership switches from controller-1 to controller-2, what happens to controller-1?
**A**: When controller-1 loses leadership, controller-runtime shuts its manager down and stops reconciliation.

### Q4 : How does the transition happen without causing duplicate work?
**A**: Idempotent reconciliation + resourceVersion
 - All writes go through the API server with `resourceVersion` checks.
 - If two leaders ever overlap briefly, one update wins and the other sees a conflict and requeues.
 - Since `Reconcile()` should be idempotent, re-running it converges to the same desired state instead of causing double side effects.

## Takeaway

- With leader election enabled, **only the leader’s manager starts leader-elected runnables**, so only the leader actually runs `Start()` and `Reconcile()`. Other replicas stay as hot standbys.
- Start() is only called on the leader, so you can use it to implement **leader-only custom behavior**. For example, [Kubewarden CertController](https://github.com/kubewarden/kubewarden-controller/blob/85f71061da955ef55714a26769e06554b0a681bb/internal/controller/cert_controller.go#L34). uses Start() to run a periodic certificate-rotation loop.
- **No reconciliation happens before a leader is elected** – controllers are registered with the manager but don’t start processing queues until the lease is acquired.
- The transition to a new leader is safe because of **a single Lease object + API server `resourceVersion` checks + idempotent reconcilers**. Even if a resource is reconciled again, it converges to the same desired state instead of causing double side effects.


## References

- [controller-runtime Manager](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/runnable_group.go#L55)
- [Leader Election Implementation](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go#L408)
- [Kubewarden Certificate Controller](https://github.com/kubewarden/kubewarden-controller/blob/57d46e64b05bdbcf1f7849e25af56037d09f273c/internal/controller/cert_controller.go)
- [Controller-Runtime Internal](https://github.com/kubernetes-sigs/controller-runtime/blob/6ad5c1dd4418489606d19dfb87bf38905b440561/pkg/manager/internal.go#L580)
- [Kubernetes Controller 机制详解](https://www.zhaohuabing.com/post/2023-03-09-how-to-create-a-k8s-controller/)
- [Implementing Leader Election in Kubernetes Controllers](https://medium.com/@vamshitejanizam/implementing-leader-election-in-kubernetes-controllers-1820fd5b8f72)