<!-- markdownlint-configure-file
     { "MD013": { "tables": false, "code_blocks": false } } -->
# 001: Node Pool Creation Cancellation

- **Author:** edwinchiu
- **Date Created:** 2026-10-05
- **Issue Reference:** b/471094575, b/479537067

## Summary

This plan introduces opt-in support for GKE Node Pool Creation Cancellation via
`ENABLE_NODEPOOL_CREATION_CANCELLATION` (off by default) in `tpu-provisioner`.
When enabled, the creation controller initiates node pool creation
asynchronously, monitors in-flight `CREATE_NODE_POOL` operations across
reconciliation loops, and cancels in-flight operations via the GKE
`operations.cancel` API if:

1. The waiting `JobSet` or leader `Pod` goes away (is deleted, completed, or
   failed) while the node pool is still `PROVISIONING`.
2. The `CREATE_NODE_POOL` operation remains in `RUNNING` state longer than
   `NODEPOOL_CREATION_TIMEOUT` (default `30m`), or
   `SPOT_NODEPOOL_CREATION_TIMEOUT` (defaults to `NODEPOOL_CREATION_TIMEOUT` if
   unset) for Spot node pools.

Both triggers are enforced from two places: the `CreationReconciler` (driven by
`Pod`/`JobSet` watch events and periodic polling) and, as a safety net for state
the controller never observed (for example a `JobSet` deleted while the
controller was down), the periodic `NodePoolGarbageCollector`. Cancellations and
failed creation operations are exported as Prometheus counters.

## Motivation

Previously, `tpu-provisioner` created node pools by calling `NodePools.Create`
and blocking the `CreationReconciler` worker goroutine inside `waitForGkeOp` for
up to 30 minutes:

1. **Wasted TPU Capacity & Blocked Cluster Operations:** If a `JobSet` is
   deleted, fails, or completes while its TPU node pool is still `PROVISIONING`
   (for example, waiting for TPU stockout/capacity), `tpu-provisioner` could not
   abort the creation. Calling `NodePools.Delete` while `CREATE_NODE_POOL` is
   `RUNNING` fails with `HTTP 400: Cluster is running incompatible operation
   <opID>`. The node pool would either stay stuck waiting on capacity or finish
   provisioning unused TPU nodes before `DeletionReconciler` could tear it down.
2. **Worker Starvation:** Blocking a `CreationReconciler` worker goroutine
   inside `waitForGkeOp` for up to 30 minutes ties up one of `CONCURRENCY`
   (default `3`) workers per provisioning node pool and prevents
   `controller-runtime` from processing new events for that `Pod` key until the
   blocking call returns.

## Goals

- Support asynchronous GKE `CREATE_NODE_POOL` initiation and non-blocking
  reconciliation polling when `ENABLE_NODEPOOL_CREATION_CANCELLATION=true`.
- Automatically cancel in-flight `CREATE_NODE_POOL` operations when the waiting
  `JobSet` is deleted, completed, or failed, or when the waiting leader `Pod` is
  deleted.
- Automatically cancel in-flight `CREATE_NODE_POOL` operations that exceed
  `NODEPOOL_CREATION_TIMEOUT` (or `SPOT_NODEPOOL_CREATION_TIMEOUT` for Spot TPU
  node pools).
- Properly handle the GKE `ABORTING` rollback window—during which GKE holds a
  lock on the node pool name and automatically tears down partially created GCE
  resources and the node pool itself—by waiting for the operation to reach
  `status: DONE` without issuing conflicting `Create` or `Delete` calls.
- Recover in-flight `CREATE_NODE_POOL` operation state across controller
  restarts via `Operations.List` and garbage-collect orphaned `PROVISIONING`
  node pools in `NodePoolGarbageCollector`.
- Keep the feature **off by default**
  (`ENABLE_NODEPOOL_CREATION_CANCELLATION=false`) to ensure backward
  compatibility and safe rollout.

## Proposal

### Assumptions

1. At the GKE REST / Go client API level
   (`google.golang.org/api/container/v1beta1`),
   `Projects.Locations.Clusters.NodePools.Create` is inherently asynchronous: it
   immediately returns a `*containerv1beta1.Operation` with `"operationType":
   "CREATE_NODE_POOL"` and `"status": "PENDING"` (then `"RUNNING"`), and the
   operation is immediately listed in `Operations.List` while the `NodePool` is
   in `"status": "PROVISIONING"`.
2. GKE `CREATE_NODE_POOL` cancellation (`Operations.Cancel`) needs to be enabled
   on the project; if it is not, `Operations.Cancel` returns `HTTP 400
   INVALID_ARGUMENT` and the controller must handle it gracefully without
   infinite retry loops. Cancelling an operation that already finished returns
   `HTTP 404`. Neither is a retriable error.

### Configuration Flags

Four environment variables are added to `cmd/config/config.go` and logged at
startup in `cmd/main.go`:

| Environment Variable | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `ENABLE_NODEPOOL_CREATION_CANCELLATION` | `bool` | `false` | Enables asynchronous node pool creation tracking and `CREATE_NODE_POOL` operation cancellation. |
| `NODEPOOL_CREATION_TIMEOUT` | `time.Duration` | `30m` | Maximum duration an On-Demand or Reserved `CREATE_NODE_POOL` operation may remain `RUNNING` before being cancelled. |
| `SPOT_NODEPOOL_CREATION_TIMEOUT` | `time.Duration` | `NODEPOOL_CREATION_TIMEOUT` (`30m`) | Maximum duration a Spot (`cloud.google.com/gke-spot: "true"`) `CREATE_NODE_POOL` operation may remain `RUNNING` before being cancelled. Defaults to `NODEPOOL_CREATION_TIMEOUT` if unset (`0`). |
| `NODEPOOL_OPERATION_POLL_INTERVAL` | `time.Duration` | `15s` | Requeue interval used by `CreationReconciler` while polling an in-flight `CREATE_NODE_POOL` operation across Phase 1 (`RUNNING`) and Phase 2 (`ABORTING`) until Phase 3 (`DONE`) is reached. When the feature is enabled it is also used as the poll interval for the remaining synchronous operation waits (`waitForGkeOp`, e.g. node pool deletion and static node pools). On transient API errors, `CreationReconciler` falls back to per-item exponential backoff (`BACKOFF_BASE_DELAY=5s` up to `BACKOFF_MAX_DELAY=5m`). |

Additional configuration behavior:

- `SpotNodepoolCreationTimeout` is defaulted to `NodepoolCreationTimeout` in
  `config.ParseEnv` when unset or non-positive; `GKE.creationTimeout(spot)`
  applies the same fallback defensively.
- `config.Config.Warnings()` returns human-readable warnings for values that
  parse but are likely unintended, and `cmd/main.go` logs them at startup with
  the repository's `WARNING:` prefix. Warnings are only produced when
  `ENABLE_NODEPOOL_CREATION_CANCELLATION=true`: a non-positive
  `NODEPOOL_CREATION_TIMEOUT` (timeouts disabled, stuck creations are never
  cancelled) and a non-positive `NODEPOOL_OPERATION_POLL_INTERVAL` (the built-in
  default poll interval is used).
- `cmd/main.go` only passes `OperationPollInterval` and `ReportOperationErrors`
  to `GKENodePoolService` when the feature is enabled, so synchronous waits are
  unchanged when the flag is off.
- A new kustomize overlay, `config/default-next`, layers a `ConfigMap` patch on
  top of `config/default` that enables the feature with the four variables set
  to their documented defaults (`true` / `30m` / `30m` / `15s`). Deploy it with
  `kubectl apply -k config/default-next`.
- `config/manager/collector-config.yaml` forwards the new
  `tpu_provisioner_nodepool_creation_.*` metrics through the OpenTelemetry
  collector filter.

## Design

### 1. GKE Operation & NodePool State Machine

```text
+-----------------------+
|  NodePools.Create()   | <----------------------------------------------------+
+-----------+-----------+                                                      |
            |                                                                  |
            v                                                                  |
+---------------------------------------------------------------+              |
| Phase 1: Provisioning (polled every PollInterval = 15s)       |              |
| - Operation.status = "PENDING" -> "RUNNING"                   |              |
| - NodePool.status  = "PROVISIONING"                           |              |
| - DeleteNodePool() -> HTTP 400 (incompatible operation)       |              |
+-----------+---------------------------------------+-----------+              |
            |                                       |                          |
            | Workload active &                     | JobSet/Pod gone OR       |
            | capacity acquired                     | elapsed >= TIMEOUT       |
            v                                       v                          |
+-------------------------------+       +---------------------------------------+
| NodePool Ready                |       | Operations.Cancel(opName)             |
| - Operation.status = "DONE"   |       | - RUNNING/PENDING -> HTTP 200         |
| - NodePool.status  = "RUNNING"|       | - ABORTING        -> HTTP 200 (no-op) |
+-------------------------------+       | - DONE            -> HTTP 404         |
                                        | - not allowlisted -> HTTP 400         |
                                        +-------------------+-------------------+
                                                            |                  |
                                                            v                  |
                                        +---------------------------------------+
                                        | Phase 2: Cancelling / Rollback        |
                                        | (polled every PollInterval = 15s)     |
                                        | - Operation.status = "ABORTING"       |
                                        |   (immediately after Cancel)          |
                                        | - NodePool.status  = PROVISIONING     |
                                        |   -> STOPPING (~12s) -> 404 (~65s),   |
                                        |   ERROR possible between steps        |
                                        | - Create/Delete -> HTTP 400 (locked)  |
                                        +-------------------+-------------------+
                                                            |                  |
                                                            | Rollback         |
                                                            | completes        |
                                                            | (~75s observed)  |
                                                            v                  |
                                        +---------------------------------------+
                                        | Phase 3: Cancelled & Deleted          |
                                        | - Operation.status = "DONE"           |
                                        | - Operation.error.code = 1 (CANCELLED)|
                                        | - NodePool.Get() -> HTTP 404 NotFound |
                                        | - NodePool lock released; name        |
                                        |   immediately reusable                |
                                        +-------------------+-------------------+
                                                            |                  |
                                                            | JobSet active &  |
                                                            | Pod unschedulable*
                                                            +------------------+
```

> \* **How `NODEPOOL_OPERATION_POLL_INTERVAL` Drives Phase 2 $\rightarrow$ Phase
> 3 Transitions:**
> Because GKE does not push watch events to Kubernetes when an operation
> transitions from `RUNNING` $\rightarrow$ `ABORTING` $\rightarrow$ `DONE`,
> `CreationReconciler` requeues the pod every `NODEPOOL_OPERATION_POLL_INTERVAL`
> (`15s`) to poll the operation state:
>
> 1. **During Phase 2 (`ABORTING`):** Calling `Operations.Cancel(opName)` does
>    not immediately delete the node pool or release GKE's lock on the node pool
>    name. The operation flips to `ABORTING` within seconds, but the GKE worker
>    only notices the cancellation on its next lease renewal (about `15s`),
>    marks the partially provisioned instances as errored and then runs the
>    regular node pool deletion pipeline, so the node pool reports
>    `PROVISIONING` $\rightarrow$ `STOPPING` (observed ~`12s` after the cancel)
>    $\rightarrow$ `HTTP 404` (observed ~`65s`), with `ERROR` possible in
>    between. The operation stays `ABORTING` for a few more seconds after the
>    node pool is gone. Throughout, GKE rejects any `CreateNodePool` or
>    `DeleteNodePool` call with `HTTP 400: Cluster is running incompatible
>    operation`. `checkExistingNodePoolState` therefore maps `STOPPING` /
>    `ERROR` node pools whose create operation is `ABORTING` (and a `404` node
>    pool with an `ABORTING` operation) to `ErrNodePoolCreationAborting`, and
>    `CreationReconciler` continues requeuing every
>    `NODEPOOL_OPERATION_POLL_INTERVAL` (`15s`).
> 2. **Reaching Phase 3 (`Cancelled & Deleted`):** Once GKE finishes rolling
>    back the resources, the operation reaches `Operation.status = "DONE"`
>    (`error.code = 1 CANCELLED`), GKE automatically deletes the node pool
>    (`NodePool.Get() -> HTTP 404 NotFound`), and the node pool lock is
>    released:
>    - **If cancelled due to timeout (`NODEPOOL_CREATION_TIMEOUT` /
>      `SPOT_NODEPOOL_CREATION_TIMEOUT`) and the `JobSet` is still active:** On
>      the next `15s` poll after Phase 3 is reached, `EnsureNodePoolForPod`
>      observes `State == NotExists` (`404 NotFound`) and immediately starts a
>      fresh `CREATE_NODE_POOL` operation.
>    - **If cancelled because the `JobSet` or `Pod` went away:** On the next
>      `15s` poll after Phase 3 is reached, `CancelNodePoolCreation` observes
>      that no active operation remains (`DONE` / `nil`), removes the pod from
>      `inFlightPods`, and stops polling.

### 2. Example GKE API Payloads

#### In-Flight `CREATE_NODE_POOL` Operation (`RUNNING`)

```json
{
  "name": "operation-1791232512065-c45d1d68-e2c5-4a69-b750-29465625b854",
  "operationType": "CREATE_NODE_POOL",
  "selfLink": "https://container.googleapis.com/v1/projects/123456789012/zones/us-central1-c/operations/operation-1791232512065-c45d1d68-e2c5-4a69-b750-29465625b854",
  "startTime": "2026-10-05T20:35:12.065448749Z",
  "status": "RUNNING",
  "targetLink": "https://container.googleapis.com/v1/projects/123456789012/zones/us-central1-c/clusters/example-cluster/nodePools/example-pool",
  "zone": "us-central1-c"
}
```

#### Cancelling `CREATE_NODE_POOL` Operation (`ABORTING`)

```json
{
  "name": "operation-1791232512065-c45d1d68-e2c5-4a69-b750-29465625b854",
  "operationType": "CREATE_NODE_POOL",
  "selfLink": "https://container.googleapis.com/v1/projects/123456789012/zones/us-central1-c/operations/operation-1791232512065-c45d1d68-e2c5-4a69-b750-29465625b854",
  "startTime": "2026-10-05T20:35:12.065448749Z",
  "status": "ABORTING",
  "targetLink": "https://container.googleapis.com/v1/projects/123456789012/zones/us-central1-c/clusters/example-cluster/nodePools/example-pool",
  "zone": "us-central1-c"
}
```

#### Completed Cancelled `CREATE_NODE_POOL` Operation (`DONE`)

```json
{
  "endTime": "2026-10-05T20:36:45.192837465Z",
  "error": {
    "code": 1,
    "message": "Operation was aborted: node pool is deleted."
  },
  "name": "operation-1791232512065-c45d1d68-e2c5-4a69-b750-29465625b854",
  "operationType": "CREATE_NODE_POOL",
  "selfLink": "https://container.googleapis.com/v1/projects/123456789012/zones/us-central1-c/operations/operation-1791232512065-c45d1d68-e2c5-4a69-b750-29465625b854",
  "startTime": "2026-10-05T20:35:12.065448749Z",
  "status": "DONE",
  "statusMessage": "Failed to create node pool: operation was aborted: node pool is deleted.",
  "targetLink": "https://container.googleapis.com/v1/projects/123456789012/zones/us-central1-c/clusters/example-cluster/nodePools/example-pool",
  "zone": "us-central1-c"
}
```

