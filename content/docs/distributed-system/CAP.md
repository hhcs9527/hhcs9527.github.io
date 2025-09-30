---
title: "CAP theorem"
type: docs
---

## What is CAP?
- **Consistency (C)**: All nodes see the same data at the same time. After a write, all reads return the latest value.
- **Availability (A)**: Every request receives a response (success or failure), without guaranteeing the data is up-to-date.
- **Partition Tolerance (P)**: The system continues operating despite network failures between nodes.

## Real-World Trade-offs
In distributed systems, network partitions are inevitable, making P a requirement rather than a choice.

The real decision is between **CP and AP**: when a network partition occurs, should the system prioritize Consistency (reject requests to maintain data accuracy) or Availability (continue serving requests despite potential inconsistency)?

## Use Case
### [KEP-3157 RV semantic](../kubernetes/KEP-3157.md#resourceversion-semantics)
- **RV="" (CP approach)**: Performs quorum read from etcd to ensure strong consistency, but may block during network partitions.
- **RV="0" (AP approach)**: Serves from apiserver's watch cache for immediate response and high availability, tolerating eventual consistency.