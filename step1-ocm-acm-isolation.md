# Step 1: Isolating RamenDR from OCM/ACM Dependencies

## 1. Goal

Introduce **abstraction interfaces** between Ramen's controller logic and the OCM/ACM APIs it depends on. After this step:

- Ramen's hub-spoke architecture is **unchanged**. A hub cluster still orchestrates DR across managed clusters.
- All OCM/ACM types are confined behind well-defined Go interfaces. No controller logic directly imports OCM packages.
- An **OCM backend** implements each interface, preserving 100% backward compatibility with existing ACM deployments.
- Alternative backends (direct Kubernetes API, agent-based, etc.) can be implemented against the same interfaces in the future.

This is a **refactoring step**, not a behavioral change. No new features, no new CRDs, no changes to user-facing APIs. The existing hub-spoke model continues to work exactly as it does today.

---

## 2. Current State

### Go Module Dependencies on OCM

| Package | Version | Purpose |
|---------|---------|---------|
| `open-cluster-management.io/api` | v0.15.0 | ManagedCluster, Placement, PlacementDecision, ManifestWork, ClusterClaim |
| `open-cluster-management.io/config-policy-controller` | v0.15.0 | ConfigurationPolicy API for secret propagation |
| `open-cluster-management.io/governance-policy-propagator` | v0.16.0 | Policy and PlacementBinding APIs for secret propagation |
| `open-cluster-management.io/multicloud-operators-subscription` | v0.15.0 | ManagedClusterView API for cross-cluster resource querying |
| `github.com/stolostron/multicloud-operators-placementrule` | v1.2.4-... | PlacementRule API (legacy placement) |

### Where OCM Lives in the Codebase

| Operator Mode | Runs On | OCM Dependency Level |
|---------------|---------|---------------------|
| `dr-hub` | Hub cluster | **Heavy** — ManifestWork, MCV, Placement, Policy, ManagedCluster |
| `dr-cluster` | Managed clusters | **Light** — ClusterClaim only |

Nearly all OCM coupling is in the **hub operator**. The DR cluster operator is already close to independent.

### Files with Direct OCM Imports (production code)

| File | OCM APIs Used | Lines |
|------|--------------|-------|
| `internal/controller/util/mw_util.go` | ManifestWork | ~720 |
| `internal/controller/util/mcv_util.go` | ManagedClusterView | ~620 |
| `internal/controller/util/managedcluster.go` | ManagedCluster | ~118 |
| `internal/controller/util/secrets_util.go` | Policy, PlacementBinding, ConfigurationPolicy, PlacementRule | ~775 |
| `internal/controller/drplacementcontrol_controller.go` | Placement, PlacementDecision, PlacementRule | ~3,000 |
| `internal/controller/drplacementcontrol.go` | Placement, PlacementDecision | ~2,900 |
| `internal/controller/drplacementcontrol_watcher.go` | ManifestWork, MCV, Placement, PlacementRule | ~700 |
| `internal/controller/drplacementcontrolvolsync.go` | ManifestWork | ~500 |
| `internal/controller/drpolicy_controller.go` | ManagedCluster, ManagedClusterView | ~600 |
| `internal/controller/drpolicy_peerclass.go` | ManagedClusterView | ~200 |
| `internal/controller/drcluster_controller.go` | ManifestWork, MCV, ManagedCluster | ~1,600 |
| `internal/controller/drcluster_mmode.go` | ManifestWork, ManagedClusterView | ~300 |
| `internal/controller/drclusterconfig_controller.go` | ClusterClaim | ~500 |
| `internal/controller/volsync/secret_propagator.go` | Policy, PlacementBinding, ConfigurationPolicy, PlacementRule | ~346 |
| `internal/controller/volsync/deploy_volsync.go` | ManagedClusterAddOn | ~84 |
| `cmd/main.go` | All OCM scheme registrations | ~500 |

---

## 3. Dependency Catalog and Abstraction Plan

### 3.1 ManifestWork — Resource Distribution to Remote Clusters

**OCM API:** `work.open-cluster-management.io/v1` — `ManifestWork`

**What it does:** ManifestWork is the mechanism the hub uses to deploy resources to managed clusters. The hub creates a ManifestWork in the managed cluster's namespace, and the OCM work agent on that cluster applies the embedded manifests.

**How Ramen uses it:**
- Deploy VRG resources to managed clusters
- Create application namespaces on managed clusters
- Deploy NetworkFence resources for fencing operations
- Deploy MaintenanceMode resources for storage maintenance
- Deploy DRClusterConfig resources to managed clusters
- Distribute RBAC ClusterRoles granting the work agent permissions
- Read ManifestWork status conditions (`WorkApplied`, `WorkAvailable`, `WorkDegraded`) to determine deployment success

**Key files:**
- `internal/controller/util/mw_util.go` — `MWUtil` struct with all ManifestWork CRUD operations (~720 lines)
- `internal/controller/drplacementcontrol_controller.go` — Creates VRG ManifestWorks
- `internal/controller/drcluster_controller.go` — Creates config ManifestWorks
- `internal/controller/drplacementcontrol_watcher.go` — Watches ManifestWork status changes

**Pervasiveness:** Very high. Used across 15+ files with a dedicated ~720-line utility file.

**Proposed interface:**

```go
// RemoteResourceManager handles deploying and tracking resources on remote clusters.
type RemoteResourceManager interface {
    // CreateOrUpdateRemoteResource deploys a set of Kubernetes objects to a
    // remote cluster. Returns the operation result (created/updated/none).
    CreateOrUpdateRemoteResource(ctx context.Context, name, remoteCluster string,
        objects []client.Object, labels, annotations map[string]string,
    ) (OperationResult, error)

    // CreateOrUpdateRemoteResourceWithDeleteOrphan is the same as
    // CreateOrUpdateRemoteResource but orphans resources on the remote
    // cluster when the remote resource wrapper is deleted.
    CreateOrUpdateRemoteResourceWithDeleteOrphan(ctx context.Context, name, remoteCluster string,
        objects []client.Object, labels, annotations map[string]string,
    ) (OperationResult, error)

    // FindRemoteResource retrieves a previously-created remote resource
    // deployment by name and cluster.
    FindRemoteResource(ctx context.Context, name, remoteCluster string,
    ) (*RemoteResourceStatus, error)

    // IsRemoteResourceApplied returns true if the remote resource has been
    // successfully applied on the remote cluster.
    IsRemoteResourceApplied(name, remoteCluster string) bool

    // DeleteRemoteResource removes a remote resource deployment.
    DeleteRemoteResource(ctx context.Context, name, remoteCluster string) error

    // ListRemoteResources lists remote resource deployments matching labels
    // in a given cluster namespace.
    ListRemoteResources(ctx context.Context, remoteCluster string,
        matchLabels map[string]string,
    ) ([]RemoteResourceStatus, error)

    // ExtractObject extracts a typed Kubernetes object from a remote resource
    // deployment by GVK.
    ExtractObject(status *RemoteResourceStatus, obj client.Object,
        gvk schema.GroupVersionKind,
    ) error
}

// RemoteResourceStatus represents the state of a remote resource deployment.
type RemoteResourceStatus struct {
    Name        string
    Namespace   string
    Applied     bool
    Available   bool
    Degraded    bool
    Manifests   []runtime.RawExtension
    Labels      map[string]string
    Annotations map[string]string
}
```

**OCM backend:** Wraps the existing `MWUtil` code. `CreateOrUpdateRemoteResource` creates a ManifestWork, `FindRemoteResource` reads ManifestWork status, etc.

**Isolation difficulty: HIGH** — Large surface area. The `MWUtil` struct has many methods and is called from many files. Each call site must be updated to use the interface.

---

### 3.2 ManagedClusterView — Cross-Cluster Resource Querying

**OCM API:** `view.open-cluster-management.io/v1beta1` — `ManagedClusterView`

**What it does:** ManagedClusterView allows the hub to query individual resources on managed clusters. The hub creates an MCV specifying what to look up, and the OCM controller reports the result back via the MCV status.

**How Ramen uses it:**
- Query VRG status from managed clusters
- Query storage classes, volume snapshot classes, volume replication classes, etc.
- Query DRClusterConfig status
- Query MaintenanceMode and NetworkFence resources
- Check namespace existence on managed clusters

**Key files:**
- `internal/controller/util/mcv_util.go` — `ManagedClusterViewGetter` interface with 20+ methods (~620 lines)
- Used by `drplacementcontrol_controller.go`, `drcluster_controller.go`, `drpolicy_controller.go`

**Pervasiveness:** Very high. 20+ methods in the interface, consumed by all hub controllers.

**Proposed interface:**

Ramen **already has** an abstraction for this: the `ManagedClusterViewGetter` interface in `mcv_util.go`. However, it has two problems:
1. It returns OCM-specific types (`*viewv1beta1.ManagedClusterViewList`) in some methods.
2. Its method signatures expose OCM concepts (annotations used as MCV metadata).

The interface should be generalized:

```go
// RemoteResourceReader queries resources on remote clusters.
type RemoteResourceReader interface {
    // GetResourceFromCluster fetches a single resource from a remote cluster.
    // The result is unmarshaled into the provided object.
    GetResourceFromCluster(ctx context.Context, resourceName, resourceNamespace,
        clusterName string, obj client.Object,
    ) error

    // DeleteResourceView removes any cached view/query for the named resource.
    DeleteResourceView(ctx context.Context, clusterName, viewName string) error
}
```

Higher-level typed methods (e.g., `GetVRGFromManagedCluster`, `GetSClassFromManagedCluster`) remain as **convenience functions** that call the generic interface, but they live in Ramen code, not in the interface contract. This keeps the interface small while preserving ergonomics.

**OCM backend:** Wraps the existing MCV logic. `GetResourceFromCluster` creates/reads an MCV. `DeleteResourceView` deletes the MCV.

**Isolation difficulty: HIGH** — The existing interface is already a good starting point, but the 20+ typed methods must be refactored to delegate to the generic interface, and all return types must be stripped of OCM-specific types.

---

### 3.3 Placement and PlacementDecision — Workload Scheduling

**OCM API:** `cluster.open-cluster-management.io/v1beta1` — `Placement`, `PlacementDecision`

**What it does:** Placement selects which managed clusters host a workload. PlacementDecision contains the result. The DRPC controller reads PlacementDecision to determine the current cluster and creates/updates PlacementDecisions during failover/relocate.

**How Ramen uses it:**
- DRPC references a Placement (or PlacementRule) via `placementRef`
- Reads PlacementDecision to find which cluster is selected
- Annotates Placement with `experimental-scheduling-disable` to prevent OCM rescheduling
- Creates/updates/deletes PlacementDecision during DR operations
- Watches Placement and PlacementDecision for changes

**Key files:**
- `internal/controller/drplacementcontrol_controller.go` — RBAC, reconciliation, setup (~3,000 lines)
- `internal/controller/drplacementcontrol.go` — Placement logic (~2,900 lines)
- `internal/controller/drplacementcontrol_watcher.go` — Watchers (~700 lines)

**Pervasiveness:** High. Deeply integrated into the DRPC controller.

**Proposed interface:**

```go
// PlacementManager manages workload placement across clusters.
type PlacementManager interface {
    // GetPlacementRef returns the referenced placement object for a given DRPC.
    GetPlacementRef(ctx context.Context, namespace, name, kind string,
    ) (client.Object, error)

    // GetClusterDecision reads the current cluster decision from the placement.
    GetClusterDecision(ctx context.Context, placement client.Object,
    ) ([]ClusterDecision, error)

    // UpdateClusterDecision sets the cluster decision for a placement,
    // directing the workload to the specified cluster(s).
    UpdateClusterDecision(ctx context.Context, placement client.Object,
        decisions []ClusterDecision,
    ) error

    // DisableScheduling prevents the placement framework from automatically
    // rescheduling the workload.
    DisableScheduling(ctx context.Context, placement client.Object) error

    // SetupWatches registers watchers for placement-related resources that
    // should trigger DRPC reconciliation.
    SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error
}

// ClusterDecision represents a placement decision for a single cluster.
type ClusterDecision struct {
    ClusterName string
    Reason      string
}
```

**OCM backend:** Wraps the existing Placement/PlacementDecision logic.

**Isolation difficulty: HIGH** — Placement is deeply woven into the DRPC controller's reconcile loop. Extracting it behind an interface requires careful refactoring of the 3,000-line DRPC controller.

---

### 3.4 PlacementRule — Legacy Placement

**OCM API:** `apps.open-cluster-management.io/v1` — `PlacementRule`

**What it does:** PlacementRule is the legacy cluster selection mechanism. Used for backward compatibility in DRPC and as the targeting mechanism for policy-based secret propagation.

**Key files:**
- `internal/controller/drplacementcontrol_controller.go` — DRPC compatibility
- `internal/controller/util/secrets_util.go` — Secret propagation targeting
- `internal/controller/volsync/secret_propagator.go` — VolSync secret targeting
- `internal/controller/drplacementcontrol_watcher.go` — Watcher

**Proposed interface:** Subsumed by `PlacementManager` (section 3.3) and `SecretDistributor` (section 3.6). PlacementRule support in the `PlacementManager` OCM backend handles backward compatibility. PlacementRule usage in secret propagation is subsumed by the `SecretDistributor` interface.

**Isolation difficulty: MEDIUM** — Coupled to both Placement and Policy. Isolating Placement and Policy automatically isolates PlacementRule.

---

### 3.5 ManagedCluster — Cluster Registration and Discovery

**OCM API:** `cluster.open-cluster-management.io/v1` — `ManagedCluster`

**What it does:** Represents a cluster registered with the hub. Provides join status, cluster ID, and ClusterClaim metadata (storage classes, etc.).

**How Ramen uses it:**
- Validate that clusters referenced in DRPolicy/DRCluster exist and are joined
- Read cluster ID from the `id.k8s.io` ClusterClaim
- Read storage class, snapshot class, and replication class claims
- Watched by DRCluster and DRPolicy controllers

**Key files:**
- `internal/controller/util/managedcluster.go` — `ManagedClusterInstance` wrapper (~118 lines)
- `internal/controller/drpolicy_controller.go`, `drcluster_controller.go` — Consumers

**Proposed interface:**

```go
// ClusterRegistry provides information about known clusters.
type ClusterRegistry interface {
    // GetClusterInfo returns metadata for a named cluster.
    // Returns an error if the cluster is not found or not ready.
    GetClusterInfo(ctx context.Context, clusterName string) (*ClusterInfo, error)

    // SetupWatches registers watchers for cluster registry resources that
    // should trigger reconciliation.
    SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error
}

// ClusterInfo holds metadata about a cluster.
type ClusterInfo struct {
    Name                      string
    ID                        string
    Ready                     bool
    StorageClassNames         []string
    VolumeSnapshotClassNames  []string
    VolumeReplicationClassNames []string
}
```

**OCM backend:** Wraps the existing `ManagedClusterInstance` code. `GetClusterInfo` reads the `ManagedCluster` resource and extracts claims.

**Isolation difficulty: LOW** — The existing wrapper is thin (~118 lines) and the interface is straightforward. This is one of the easiest dependencies to isolate.

---

### 3.6 Policy Framework — Secret Propagation

**OCM APIs:**
- `policy.open-cluster-management.io/v1` — `Policy`, `PlacementBinding`
- `open-cluster-management.io/config-policy-controller/api/v1` — `ConfigurationPolicy`

**What it does:** Propagates S3 credentials and VolSync PSK secrets from the hub to managed clusters using a Policy → PlacementBinding → PlacementRule chain with hub template functions.

**How Ramen uses it:**
- S3 secret propagation: creates Policy/PlacementRule/PlacementBinding to deliver S3 credentials to clusters
- VolSync PSK secret propagation: same pattern for VolSync pre-shared keys
- Uses hub template functions (`{{hub fromSecret ...}}`) for secret value injection
- Uses `policy.open-cluster-management.io/trigger-update` annotation to force reprocessing

**Key files:**
- `internal/controller/util/secrets_util.go` — `SecretsUtil` (~775 lines)
- `internal/controller/volsync/secret_propagator.go` — VolSync secrets (~346 lines)

**Proposed interface:**

```go
// SecretDistributor manages distributing secrets to remote clusters.
type SecretDistributor interface {
    // AddSecretToCluster ensures the named secret (in sourceNamespace on the
    // hub) is delivered to the specified cluster in the target namespace, in
    // the requested format.
    AddSecretToCluster(ctx context.Context, secretName, clusterName,
        sourceNamespace, targetNamespace string, format SecretFormat,
        veleroNamespace string,
    ) error

    // RemoveSecretFromCluster removes the secret from the specified cluster.
    // If this was the last cluster, associated propagation resources are
    // cleaned up.
    RemoveSecretFromCluster(ctx context.Context, secretName, clusterName,
        sourceNamespace string, format SecretFormat,
    ) error
}

// SecretFormat defines the target secret format on the cluster.
type SecretFormat string

const (
    SecretFormatRamen  SecretFormat = "ramen"
    SecretFormatVelero SecretFormat = "velero"
)
```

**OCM backend:** Wraps the existing `SecretsUtil` code. `AddSecretToCluster` creates the Policy/PlacementRule/PlacementBinding chain.

A separate `VolSyncSecretPropagator` interface (or extension of the same) handles VolSync PSK secrets:

```go
// VolSyncSecretPropagator manages VolSync PSK secret distribution.
type VolSyncSecretPropagator interface {
    PropagateSecretToClusters(ctx context.Context, sourceSecret *corev1.Secret,
        owner metav1.Object, destClusters []string,
        destSecretName, destSecretNamespace string,
    ) error

    CleanupSecretPropagation(ctx context.Context, ownerName, ownerNamespace,
        secretNamespace string,
    ) error
}
```

**OCM backend:** Wraps the existing `secret_propagator.go` code.

**Isolation difficulty: MEDIUM** — The code is well-contained in two files. The interface is clean. The main complexity is handling the hub template functions (`fromSecret`, `lookup`) which are OCM-specific.

---

### 3.7 ClusterClaim — Cluster Metadata Advertisement

**OCM API:** `cluster.open-cluster-management.io/v1alpha1` — `ClusterClaim`

**What it does:** The DRClusterConfig controller creates ClusterClaim resources on managed clusters to advertise storage capabilities to the hub via ManagedCluster status.

**Key files:**
- `internal/controller/drclusterconfig_controller.go` — Creates/deletes ClusterClaims (~relevant portion: ~100 lines)

**Proposed interface:**

```go
// ClusterMetadataPublisher advertises local cluster metadata to the
// management plane (hub or peer).
type ClusterMetadataPublisher interface {
    // PublishStorageClasses advertises storage class names.
    PublishStorageClasses(ctx context.Context, classNames []string) error

    // PublishVolumeSnapshotClasses advertises snapshot class names.
    PublishVolumeSnapshotClasses(ctx context.Context, classNames []string) error

    // PublishVolumeReplicationClasses advertises replication class names.
    PublishVolumeReplicationClasses(ctx context.Context, classNames []string) error

    // (additional methods for VolumeGroupSnapshotClass, VolumeGroupReplicationClass,
    //  NetworkFenceClass as needed)

    // CleanupAll removes all published metadata.
    CleanupAll(ctx context.Context) error
}
```

**OCM backend:** Wraps ClusterClaim CRUD operations.

**Isolation difficulty: LOW** — Small, well-scoped. The DRClusterConfig controller already has the discovery logic; only the publishing mechanism needs to be abstracted.

---

### 3.8 ManagedClusterAddOn — Addon Deployment

**OCM API:** `addon.open-cluster-management.io/v1alpha1` — `ManagedClusterAddOn`

**What it does:** Ramen creates a ManagedClusterAddOn on the hub to trigger ACM to deploy VolSync to a managed cluster.

**Key files:**
- `internal/controller/volsync/deploy_volsync.go` (~84 lines)

**Proposed interface:**

```go
// AddonDeployer manages deploying add-on operators to remote clusters.
type AddonDeployer interface {
    // DeployAddon ensures the named addon is deployed to the given cluster.
    DeployAddon(ctx context.Context, addonName, clusterName string) error
}
```

**OCM backend:** Wraps the existing `ManagedClusterAddOn` creation.

**Isolation difficulty: VERY LOW** — Single 84-line file. Trivial to abstract.

---

### 3.9 ManagedClusterSetBinding — Cluster Set Binding

**Current role:** Test/development YAML only. Not used in production Go code.

**Action:** No interface needed. Remove from test/dev YAML when OCM is no longer the target environment, or leave as an OCM-specific test fixture.

**Isolation difficulty: VERY LOW**

---

### 3.10 Subscription and Channel — E2E Testing

**Current role:** Used in e2e test deployers and minor annotation references in production code.

**Action:** No interface needed. The subscription-based e2e deployer is already one of several deployers. Annotation references in production code are removed when ManifestWork code is refactored behind the `RemoteResourceManager` interface.

**Isolation difficulty: LOW**

---

## 4. Summary Matrix

| # | Dependency | Proposed Interface | Isolation Difficulty |
|---|-----------|-------------------|---------------------|
| 3.1 | ManifestWork | `RemoteResourceManager` | **High** |
| 3.2 | ManagedClusterView | `RemoteResourceReader` | **High** |
| 3.3 | Placement/PlacementDecision | `PlacementManager` | **High** |
| 3.4 | PlacementRule | Subsumed by `PlacementManager` + `SecretDistributor` | **Medium** |
| 3.5 | ManagedCluster | `ClusterRegistry` | **Low** |
| 3.6 | Policy Framework | `SecretDistributor` + `VolSyncSecretPropagator` | **Medium** |
| 3.7 | ClusterClaim | `ClusterMetadataPublisher` | **Low** |
| 3.8 | ManagedClusterAddOn | `AddonDeployer` | **Very Low** |
| 3.9 | ManagedClusterSetBinding | N/A (test/dev only) | **Very Low** |
| 3.10 | Subscription/Channel | N/A (e2e test only) | **Low** |

---

## 5. Implementation Plan

### Phase 1: Define Interfaces (1-2 weeks)

Create a new package (e.g., `internal/controller/multicluster/`) containing the interface definitions:
- `RemoteResourceManager`
- `RemoteResourceReader`
- `PlacementManager`
- `ClusterRegistry`
- `SecretDistributor` / `VolSyncSecretPropagator`
- `ClusterMetadataPublisher`
- `AddonDeployer`
- Shared types (`ClusterDecision`, `ClusterInfo`, `RemoteResourceStatus`, etc.)

No behavioral changes. Just the interface `.go` files.

### Phase 2: Implement OCM Backends (3-4 weeks)

Create OCM implementations in a subpackage (e.g., `internal/controller/multicluster/ocm/`):
- `OCMRemoteResourceManager` — wraps `MWUtil`
- `OCMRemoteResourceReader` — wraps `ManagedClusterViewGetterImpl`
- `OCMPlacementManager` — wraps Placement/PlacementDecision logic
- `OCMClusterRegistry` — wraps `ManagedClusterInstance`
- `OCMSecretDistributor` — wraps `SecretsUtil`
- `OCMVolSyncSecretPropagator` — wraps `secret_propagator.go`
- `OCMClusterMetadataPublisher` — wraps ClusterClaim CRUD
- `OCMAddonDeployer` — wraps ManagedClusterAddOn creation

Each backend wraps the existing code with minimal changes. The existing utility files (`mw_util.go`, `mcv_util.go`, etc.) move into the OCM backend package.

### Phase 3: Refactor Controllers to Use Interfaces (4-6 weeks)

Update all controllers to accept interfaces instead of concrete OCM types:

| Controller | Interfaces Consumed |
|------------|-------------------|
| `DRPlacementControlReconciler` | `RemoteResourceManager`, `RemoteResourceReader`, `PlacementManager` |
| `DRPolicyReconciler` | `ClusterRegistry`, `RemoteResourceReader`, `SecretDistributor` |
| `DRClusterReconciler` | `RemoteResourceManager`, `RemoteResourceReader`, `ClusterRegistry`, `AddonDeployer` |
| `DRClusterConfigReconciler` | `ClusterMetadataPublisher` |

This is the largest phase. Each controller must be updated to:
1. Accept interfaces via constructor/struct fields (dependency injection).
2. Replace direct OCM type references with interface method calls.
3. Move OCM-specific watcher setup into the backend's `SetupWatches` method.

### Phase 4: Wire Up and Validate (2-3 weeks)

- Update `cmd/main.go` to instantiate OCM backends and inject them into controllers.
- Move OCM scheme registration into the OCM backend package.
- Update all unit tests to use mock implementations of the interfaces.
- Run full integration and e2e test suites to confirm no regressions.

### Phase 5: Remove Direct OCM Imports from Controllers (1-2 weeks)

Final cleanup:
- Verify that no controller file directly imports any `open-cluster-management.io` package.
- The only files importing OCM packages are in `internal/controller/multicluster/ocm/`.
- Update `go.mod` to ensure OCM dependencies are only required by the OCM backend.
- Update RBAC markers to be generated by the backend package.

---

## 6. Estimated Effort

| Phase | Scope | Effort |
|-------|-------|--------|
| 1. Define interfaces | 7 interfaces + shared types | 1-2 weeks |
| 2. OCM backends | Wrap existing code in OCM implementations | 3-4 weeks |
| 3. Refactor controllers | Update 4 controllers + watchers to use interfaces | 4-6 weeks |
| 4. Wire up and validate | DI wiring, test updates, regression testing | 2-3 weeks |
| 5. Final cleanup | Remove direct imports, update go.mod | 1-2 weeks |
| **Total** | | **11-17 weeks** |

---

## 7. Risks and Mitigations

### 7.1 Interface Surface Area

The `RemoteResourceManager` and `RemoteResourceReader` interfaces must cover every operation the controllers need. If the interfaces are too narrow, controllers will need workarounds. If too wide, alternative backends become hard to implement.

**Mitigation:** Start with the minimum interface that satisfies existing call sites. Use the OCM backend as the reference implementation to validate completeness.

### 7.2 Watcher Registration

OCM-specific watchers (ManifestWork status changes, MCV updates, Placement changes) are currently set up in the controller's `SetupWithManager` method. These must be delegated to the backend.

**Mitigation:** Each interface that requires watchers includes a `SetupWatches` method that the backend implements. The controller calls `SetupWatches` during setup.

### 7.3 RBAC Markers

Kubebuilder RBAC markers (e.g., `+kubebuilder:rbac:groups=work.open-cluster-management.io`) are currently in controller files. These are OCM-specific.

**Mitigation:** Move OCM RBAC markers to the OCM backend package. Generate backend-specific RBAC as part of the build.

### 7.4 Test Mocking

Existing tests use real OCM types in test fixtures and scheme setup. After isolation, tests should mock the interfaces instead.

**Mitigation:** Phase 4 includes updating all tests. The interface-based design makes mocking straightforward.

---

## 8. Success Criteria

After completing this step:

1. **No controller file imports OCM packages.** Only files under `internal/controller/multicluster/ocm/` import OCM.
2. **All existing tests pass** with the OCM backend.
3. **A mock backend can be written** for unit testing without requiring any OCM types.
4. **The operator behaves identically** to the pre-refactor version — no user-visible changes.
5. **The codebase is ready for Step 2** (peer-to-peer model) or any other alternative multi-cluster backend.
