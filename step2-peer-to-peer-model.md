# Step 2: Eliminating the Hub — Peer-to-Peer DR Model

## 1. Prerequisites

This document assumes **Step 1 (OCM/ACM Isolation)** is complete. All OCM/ACM dependencies are behind abstraction interfaces:

| Interface | What It Abstracts |
|-----------|-------------------|
| `RemoteResourceManager` | Deploying resources to remote clusters (ManifestWork) |
| `RemoteResourceReader` | Querying resources on remote clusters (ManagedClusterView) |
| `PlacementManager` | Workload placement decisions (Placement/PlacementDecision) |
| `ClusterRegistry` | Cluster discovery and metadata (ManagedCluster) |
| `SecretDistributor` | Secret propagation to clusters (Policy framework) |
| `ClusterMetadataPublisher` | Advertising cluster metadata (ClusterClaim) |
| `AddonDeployer` | Deploying addons to clusters (ManagedClusterAddOn) |

With these interfaces in place, this step replaces the hub-spoke architecture with a peer-to-peer model by implementing new backends and restructuring the orchestration logic.

---

## 2. Goal

Eliminate the hub/management cluster entirely. The two DR clusters communicate directly with each other. There is no central orchestrator.

### Target Architecture

```
┌─────────────────┐                         ┌─────────────────┐
│   DR Cluster A  │◄───── peer-to-peer ────►│   DR Cluster B  │
│                 │     (S3 state store +    │                 │
│  Ramen Operator │      optional direct     │  Ramen Operator │
│  (unified)      │      API access)        │  (unified)      │
└────────┬────────┘                         └────────┬────────┘
         │                                           │
         └──────────┐                   ┌────────────┘
                    ▼                   ▼
              ┌──────────────────────────────┐
              │     S3 Object Storage        │
              │  (shared state, metadata,    │
              │   coordination, backups)     │
              └──────────────────────────────┘
```

### What Changes

| Aspect | Hub-Spoke (Current) | Peer-to-Peer (Target) |
|--------|--------------------|-----------------------|
| Orchestration | Single DRPC controller on hub | Each cluster runs its own DR controller |
| Resource deployment | Hub pushes ManifestWork to spoke | Each cluster creates resources locally |
| Status querying | Hub queries spoke via MCV | Clusters exchange state via S3 |
| Placement | Hub manages Placement/PlacementDecision | Eliminated; Primary/Secondary role tracked in VRG |
| Secrets | Hub pushes secrets via Policy | Secrets configured locally on each cluster |
| Cluster discovery | Hub reads ManagedCluster | Peer info configured in DRPolicy |
| Operator modes | `dr-hub` + `dr-cluster` (separate) | Single unified operator |

---

## 3. S3 as the Peer Coordination Medium

### 3.1 Why S3

Ramen already uses S3 for VRG metadata and PV/PVC metadata exchange. S3 is the natural coordination medium because:

- **Both clusters already have S3 access** — credentials and connectivity are already configured.
- **Available during partial failures** — if one cluster is down, S3 remains accessible for the surviving cluster to read the last known state and execute failover.
- **Eventually consistent** — acceptable for DR state machines where operations are measured in seconds/minutes, not milliseconds.
- **No new infrastructure** — no additional services to deploy.

### 3.2 Extended S3 Key Schema

The S3 store is extended beyond the existing VRG and PV/PVC metadata to carry coordination state:

| S3 Key Pattern | Content | Written By | Read By | Frequency |
|---------------|---------|------------|---------|-----------|
| `{ns}/{vrg}/v1.VolumeReplicationGroup/{vrg-name}` | VRG metadata (existing) | Primary cluster | Peer cluster | On VRG reconcile |
| `{ns}/{vrg}/v1.PersistentVolume/{pv-name}` | PV metadata (existing) | Primary cluster | Peer cluster | On PV protect |
| `{ns}/{vrg}/v1.PersistentVolumeClaim/{pvc-name}` | PVC metadata (existing) | Primary cluster | Peer cluster | On PVC protect |
| `{ns}/{vrg}/dr-state` | Current DR state (who is Primary, generation, timestamp) | Primary cluster | Peer cluster | On state change |
| `{ns}/{vrg}/dr-intent/{cluster-name}` | DR operation intent (failover, relocate, idle) | Initiating cluster | Peer cluster | On user action |
| `{ns}/{vrg}/cluster-status/{cluster-name}` | Cluster's DR readiness (VRG status summary, replication health) | Each cluster | Peer cluster | Periodic (~30s) |
| `cluster-info/{cluster-name}/storage-classes` | Available storage classes and capabilities | Each cluster | Peer cluster | On change |
| `cluster-info/{cluster-name}/health` | Cluster health/readiness | Each cluster | Peer cluster | Periodic (~60s) |

### 3.3 Coordination Protocol

#### State Machine

Each cluster-VRG pair operates a simple state machine:

```
                  ┌──────────┐
           ┌─────►│  Idle    │◄─────┐
           │      │ (Primary)│      │
           │      └────┬─────┘      │
           │           │            │
    relocate-complete  │ failover   │ relocate-complete
           │           │ or         │
           │           │ relocate   │
           │      ┌────▼─────┐      │
           │      │ Failing  │      │
           │      │  Over    │      │
           │      └────┬─────┘      │
           │           │            │
           │      ┌────▼──────┐     │
           └──────┤  Idle     ├─────┘
                  │(Secondary)│
                  └───────────┘
```

#### Failover (Cluster A down, Cluster B takes over)

```
Time    Cluster A (down)                   S3                          Cluster B (surviving)
─────   ────────────────                   ──                          ───────────────────────
 t0     [unavailable]                                                  Operator detects Cluster A
                                                                       health missing from S3

 t1                                                                    User creates DRAction{Failover}
                                                                       on Cluster B

 t2                                        ◄── dr-intent/cluster-b:   Cluster B writes failover intent
                                                {action: failover}

 t3                                        Cluster B reads             Cluster B reads last VRG state,
                                           dr-state, PV/PVC           PV/PVC metadata from S3
                                           metadata ──►

 t4                                                                    Cluster B promotes local VRG:
                                                                         Secondary → Primary

 t5                                                                    Cluster B restores PVs/PVCs
                                                                       from S3 metadata

 t6                                        ◄── dr-state:              Cluster B writes new dr-state:
                                                {primary: cluster-b,     primary=cluster-b, gen=N+1
                                                 generation: N+1}

 t7                                        ◄── cluster-status/b:      Cluster B signals readiness:
                                                {ready: true}            app can be started

 ...    [Cluster A recovers]

 t8     Cluster A reads                    dr-state ──►                
        dr-state from S3,
        sees cluster-b is Primary
        with generation N+1

 t9     Cluster A demotes its              ◄── cluster-status/a:
        VRG: Primary → Secondary              {role: secondary}
```

#### Relocate (both clusters healthy, planned migration)

```
Time    Cluster A (current Primary)        S3                          Cluster B (current Secondary)
─────   ────────────────────────           ──                          ──────────────────────────────
 t0     User creates DRAction{Relocate,
        target: cluster-b}

 t1     Cluster A ensures final            ◄── VRG metadata,          
        replication sync complete              PV/PVC metadata

 t2     Cluster A writes relocate          ◄── dr-intent/cluster-a:
        intent to S3                            {action: relocate,
                                                 target: cluster-b}

 t3     Cluster A demotes VRG:             ◄── cluster-status/a:
        Primary → Secondary                    {role: secondary-ready}

 t4                                        dr-intent, status ──►      Cluster B reads intent and
                                                                       Cluster A's status from S3

 t5                                                                    Cluster B promotes VRG:
                                                                         Secondary → Primary

 t6                                        ◄── dr-state:              Cluster B writes new dr-state:
                                                {primary: cluster-b,     primary=cluster-b, gen=N+1
                                                 generation: N+1}

 t7                                        ◄── cluster-status/b:      Cluster B signals readiness:
                                                {role: primary-ready}    app can be started

 t8     Cluster A reads new                dr-state, status ──►
        dr-state, confirms
        Cluster B is Primary

 t9     Relocate complete on both clusters
```

### 3.4 Conflict Resolution and Fencing

Without a hub, split-brain is possible: after a network partition heals, both clusters may believe they are Primary.

**Prevention mechanisms:**

1. **Generation counter in S3.** The `dr-state` object contains a monotonically increasing generation number. A cluster can only become Primary by incrementing the generation. If two clusters race, S3's conditional write (ETag/If-Match) ensures only one wins.

2. **S3-based fencing token.** Before promoting to Primary, a cluster writes a fencing token to S3. The peer must read the token and confirm it has stepped down before the new Primary proceeds. This provides a serialization point.

3. **Storage-level fencing.** CSI NetworkFence is used to ensure that even if both clusters attempt to write, the storage system rejects writes from the non-Primary cluster. This is the last line of defense.

4. **Heartbeat-based detection.** Each cluster writes a periodic heartbeat to `cluster-info/{cluster-name}/health`. If a peer's heartbeat is stale beyond a configurable threshold, the surviving cluster can initiate failover.

---

## 4. Operator Unification

### 4.1 Current: Two Operator Modes

Ramen currently builds a single binary that operates in one of two modes, selected at startup:

- `dr-hub`: Runs DRPC, DRPolicy, DRCluster controllers on the hub cluster.
- `dr-cluster`: Runs VRG, DRClusterConfig, ReplicationGroupSource/Dest controllers on managed clusters.

This split is controlled by `RamenConfig.ramenControllerType` and enforced in `cmd/main.go`.

### 4.2 Target: Single Unified Operator

In the peer-to-peer model, every cluster runs the same operator with all controllers:

| Controller | Origin | Status in Unified Operator |
|------------|--------|---------------------------|
| VRG | `dr-cluster` | **Unchanged** |
| DRClusterConfig | `dr-cluster` | **Modified** — publishes metadata to S3 instead of ClusterClaim |
| ReplicationGroupSource/Dest | `dr-cluster` | **Unchanged** |
| DRControl (new) | Replaces `dr-hub` DRPC | **New** — peer-aware DR coordination |
| DRPolicy | `dr-hub` | **Simplified** — validates local config and peer S3 connectivity |
| DRCluster | `dr-hub` | **Simplified** — local fencing and config, no ManifestWork/MCV |

The `RamenConfig.ramenControllerType` field and the associated `dr-hub`/`dr-cluster` split in `cmd/main.go` are **removed**. All controllers always run.

### 4.3 Impact on `cmd/main.go`

- Remove the `ControllerType` switch that selects which controllers to start.
- Remove all OCM scheme registrations (they were already moved to the OCM backend in Step 1; with no OCM backend, they are gone entirely).
- Register the new `DRControl` controller.
- Simplify manager setup — one mode, one set of controllers.

---

## 5. New and Modified CRDs

### 5.1 DRControl (New — Replaces DRPlacementControl)

The `DRPlacementControl` CRD is deeply coupled to the hub-spoke model (it references a Placement and is designed to run on the hub). It is replaced by `DRControl`, which runs on each DR cluster.

```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRControl
metadata:
  name: my-app-dr
  namespace: my-app
spec:
  # Which DRPolicy governs this DR relationship
  drPolicyRef:
    name: dr-policy-east-west

  # The VRG spec for the protected application
  vrgSpec:
    replicationType: async
    pvcSelector:
      matchLabels:
        app: my-app
    s3Profiles:
      - s3-profile-east

  # DR action (set by user to trigger operations)
  action: ""          # "", "Failover", "Relocate"
  targetCluster: ""   # required for Relocate

status:
  phase: Protecting   # Protecting, FailingOver, Relocating, Idle
  currentPrimary: cluster-east
  peerStatus:
    clusterName: cluster-west
    vrgHealthy: true
    lastSeen: "2026-02-09T12:00:00Z"
  conditions:
    - type: PeerConnected
      status: "True"
    - type: VRGHealthy
      status: "True"
    - type: ReadyForApplication
      status: "True"
```

**Key differences from DRPlacementControl:**
- No `placementRef` — there is no Placement to reference.
- `action` is set directly by the user (instead of the hub DRPC controller interpreting it).
- `status.peerStatus` shows the peer cluster's state as read from S3.
- `status.conditions` includes `ReadyForApplication` for external deployment frameworks.

### 5.2 DRPolicy (Modified)

```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRPolicy
metadata:
  name: dr-policy-east-west
spec:
  # Peer clusters (exactly 2)
  drClusters:
    - cluster-east
    - cluster-west

  # Replication configuration
  schedulingInterval: "5m"

  # S3 coordination store (used for peer state exchange)
  s3CoordinationProfile: s3-profile-shared
```

**Key differences:**
- `drClusters` no longer reference ManagedCluster names — they reference `DRCluster` resources on the local cluster.
- `s3CoordinationProfile` explicitly identifies the S3 profile used for peer coordination.

### 5.3 DRCluster (Modified)

```yaml
apiVersion: ramendr.openshift.io/v1alpha1
kind: DRCluster
metadata:
  name: cluster-west
spec:
  # Identity
  clusterName: cluster-west

  # S3 store for metadata and coordination
  s3ProfileName: s3-profile-shared

  # Peer connectivity (optional — for direct API access)
  peerEndpoint:
    apiServer: "https://api.cluster-west.example.com:6443"
    secretRef:
      name: cluster-west-kubeconfig
      namespace: ramen-system

  # Cluster fencing configuration
  cidrs:
    - "10.0.0.0/16"

status:
  phase: Available
  peerStorageClasses:
    - name: ceph-rbd
      provisioner: rbd.csi.ceph.com
  conditions:
    - type: Validated
      status: "True"
    - type: S3Reachable
      status: "True"
```

**Key differences:**
- No reference to a ManagedCluster.
- `peerEndpoint` is optional — allows direct API access to the peer for lower latency, but S3 is the primary coordination mechanism.
- `status.peerStorageClasses` is populated from S3 metadata (read from the peer's published storage class info).

---

## 6. Application Lifecycle

### 6.1 Decoupling Application Deployment from DR

In the current model, Ramen controls application placement via OCM Placement/PlacementDecision. ACM's application manager (or ArgoCD via ApplicationSet) watches the PlacementDecision and deploys the application to the selected cluster.

In the peer-to-peer model, **Ramen does not control application deployment**. It manages DR state (which cluster is Primary, replication health, data readiness) and signals when the target cluster is ready for the application.

### 6.2 Integration Points for Application Frameworks

Ramen exposes readiness through the `DRControl` status:

```yaml
status:
  conditions:
    - type: ReadyForApplication
      status: "True"
      reason: "DataReady"
      message: "PVs/PVCs restored, VRG is Primary, application can start"
```

Application frameworks integrate by watching this condition:

| Framework | Integration Pattern |
|-----------|-------------------|
| **ArgoCD** | An ArgoCD ApplicationSet generator watches `DRControl` status. When `ReadyForApplication` is True on a cluster, ArgoCD deploys the app there. |
| **Flux** | A Flux Kustomization uses a health check gate on the `DRControl` resource. |
| **Manual / Script** | A simple script or CI/CD pipeline polls `DRControl` status and applies manifests. |
| **Webhook** | Ramen optionally calls a webhook URL when `ReadyForApplication` transitions to True. |

### 6.3 Trade-offs

| Aspect | Hub Model (Current) | Peer Model (Target) |
|--------|--------------------|-----------------------|
| End-to-end DR | Ramen handles everything: data + app placement | Ramen handles data; app placement is external |
| Setup complexity | Single ACM integration | Requires configuring app framework integration |
| Flexibility | Tied to ACM/OCM app management | Works with any app deployment framework |
| Failure scope | Hub failure blocks all DR operations | No single point of failure for orchestration |

---

## 7. Controller-Level Redesign

### 7.1 DRControl Controller (New — replaces DRPC)

This is the most significant new component. It runs on **every cluster** and coordinates DR operations with the peer via S3.

**Responsibilities:**
1. **Watch `DRControl` CRs** — react to user-initiated actions (Failover, Relocate).
2. **Manage local VRG** — create/update the local VRG directly (no ManifestWork).
3. **Exchange state with peer via S3** — write local status, read peer status, write/read intent.
4. **Coordinate state transitions** — implement the failover/relocate state machine.
5. **Signal application readiness** — set `ReadyForApplication` condition when data is ready.

**Contrast with DRPC:**

| Aspect | DRPC (Hub) | DRControl (Peer) |
|--------|-----------|-----------------|
| Runs on | Hub only | Every cluster |
| Creates VRG via | ManifestWork to remote cluster | Direct local creation |
| Reads VRG status via | ManagedClusterView | Direct local read + S3 for peer |
| Moves application via | PlacementDecision update | Sets `ReadyForApplication` status |
| Coordinates with peer | N/A (hub sees both sides) | S3 state exchange |

### 7.2 DRPolicy Controller (Simplified)

Currently validates DRPolicy against ManagedCluster resources, creates ManifestWorks for DR cluster setup, and manages secrets via Policy.

**Simplified responsibilities:**
1. Validate that the referenced `DRCluster` resources exist.
2. Validate S3 connectivity to the coordination store.
3. No ManifestWork creation — each cluster configures itself.
4. No secret propagation — secrets are configured locally.

### 7.3 DRCluster Controller (Simplified)

Currently creates ManifestWorks, queries MCVs, deploys VolSync addon, and manages fencing.

**Simplified responsibilities:**
1. Read peer storage class metadata from S3.
2. Apply fencing (NetworkFence) locally when needed.
3. No ManifestWork creation — DRClusterConfig is managed locally by DRClusterConfig controller.
4. No MCV queries — peer state comes from S3.
5. No VolSync addon deployment — VolSync is a prerequisite.

### 7.4 DRClusterConfig Controller (Modified)

Currently discovers storage classes and creates ClusterClaims.

**Modified behavior:**
- Storage class discovery is unchanged (local API reads).
- Instead of creating ClusterClaim resources, publishes storage metadata to S3 using the new `S3ClusterMetadataPublisher` backend (replaces `OCMClusterMetadataPublisher`).

### 7.5 VRG Controller (Unchanged)

The VRG controller runs on the local cluster and manages local VolumeReplication, VolSync, and Velero resources. It does not interact with the hub or peer directly. **No changes needed.**

---

## 8. New Interface Backends

Step 1 defined interfaces. Step 2 provides **peer-to-peer backends** for the interfaces that are still needed, and eliminates the interfaces that no longer apply.

| Interface | Peer-to-Peer Approach |
|-----------|----------------------|
| `RemoteResourceManager` | **Eliminated.** Resources are created locally — the DRControl controller creates VRGs directly on its own cluster. No remote resource deployment is needed. |
| `RemoteResourceReader` | **Replaced by `S3PeerStateReader`.** Reads peer cluster state from S3 instead of querying the remote cluster's API. |
| `PlacementManager` | **Eliminated.** There is no placement framework. The DRControl controller manages Primary/Secondary state directly. |
| `ClusterRegistry` | **Replaced by `LocalClusterRegistry`.** Reads cluster info from local `DRCluster` resources and S3 peer metadata. No ManagedCluster. |
| `SecretDistributor` | **Eliminated.** Secrets are configured locally on each cluster by the administrator. No hub-to-spoke push. |
| `ClusterMetadataPublisher` | **Replaced by `S3ClusterMetadataPublisher`.** Writes storage class info to S3 instead of ClusterClaim. |
| `AddonDeployer` | **Eliminated.** VolSync (and other addons) are prerequisites. |

**Only two new backends are needed:**
1. `S3PeerStateReader` — reads peer status, intent, and metadata from S3.
2. `S3ClusterMetadataPublisher` — writes local metadata to S3.

Plus the new **S3 Peer Coordination Protocol** library used by the DRControl controller.

---

## 9. Risk Analysis

### 9.1 Split-Brain

**Risk:** Without a hub as the single source of truth, both clusters could simultaneously believe they are Primary after a network partition heals.

**Mitigations:**
- **S3 generation counter with conditional writes** — only the cluster that successfully increments the generation can become Primary. S3's ETag-based conditional PutObject provides the serialization.
- **Storage-level fencing** — CSI NetworkFence prevents data corruption even if both clusters temporarily believe they are Primary.
- **Heartbeat staleness threshold** — configurable timeout before a surviving cluster initiates failover, reducing false positives.

### 9.2 S3 Availability

**Risk:** If S3 is unavailable, peer coordination stalls.

**Mitigations:**
- CSI-level replication continues independently of S3.
- Multiple S3 profiles (already supported) provide redundancy.
- Locally cached state allows failover to proceed with last-known data.
- S3 connectivity is monitored and surfaced as a `DRControl` status condition.

### 9.3 Latency of S3-Based Coordination

**Risk:** S3 polling introduces latency compared to the hub model where ManifestWork/MCV updates trigger controller reconciliation immediately via watchers.

**Mitigations:**
- Configurable polling interval (default ~30 seconds).
- For time-sensitive operations, the optional `peerEndpoint` in DRCluster allows direct API-based notification.
- DR operations (failover, relocate) are inherently minutes-long; 30-second polling latency is acceptable.

### 9.4 Application Lifecycle Decoupling

**Risk:** Users who rely on ACM's integrated app-placement-plus-DR experience will need to configure a separate app deployment integration.

**Mitigations:**
- Provide reference integrations for ArgoCD and Flux.
- Provide a simple CLI/webhook tool for manual deployments.
- Document the `ReadyForApplication` condition and how to consume it.

### 9.5 Loss of Centralized Observability

**Risk:** Without a hub, there is no single-pane-of-glass view of DR across both clusters.

**Mitigations:**
- A lightweight CLI tool (e.g., `ramenctl`) that reads S3 state and presents a unified view.
- Each cluster's `DRControl` status shows its own state and the peer's last-known state.
- Prometheus metrics exported from each cluster can be aggregated in a shared monitoring stack.

---

## 10. Estimated Effort

This estimate assumes Step 1 is already complete.

| Phase | Scope | Effort |
|-------|-------|--------|
| **1. Design** | S3 coordination protocol, DRControl CRD, state machine, conflict resolution | 3-4 weeks |
| **2. S3 peer coordination library** | Implement S3 key schema, read/write/poll, conditional writes, generation counters | 3-4 weeks |
| **3. DRControl controller** | New controller with failover/relocate state machine, local VRG management, S3 coordination | 6-8 weeks |
| **4. Unify operator** | Remove `dr-hub`/`dr-cluster` split, merge controllers into single operator | 1-2 weeks |
| **5. Simplify existing controllers** | Update DRPolicy, DRCluster, DRClusterConfig for peer-to-peer operation | 2-3 weeks |
| **6. CRD changes** | DRControl CRD, modify DRPolicy/DRCluster, admission webhooks, validation | 2-3 weeks |
| **7. App framework integrations** | Reference integrations for ArgoCD, Flux; webhook support | 2-3 weeks |
| **8. Testing** | Unit tests, integration tests, e2e tests with peer model, split-brain testing | 4-6 weeks |
| **9. Documentation and migration** | User docs, upgrade path from hub model, architecture docs | 2-3 weeks |
| **Total** | | **25-36 weeks** |

**Combined with Step 1 (11-17 weeks): Total project estimate is 36-53 weeks.**

---

## 11. Code Impact Summary

### Files Deleted (hub-only logic, already behind interfaces from Step 1)

After Step 1, the OCM backend files contain the ManifestWork, MCV, Placement, Policy, ManagedCluster, and ClusterClaim implementations. In Step 2, the entire OCM backend package is deleted:

| Package | Approximate Lines | Reason |
|---------|------------------|--------|
| `internal/controller/multicluster/ocm/` | ~3,500 (moved here in Step 1) | Entire OCM backend — no longer used |

### Files Deleted or Gutted (hub-only controllers)

| File | Lines | Reason |
|------|-------|--------|
| `internal/controller/drplacementcontrol_controller.go` | ~3,000 | Replaced by DRControl controller |
| `internal/controller/drplacementcontrol.go` | ~2,900 | Replaced by DRControl instance logic |
| `internal/controller/drplacementcontrolvolsync.go` | ~500 | Replaced by local VolSync management in DRControl |
| `internal/controller/drplacementcontrol_watcher.go` | ~700 | No longer needed (no ManifestWork/MCV/Placement watchers) |

**Subtotal: ~7,100 lines removed**

### Files Modified

| File | Lines | Changes |
|------|-------|---------|
| `internal/controller/drcluster_controller.go` | ~1,600 | Remove `RemoteResourceManager`/`RemoteResourceReader` usage; simplify to local + S3 |
| `internal/controller/drpolicy_controller.go` | ~600 | Remove `ClusterRegistry`/`RemoteResourceReader` usage; validate local config + S3 |
| `internal/controller/drclusterconfig_controller.go` | ~500 | Switch to `S3ClusterMetadataPublisher` |
| `internal/controller/drcluster_mmode.go` | ~300 | Local fencing only |
| `internal/controller/drpolicy_peerclass.go` | ~200 | Read peer classes from S3 |
| `cmd/main.go` | ~500 | Remove operator mode split; register DRControl controller |

**Subtotal: ~3,700 lines modified**

### New Files

| File / Package | Approximate Lines | Purpose |
|---------------|------------------|---------|
| `internal/controller/drcontrol_controller.go` | ~2,000-3,000 | DRControl controller (main orchestration) |
| `internal/controller/drcontrol.go` | ~1,500-2,000 | DRControl instance logic (state machine) |
| `internal/controller/s3peer/` | ~1,000-1,500 | S3 peer coordination library (read/write/poll) |
| `internal/controller/s3peer/metadata.go` | ~300-500 | S3 cluster metadata publisher |
| `api/v1alpha1/drcontrol_types.go` | ~200-300 | DRControl CRD types |
| Webhook / integration helpers | ~200-400 | App framework readiness signaling |

**Subtotal: ~5,200-7,700 new lines**

### Grand Total

| Category | Lines |
|----------|-------|
| OCM backend deleted | ~3,500 |
| Hub controller code deleted | ~7,100 |
| Existing files modified | ~3,700 |
| New code written | ~5,200-7,700 |
| **Net change** | **~-1,600 to +1,100 lines** (roughly neutral) |

The codebase stays approximately the same size — hub-specific orchestration code is replaced by peer coordination code of similar complexity.

---

## 12. Migration Path

For users migrating from the hub-spoke model to the peer-to-peer model:

1. **Install unified Ramen operator** on both clusters (replacing `dr-hub` on hub and `dr-cluster` on managed clusters).
2. **Create DRCluster resources** on each cluster, referencing the S3 coordination profile and (optionally) the peer's API endpoint.
3. **Create DRPolicy resources** on each cluster, referencing the two DRCluster resources.
4. **Create DRControl resources** on the Primary cluster for each protected application (migrated from DRPlacementControl).
5. **Configure application framework integration** (ArgoCD ApplicationSet, Flux health gate, or webhook).
6. **Decommission the hub cluster** (or repurpose it) — it is no longer needed for DR operations.

A migration tool or script can automate steps 2-4 by reading existing DRPC/DRPolicy/DRCluster resources from the hub and generating the equivalent peer-to-peer resources.
