---
title: "Go env: Troubleshoot Missing C Compiler"
type: docs
---

## Problem
When build the project locally, the following errors occured
```bash
$ make build
# github.com/neuvector/neuvector/controller/common
../go/pkg/mod/github.com/neuvector/neuvector@v0.0.0-20250903232727-c2d83d9df18f/controller/common/output.go:55:13: undefined: LevelToPrio
../go/pkg/mod/github.com/neuvector/neuvector@v0.0.0-20250903232727-c2d83d9df18f/controller/common/output.go:92:14: undefined: LevelToPrio
# ... more undefined errors
```
This build works fine in CI and other Linux environments, but fails locally.

## Environment
```bash
- OS: Ubuntu 24.04.1 LTS (x86_64)
- Architecture: x86_64
- RKE2: v1.33.0+rke2r1
- go: go1.24.2
```

## Root Cause Analysis
- Ubuntu version: Same as CI → not the cause
- RKE2 version: Unrelated to local build
- Go version: Same as CI → not the cause
- Go environment: Potential cause

To inspect the Go environment, run:

```
go env -> check all of the var
go env CGO_ENABLED GOPROXY GOSUMDB GOOS GOARCH
```

### Key Difference Found

Upon comparing with the CI environment, this stood out:

| Machine | CGO\_ENABLED | Build Result      |
| ------- | ------------ | ----------------- |
| CI      | `1`          | ✅ Success         |
| Local   | `0`          | ❌ Undefined funcs |

### Fixes

#### Problem 1: `CGO_ENABLED=0`

Some Go modules rely on CGO for system-level functionality. If CGO is disabled, files using `import "C"` are skipped during compilation, leading to undefined symbols.

#### **Solution:**

```bash
export CGO_ENABLED=1
make build
```

#### Problem 2: Missing C Compiler

After enabling CGO, you might hit this error:

```bash
# runtime/cgo
cgo: C compiler "cc" not found: exec: "cc": executable file not found in $PATH
```

#### Solution: Install a C compiler

```bash
sudo apt update
sudo apt install build-essential
```

This provides the required C toolchain (`gcc`, `cc`, `make`, etc.).

## Summary

The root cause of the build failure was a mismatch in the `CGO_ENABLED` setting and a missing C compiler. By aligning the local environment with CI (enabling CGO and installing build tools), the issue was resolved.

## Take away
Using `go env` was the key to identifying the environment difference.
