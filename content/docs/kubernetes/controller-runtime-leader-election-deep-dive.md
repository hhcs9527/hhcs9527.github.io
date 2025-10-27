---
title: "Deep Dive: Leader Election in Controller-Runtime"
type: docs
---

## The Questions You'll Ask

When building Kubernetes controllers with high availability, you'll inevitably hit these questions:

- Does only the leader's reconciler execute its `Start()` method?
- Are all reconcilers' `Reconcile()` methods blocked until leader election completes?
- Can a leader lose its role without the pod restarting?
- When leadership switches from controller-1 to controller-2, what happens to controller-1?
- How does the transition happen without causing duplicate work?

This article dives into the mechanism behind the scenes using cert-manager as a concrete example. Want quick answers? Jump to [Takeaway](#takeaway).

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

## Core Architecture: Manager and Runnable Interface
### Manager

### Runnable Interface

## How It Works: The Complete Flow

[展开细节时使用]


## Takeaway

## References

- [controller-runtime Manager](https://github.com/kubernetes-sigs/controller-runtime/blob/main/pkg/manager/runnable_group.go#L55)
- [Leader Election Implementation](https://github.com/kubernetes/client-go/blob/master/tools/leaderelection/leaderelection.go#L408)
- [Kubewarden Certificate Controller](https://github.com/kubewarden/kubewarden-controller/blob/57d46e64b05bdbcf1f7849e25af56037d09f273c/internal/controller/cert_controller.go)
- [Controller-Runtime Internal](https://github.com/kubernetes-sigs/controller-runtime/blob/6ad5c1dd4418489606d19dfb87bf38905b440561/pkg/manager/internal.go#L580)
- [Kubernetes Controller 机制详解](https://www.zhaohuabing.com/post/2023-03-09-how-to-create-a-k8s-controller/)
- [Implementing Leader Election in Kubernetes Controllers](https://medium.com/@vamshitejanizam/implementing-leader-election-in-kubernetes-controllers-1820fd5b8f72)