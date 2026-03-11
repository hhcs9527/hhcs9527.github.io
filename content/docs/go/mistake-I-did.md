---
title: "Go Mistakes I did"
type: docs
---
## Declaration Scope

Declare variables outside the block when you need them beyond the immediate `if` statement.

```go
// ❌ Bad: job is scoped to the if block
if job, err := h.jobClient.Create(job); err != nil {
    return err
}
// job is inaccessible here

// ✅ Good: declare first when you need job afterward
job, err := h.jobClient.Create(job)
if err != nil {
    return err
}
// job is accessible here
```

The rule of thumb: **if you only care about the error, use the short form. If you need the returned value, declare outside.**