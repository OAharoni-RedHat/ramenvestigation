# Blueprint: Complete Removal of OCM/ACM Dependencies from RamenDR

## Executive Summary

This blueprint details the complete removal of Open Cluster Management (OCM) and Advanced Cluster Management (ACM) as dependencies from Ramen. After this work, Ramen will be a **pure Kubernetes-native** disaster recovery orchestrator that runs on any conformant Kubernetes distribution — including clusters managed by OCM/ACM, but without requiring them.

### Design Decisions

| Decision Point | Choice | Rationale |
|---------------|--------|-----------|
| Multi-cluster communication | **Direct API access** | Hub holds kubeconfigs and uses `client-go` to manage remote clusters directly |
| Placement / scheduling | **Pluggable interface (Option C)** | Native backend reads DRPC fields; interface allows future integration with external schedulers |
| Secret distribution | **Direct API access** | Hub creates secrets directly on managed clusters via kubeconfig |
| Backward compatibility | **Complete OCM removal** | All OCM code, types, and `go.mod` dependencies are deleted. Ramen becomes pure K8s. It still runs on OCM clusters since it only needs kubeconfigs. |
| Cluster registration | **New `ClusterConnection` CRD** | Replaces `ManagedCluster`. Holds kubeconfig secret ref, cluster identity, health, and capabilities. |
| VolSync deployment | **Ramen deploys via direct API** | Hub deploys VolSync manifests to managed clusters using the same direct API mechanism as VRG |

---

## 1. Architecture: Before and After

### Before (OCM-dependent)

```
┌─────────────────────────────────────────────────────────┐
│                     Hub Cluster                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │   DRPC   │  │ DRPolicy │  │DRCluster │              │
│  │Controller│  │Controller│  │Controller│              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│       ▼              ▼              ▼                    │
│  ┌──────────────────────────────────────┐               │
│  │         OCM APIs (ManifestWork,      │               │
│  │    MCV, Placement, Policy, etc.)     │               │
│  └──────────────────┬───────────────────┘               │
└─────────────────────┼───────────────────────────────────┘
                      │ OCM Work Agent / View Agent
          ┌───────────┼───────────┐
          ▼                       ▼
   ┌─────────────┐         ┌─────────────┐
   │ Managed      │         │ Managed      │
   │ Cluster A    │         │ Cluster B    │
   │ (VRG, etc.)  │         │ (VRG, etc.)  │
   └─────────────┘         └─────────────┘
```

### After (Pure Kubernetes)

```
┌─────────────────────────────────────────────────────────┐
│                     Hub Cluster                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │   DRPC   │  │ DRPolicy │  │DRCluster │              │
│  │Controller│  │Controller│  │Controller│              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │              │              │                    │
│       ▼              ▼              ▼                    │
│  ┌──────────────────────────────────────┐               │
│  │    Multi-Cluster Client Manager      │               │
│  │  (kubeconfig-based, direct API)      │               │
│  └──────┬───────────────────┬───────────┘               │
│         │                   │                            │
│  ┌──────┴──────┐     ┌──────┴──────┐                    │
│  │ClusterConn A│     │ClusterConn B│                    │
│  │(kubeconfig) │     │(kubeconfig) │                    │
│  └──────┬──────┘     └──────┬──────┘                    │
└─────────┼───────────────────┼───────────────────────────┘
          │ Direct API        │ Direct API
          │ (client-go)       │ (client-go)
          ▼                   ▼
   ┌─────────────┐     ┌─────────────┐
   │ Cluster A    │     │ Cluster B    │
   │ (VRG, etc.)  │     │ (VRG, etc.)  │
   └─────────────┘     └─────────────┘
```

**Key difference:** The hub communicates with managed clusters **directly** using standard `client-go` with kubeconfigs stored as Kubernetes secrets. No OCM agents, no ManifestWork, no ManagedClusterView. Just standard Kubernetes API calls.

---

## 2. New CRD: `ClusterConnection`

Replaces OCM's `ManagedCluster` and `ClusterClaim` system. This is the single source of truth for "what remote clusters does Ramen know about."

### Spec

```go
// ClusterConnectionSpec defines connection and identity information for a remote cluster.
type ClusterConnectionSpec struct {
    // KubeconfigSecretRef references a Secret containing a kubeconfig for the remote cluster.
    // The Secret must contain a key "kubeconfig" with the full kubeconfig YAML.
    // +kubebuilder:validation:Required
    KubeconfigSecretRef corev1.SecretReference `json:"kubeconfigSecretRef"`

    // ClusterID is a unique, stable identifier for this cluster.
    // Typically the UID of the kube-system namespace on the remote cluster.
    // If omitted, the hub will auto-discover it from the remote cluster.
    // +optional
    ClusterID string `json:"clusterID,omitempty"`
}
```

### Status

```go
// ClusterConnectionStatus reports observed state of the remote cluster connection.
type ClusterConnectionStatus struct {
    // Phase indicates the overall connection state.
    // +kubebuilder:validation:Enum=Connected;Disconnected;Error
    Phase ClusterConnectionPhase `json:"phase,omitempty"`

    // ClusterID is the discovered cluster identity (kube-system namespace UID).
    ClusterID string `json:"clusterID,omitempty"`

    // LastProbeTime is when the hub last successfully contacted this cluster.
    LastProbeTime *metav1.Time `json:"lastProbeTime,omitempty"`

    // Capabilities reports discovered storage capabilities on the remote cluster.
    Capabilities ClusterCapabilities `json:"capabilities,omitempty"`

    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// ClusterCapabilities holds discovered storage-related information from the remote cluster.
type ClusterCapabilities struct {
    StorageClasses                []string `json:"storageClasses,omitempty"`
    VolumeSnapshotClasses         []string `json:"volumeSnapshotClasses,omitempty"`
    VolumeReplicationClasses      []string `json:"volumeReplicationClasses,omitempty"`
    VolumeGroupSnapshotClasses    []string `json:"volumeGroupSnapshotClasses,omitempty"`
    VolumeGroupReplicationClasses []string `json:"volumeGroupReplicationClasses,omitempty"`
    NetworkFenceClasses           []string `json:"networkFenceClasses,omitempty"`
}
```

### Relationship to Existing CRDs

```
ClusterConnection (new)          DRCluster (existing, modified)
┌──────────────────────┐        ┌──────────────────────────┐
│ name: "cluster-a"    │◄───────│ name: "cluster-a"        │
│ kubeconfigSecretRef  │        │ clusterConnectionRef:    │
│ clusterID            │        │   name: "cluster-a"      │
│ status:              │        │ cidrs: [...]             │
│   phase: Connected   │        │ clusterFence: Unfenced   │
│   capabilities: ...  │        │ region: us-east-1        │
│   lastProbeTime: ... │        │ s3ProfileName: s3-east   │
└──────────────────────┘        └──────────────────────────┘
```

- `DRCluster.Name` matches `ClusterConnection.Name` by convention (or a new `clusterConnectionRef` field is added).
- `ClusterConnection` owns connectivity and capability discovery.
- `DRCluster` owns DR-specific state (fencing, CIDRs, region, S3).

---

## 3. Changes to Existing CRDs

### 3.1 DRCluster — Add `ClusterConnectionRef`

```go
type DRClusterSpec struct {
    // ClusterConnectionRef names the ClusterConnection that provides
    // connectivity to this cluster. Defaults to the DRCluster's own name.
    // +optional
    ClusterConnectionRef string `json:"clusterConnectionRef,omitempty"`

    // ... existing fields unchanged ...
    CIDRs         []string          `json:"cidrs,omitempty"`
    ClusterFence  ClusterFenceState `json:"clusterFence,omitempty"`
    Region        Region            `json:"region,omitempty"`
    S3ProfileName string            `json:"s3ProfileName"`
}
```

### 3.2 DRPlacementControl — Make `PlacementRef` Optional

Today `PlacementRef` is required and immutable. In the new model, native placement reads directly from DRPC fields:

```go
type DRPlacementControlSpec struct {
    // PlacementRef is an optional reference to an external placement object.
    // When set, the PlacementManager backend uses this to determine cluster decisions.
    // When omitted, Ramen uses PreferredCluster and FailoverCluster directly.
    // +optional
    PlacementRef *v1.ObjectReference `json:"placementRef,omitempty"`

    // ... existing fields unchanged ...
    PreferredCluster string     `json:"preferredCluster,omitempty"`
    FailoverCluster  string     `json:"failoverCluster,omitempty"`
    Action           DRAction   `json:"action,omitempty"`
    // ...
}
```

**Note:** Making `PlacementRef` optional (from required) is a non-breaking CRD schema change. Existing DRPCs with `PlacementRef` set continue to work. New DRPCs can omit it for native placement.

### 3.3 ProgressionStatus — Remove OCM Terminology

Rename status values that reference OCM concepts:

| Current Value | New Value | Reason |
|--------------|-----------|--------|
| `CreatingMW` | `DeployingResources` | "MW" is ManifestWork |
| `UpdatingPlRule` | `UpdatingPlacement` | "PlRule" is PlacementRule |

### 3.4 RamenConfig — Remove OCM-specific Configuration

```go
type RamenConfig struct {
    // ... keep existing fields ...

    // Remove: DrClusterOperator.ChannelName, PackageName, CatalogSourceName,
    //         CatalogSourceNamespaceName, ClusterServiceVersionName
    //         (these are OLM/ACM subscription concepts)
    //
    // Replace with:
    DrClusterOperator struct {
        DeploymentAutomationEnabled bool   `json:"deploymentAutomationEnabled,omitempty"`
        S3SecretDistributionEnabled bool   `json:"s3SecretDistributionEnabled,omitempty"`
        NamespaceName               string `json:"namespaceName,omitempty"`
    } `json:"drClusterOperator,omitempty"`
}
```

### 3.5 DRClusterConfig — Remove OCM References

Update the doc comments to remove references to "OCM hub cluster":

```go
// DRClusterConfigSpec defines the desired state of DRClusterConfig.
// It carries information regarding the cluster identity as known at the hub cluster.
// It is also used to advertise required replication schedules on the cluster.
type DRClusterConfigSpec struct {
    ReplicationSchedules []string `json:"replicationSchedules,omitempty"`
    // ClusterID carries the cluster identity from the ClusterConnection resource.
    ClusterID string `json:"clusterID,omitempty"`
}
```

---

## 4. Interface Definitions

All interfaces live in a new package: **`internal/controller/multicluster/`**

### 4.1 `MultiClusterClientManager` — The Foundation

This is the central piece that replaces all OCM communication. It manages `client-go` clients for each remote cluster.

```go
package multicluster

import (
    "context"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

// MultiClusterClientManager provides Kubernetes clients for remote clusters.
// It caches clients and watches for kubeconfig secret changes to refresh them.
type MultiClusterClientManager interface {
    // GetClient returns a controller-runtime client for the named cluster.
    // The client is backed by the kubeconfig from the cluster's ClusterConnection.
    GetClient(ctx context.Context, clusterName string) (client.Client, error)

    // GetAPIReader returns an uncached reader for the named cluster.
    // Use for one-shot reads that must be strongly consistent.
    GetAPIReader(ctx context.Context, clusterName string) (client.Reader, error)

    // IsConnected returns true if the named cluster is currently reachable.
    IsConnected(clusterName string) bool
}
```

**Implementation notes:**
- Maintains a map of `clusterName → client.Client`
- Watches `ClusterConnection` resources and their referenced kubeconfig secrets
- Refreshes clients when kubeconfig secrets change
- Periodically probes clusters and updates `ClusterConnection.Status`
- Uses `controller-runtime`'s `cluster.Cluster` or raw `rest.Config` from kubeconfig

### 4.2 `RemoteResourceManager` — Deploy Resources to Remote Clusters

Replaces ManifestWork. The hub creates/updates/deletes resources directly on remote clusters.

```go
// RemoteResourceManager handles deploying and managing resources on remote clusters.
type RemoteResourceManager interface {
    // Apply creates or updates a set of Kubernetes objects on a remote cluster.
    Apply(ctx context.Context, clusterName string, objects []client.Object,
        opts ...ApplyOption,
    ) error

    // Delete removes resources from a remote cluster.
    Delete(ctx context.Context, clusterName string, objects []client.Object) error

    // Get retrieves a single resource from a remote cluster into the provided object.
    Get(ctx context.Context, clusterName string, key client.ObjectKey,
        obj client.Object,
    ) error

    // List retrieves a list of resources from a remote cluster.
    List(ctx context.Context, clusterName string, list client.ObjectList,
        opts ...client.ListOption,
    ) error

    // IsApplied returns true if the named resource exists and matches the desired
    // state on the remote cluster.
    IsApplied(ctx context.Context, clusterName string, obj client.Object) (bool, error)
}

// ApplyOption configures the behavior of an Apply operation.
type ApplyOption func(*applyOptions)

// WithOrphanOnDelete causes applied resources to be left in place (orphaned)
// if the apply tracking object is later deleted.
func WithOrphanOnDelete() ApplyOption { ... }

// WithLabels applies the given labels to applied resources.
func WithLabels(labels map[string]string) ApplyOption { ... }

// WithAnnotations applies the given annotations to applied resources.
func WithAnnotations(annotations map[string]string) ApplyOption { ... }
```

**Implementation:** Uses `MultiClusterClientManager.GetClient()` to get a client for the target cluster, then performs standard CRUD operations. Server-side apply (`client.Apply`) is preferred for idempotent creates/updates.

### 4.3 `RemoteResourceWatcher` — Watch Resources on Remote Clusters

Replaces ManagedClusterView's status-polling model with real Kubernetes watches.

```go
// RemoteResourceWatcher watches resources on remote clusters and triggers
// reconciliation when they change.
type RemoteResourceWatcher interface {
    // Watch starts watching the given resource type on a remote cluster.
    // When changes are detected, the handler is called to enqueue reconcile requests.
    Watch(ctx context.Context, clusterName string, obj client.Object,
        handler handler.EventHandler, predicates ...predicate.Predicate,
    ) error

    // StopWatch stops watching the given resource type on a remote cluster.
    StopWatch(ctx context.Context, clusterName string, obj client.Object) error
}
```

**Why watches instead of polling:** With direct API access, we can use real Kubernetes watches instead of the MCV polling pattern. This gives us:
- Lower latency (instant notification vs. polling interval)
- Lower load on managed clusters (one watch vs. repeated MCV creates)
- Simpler code (no MCV lifecycle management)

**Implementation:** Uses informers on remote clusters. Each watch creates a `source.Kind` from the remote cluster's cache and connects it to the local controller's reconcile queue.

### 4.4 `PlacementManager` — Workload Placement

Replaces OCM Placement/PlacementDecision.

```go
// PlacementManager determines which cluster should host a workload.
type PlacementManager interface {
    // GetCurrentDecision returns the current cluster assignment for a DRPC.
    GetCurrentDecision(ctx context.Context, drpc *ramendrv1alpha1.DRPlacementControl,
    ) (*PlacementDecision, error)

    // SetDecision updates the cluster assignment for a DRPC.
    SetDecision(ctx context.Context, drpc *ramendrv1alpha1.DRPlacementControl,
        decision PlacementDecision,
    ) error

    // ClearDecision removes the cluster assignment.
    ClearDecision(ctx context.Context, drpc *ramendrv1alpha1.DRPlacementControl) error

    // SetupWatches registers any watchers needed for placement changes.
    SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error
}

// PlacementDecision represents a cluster assignment.
type PlacementDecision struct {
    ClusterName string
    Reason      string
}
```

**Native backend implementation:**
- `GetCurrentDecision`: Reads `DRPC.Status.PreferredDecision`
- `SetDecision`: Updates `DRPC.Status.PreferredDecision`
- `ClearDecision`: Clears the status field
- `SetupWatches`: No-op (DRPC already watches itself)

**Future external backend:** Could watch an ArgoCD `ApplicationSet` or a custom placement CRD and translate decisions. This is where the pluggable interface pays off.

### 4.5 `ClusterRegistry` — Cluster Discovery and Health

Replaces ManagedCluster + ClusterClaim.

```go
// ClusterRegistry provides information about known clusters.
type ClusterRegistry interface {
    // GetClusterInfo returns metadata and capabilities for a named cluster.
    GetClusterInfo(ctx context.Context, clusterName string) (*ClusterInfo, error)

    // ListClusters returns all known clusters.
    ListClusters(ctx context.Context) ([]ClusterInfo, error)

    // IsReady returns true if the cluster is connected and healthy.
    IsReady(ctx context.Context, clusterName string) (bool, error)

    // SetupWatches registers watchers for cluster state changes.
    SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error
}

// ClusterInfo holds metadata about a cluster.
type ClusterInfo struct {
    Name                          string
    ID                            string
    Ready                         bool
    StorageClasses                []string
    VolumeSnapshotClasses         []string
    VolumeReplicationClasses      []string
    VolumeGroupSnapshotClasses    []string
    VolumeGroupReplicationClasses []string
    NetworkFenceClasses           []string
}
```

**Implementation:** Reads `ClusterConnection` resources. Capabilities come from:
1. `ClusterConnection.Status.Capabilities` (populated by the hub's periodic discovery), or
2. `DRClusterConfig.Status` on the remote cluster (queried via direct API)

### 4.6 `SecretDistributor` — Secret Propagation

Replaces the OCM Policy/ConfigurationPolicy/PlacementBinding chain.

```go
// SecretDistributor manages distributing secrets to remote clusters.
type SecretDistributor interface {
    // DistributeSecret copies a secret from the hub to a remote cluster.
    // The secret is read from sourceRef on the hub and created/updated
    // as targetName in targetNamespace on the remote cluster.
    DistributeSecret(ctx context.Context, opts DistributeSecretOptions) error

    // RemoveSecret removes a previously distributed secret from a remote cluster.
    RemoveSecret(ctx context.Context, clusterName string,
        targetName, targetNamespace string,
    ) error
}

// DistributeSecretOptions configures secret distribution.
type DistributeSecretOptions struct {
    SourceSecretName      string
    SourceSecretNamespace string
    ClusterName           string
    TargetSecretName      string
    TargetSecretNamespace string
    // Format determines how the secret data is transformed before distribution.
    // For example, Velero-format secrets have a different key structure.
    Format SecretFormat
    // OwnerRef allows cleanup when the owner is deleted.
    OwnerRef *metav1.OwnerReference
}
```

**Implementation:** Reads the source secret from the hub cluster, transforms it if needed (e.g., Velero format), and creates/updates it on the remote cluster via direct API. No more Policy chains, no more hub template functions.

### 4.7 `AddonDeployer` — Deploy VolSync to Remote Clusters

Replaces ManagedClusterAddOn.

```go
// AddonDeployer manages deploying add-on components to remote clusters.
type AddonDeployer interface {
    // EnsureAddon ensures the named addon is deployed on the remote cluster.
    // Returns true if the addon is ready.
    EnsureAddon(ctx context.Context, addonName, clusterName string) (bool, error)

    // RemoveAddon removes the addon from the remote cluster.
    RemoveAddon(ctx context.Context, addonName, clusterName string) error
}
```

**Implementation for VolSync:**
- Checks if VolSync CRDs exist on the remote cluster
- If not, deploys VolSync via Helm chart or static manifests
- Returns `ready=true` when the VolSync controller deployment is available

---

## 5. Package Structure

```
internal/controller/
├── multicluster/                    # NEW — Interface definitions
│   ├── interfaces.go                # All interfaces defined above
│   ├── types.go                     # Shared types (PlacementDecision, ClusterInfo, etc.)
│   ├── client_manager.go            # MultiClusterClientManager implementation
│   ├── remote_resource_manager.go   # RemoteResourceManager implementation
│   ├── remote_resource_watcher.go   # RemoteResourceWatcher implementation
│   ├── placement_native.go          # NativePlacementManager (reads DRPC fields)
│   ├── cluster_registry.go          # ClusterRegistry implementation (reads ClusterConnection)
│   ├── secret_distributor.go        # SecretDistributor implementation (direct API)
│   ├── addon_deployer.go            # AddonDeployer implementation (deploys VolSync)
│   └── clusterconnection_controller.go  # ClusterConnection reconciler (health checks, discovery)
├── drplacementcontrol_controller.go # MODIFIED — uses interfaces
├── drpolicy_controller.go           # MODIFIED — uses interfaces
├── drcluster_controller.go          # MODIFIED — uses interfaces
├── drclusterconfig_controller.go    # MODIFIED — remove ClusterClaim code
├── util/
│   ├── mw_util.go                   # DELETED (replaced by RemoteResourceManager)
│   ├── mcv_util.go                  # DELETED (replaced by RemoteResourceManager.Get/List)
│   ├── managedcluster.go            # DELETED (replaced by ClusterRegistry)
│   ├── secrets_util.go              # DELETED (replaced by SecretDistributor)
│   └── ...                          # Other utils remain
├── volsync/
│   ├── secret_propagator.go         # DELETED (replaced by SecretDistributor)
│   ├── deploy_volsync.go            # DELETED (replaced by AddonDeployer)
│   └── ...                          # Other VolSync utils remain
└── ...
```

---

## 6. Controller Refactoring

### 6.1 DRPlacementControl Controller

**Current dependencies to remove:**
- `ManifestWork` → `RemoteResourceManager.Apply/Delete/Get`
- `ManagedClusterView` → `RemoteResourceManager.Get`
- `Placement/PlacementDecision` → `PlacementManager`
- `PlacementRule` → `PlacementManager` (native backend ignores this)

**Structural changes:**

```go
// Before
type DRPlacementControlReconciler struct {
    client.Client
    APIReader      client.Reader
    Log            logr.Logger
    MCVGetter      rmnutil.ManagedClusterViewGetter  // OCM-specific
    Scheme         *runtime.Scheme
    Callback       func(string, string)
    ObjStoreGetter ObjectStoreGetter
}

// After
type DRPlacementControlReconciler struct {
    client.Client
    APIReader              client.Reader
    Log                    logr.Logger
    RemoteResourceManager  multicluster.RemoteResourceManager   // NEW
    RemoteResourceWatcher  multicluster.RemoteResourceWatcher   // NEW
    PlacementManager       multicluster.PlacementManager        // NEW
    Scheme                 *runtime.Scheme
    Callback               func(string, string)
    ObjStoreGetter         ObjectStoreGetter
}
```

**Key code transformations:**

| Before (OCM) | After (Direct API) |
|-------------|-------------------|
| `mwUtil.CreateOrUpdateVRGManifestWork(...)` | `remoteResourceMgr.Apply(ctx, clusterName, []client.Object{vrg}, ...)` |
| `mwUtil.DeleteManifestWork(name, cluster)` | `remoteResourceMgr.Delete(ctx, clusterName, []client.Object{vrg})` |
| `mcvGetter.GetVRGFromManagedCluster(...)` | `remoteResourceMgr.Get(ctx, clusterName, key, &vrg)` |
| `getPlacementDecision(placement)` | `placementMgr.GetCurrentDecision(ctx, drpc)` |
| `setPlacementDecision(placement, decision)` | `placementMgr.SetDecision(ctx, drpc, decision)` |

### 6.2 DRPolicy Controller

**Current dependencies to remove:**
- `ManagedCluster` → `ClusterRegistry`
- `ManagedClusterView` (storage class queries) → `RemoteResourceManager.Get/List` or `ClusterRegistry`
- `Policy/PlacementBinding/PlacementRule` (S3 secrets) → `SecretDistributor`

```go
// After
type DRPolicyReconciler struct {
    client.Client
    APIReader             client.Reader
    Log                   logr.Logger
    Scheme                *runtime.Scheme
    ClusterRegistry       multicluster.ClusterRegistry        // NEW
    RemoteResourceManager multicluster.RemoteResourceManager  // NEW
    SecretDistributor     multicluster.SecretDistributor      // NEW
    ObjectStoreGetter     ObjectStoreGetter
}
```

### 6.3 DRCluster Controller

**Current dependencies to remove:**
- `ManifestWork` (DRClusterConfig deployment) → `RemoteResourceManager`
- `ManagedClusterView` (status queries) → `RemoteResourceManager.Get`
- `ManagedCluster` (cluster validation) → `ClusterRegistry`
- `ManagedClusterAddOn` (VolSync) → `AddonDeployer`

```go
// After
type DRClusterReconciler struct {
    client.Client
    APIReader              client.Reader
    Log                    logr.Logger
    Scheme                 *runtime.Scheme
    RemoteResourceManager  multicluster.RemoteResourceManager  // NEW
    ClusterRegistry        multicluster.ClusterRegistry        // NEW
    AddonDeployer          multicluster.AddonDeployer          // NEW
    ObjectStoreGetter      ObjectStoreGetter
}
```

### 6.4 DRClusterConfig Controller (Cluster-side)

**Current dependencies to remove:**
- `ClusterClaim` CRUD → **Delete entirely.** Capability advertisement now happens via `DRClusterConfig.Status` (already exists) which the hub reads directly via `RemoteResourceManager.Get`. The `ClusterClaim` pruning code is removed.

The DRClusterConfig controller continues to:
1. Discover storage classes, snapshot classes, replication classes
2. Report them in `DRClusterConfig.Status`

The hub reads `DRClusterConfig.Status` directly instead of reading ClusterClaims from ManagedCluster.

### 6.5 cmd/main.go — Wiring

```go
func setupReconcilersHub(mgr ctrl.Manager) {
    // Build the multi-cluster client manager
    clientMgr := multicluster.NewMultiClusterClientManager(mgr.GetClient(), mgr.GetScheme())

    // Build implementations
    resourceMgr := multicluster.NewRemoteResourceManager(clientMgr)
    resourceWatcher := multicluster.NewRemoteResourceWatcher(clientMgr)
    placementMgr := multicluster.NewNativePlacementManager(mgr.GetClient())
    clusterReg := multicluster.NewClusterRegistry(mgr.GetClient(), clientMgr)
    secretDist := multicluster.NewSecretDistributor(mgr.GetClient(), clientMgr)
    addonDepl := multicluster.NewAddonDeployer(clientMgr)

    // Set up ClusterConnection controller (health probes, capability discovery)
    if err := multicluster.NewClusterConnectionReconciler(
        mgr.GetClient(), clientMgr,
    ).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "ClusterConnection")
        os.Exit(1)
    }

    // Inject into hub controllers
    if err := (&controllers.DRPolicyReconciler{
        Client:                mgr.GetClient(),
        APIReader:             mgr.GetAPIReader(),
        Log:                   ctrl.Log.WithName("drp"),
        Scheme:                mgr.GetScheme(),
        ClusterRegistry:       clusterReg,
        RemoteResourceManager: resourceMgr,
        SecretDistributor:     secretDist,
        ObjectStoreGetter:     controllers.S3ObjectStoreGetter(),
    }).SetupWithManager(mgr); err != nil { ... }

    // ... similarly for DRCluster and DRPC controllers
}

func configureController(ramenConfig *ramendrv1alpha1.RamenConfig) error {
    // Hub: register only Ramen + standard K8s schemes
    // NO MORE: plrv1, ocmworkv1, viewv1beta1, cpcv1, gppv1, ocmv1
    if controllers.ControllerType == ramendrv1alpha1.DRHubType {
        utilruntime.Must(argocdv1alpha1hack.AddToScheme(scheme))
        utilruntime.Must(recipe.AddToScheme(scheme))
        // ClusterConnection scheme registration
        utilruntime.Must(ramendrv1alpha1.AddToScheme(scheme))
    }
    // Cluster-side: remove clusterv1alpha1 (ClusterClaim)
    // ...
}
```

---

## 7. Capability Discovery Flow (New)

Today, storage capabilities flow: Managed Cluster → ClusterClaim → ManagedCluster.Status → Hub reads ManagedCluster.

New flow:

```
Managed Cluster                          Hub Cluster
┌──────────────────────┐                ┌──────────────────────────┐
│ DRClusterConfig      │                │ ClusterConnection        │
│ Controller           │                │ Controller               │
│                      │                │                          │
│ 1. Discovers storage │                │ 3. Reads DRClusterConfig │
│    classes, snapshot  │                │    .Status from remote   │
│    classes, etc.      │                │    via direct API        │
│                      │                │                          │
│ 2. Writes to         │  direct API    │ 4. Updates               │
│    DRClusterConfig   │◄───────────────│    ClusterConnection     │
│    .Status           │                │    .Status.Capabilities  │
└──────────────────────┘                └──────────────────────────┘
```

The hub's `ClusterConnection` controller periodically:
1. Connects to the remote cluster via kubeconfig
2. Reads `DRClusterConfig.Status` (which the cluster-side controller populates)
3. Updates `ClusterConnection.Status.Capabilities` and `ClusterConnection.Status.Phase`

This eliminates both ClusterClaim and ManagedClusterView for capability discovery.

---

## 8. Secret Propagation Flow (New)

Today: Hub → Policy → ConfigurationPolicy → PlacementRule → PlacementBinding → managed cluster agent applies secret.

New flow:

```
Hub Cluster                              Managed Cluster
┌──────────────────────────┐            ┌──────────────────────┐
│ SecretDistributor        │            │                      │
│                          │            │                      │
│ 1. Read source Secret    │            │                      │
│    from hub namespace    │  client-go │                      │
│                          │───────────►│ 3. Secret created    │
│ 2. Transform if needed   │            │    directly in       │
│    (Velero format, etc.) │            │    target namespace  │
│                          │            │                      │
└──────────────────────────┘            └──────────────────────┘
```

**Advantages over OCM Policy chain:**
- Instant propagation (no policy processing delay)
- No hub template function complexity
- No PlacementRule/PlacementBinding boilerplate
- Direct error handling (no multi-resource status chain to check)
- Simpler RBAC (just secret read on hub, secret write on managed)

---

## 9. VolSync Deployment Flow (New)

Today: Hub creates `ManagedClusterAddOn` → ACM addon framework deploys VolSync operator.

New flow:

```
Hub Cluster                              Managed Cluster
┌──────────────────────────┐            ┌──────────────────────┐
│ AddonDeployer            │            │                      │
│                          │            │                      │
│ 1. Check if VolSync CRDs│  client-go │                      │
│    exist on remote       │───────────►│ 2. If not present:   │
│                          │            │    - Create namespace │
│ 3. If not present:       │───────────►│    - Deploy VolSync  │
│    Deploy VolSync        │            │      manifests/helm  │
│    manifests             │            │                      │
│                          │            │                      │
│ 4. Check VolSync         │───────────►│ 5. Return Deployment │
│    Deployment status     │◄───────────│    .Status           │
└──────────────────────────┘            └──────────────────────┘
```

**Implementation options for deploying VolSync:**
- **Static manifests:** Embed VolSync deployment manifests and apply via `RemoteResourceManager.Apply`
- **Helm SDK:** Use the Helm Go SDK to install the VolSync chart on the remote cluster
- **Prerequisite docs:** Document that VolSync must be pre-installed (fallback if deployment complexity is too high)

Recommended approach: **static manifests** embedded in the Ramen binary, applied via direct API. This avoids Helm SDK complexity while being fully automated.

---

## 10. RBAC Changes

### Hub Cluster RBAC (remove)

All OCM RBAC markers are deleted:

```
// DELETE these from controller files:
// +kubebuilder:rbac:groups=work.open-cluster-management.io,resources=manifestworks,...
// +kubebuilder:rbac:groups=view.open-cluster-management.io,resources=managedclusterviews,...
// +kubebuilder:rbac:groups=apps.open-cluster-management.io,resources=placementrules,...
// +kubebuilder:rbac:groups=cluster.open-cluster-management.io,resources=placements;placementdecisions;managedclusters,...
// +kubebuilder:rbac:groups=policy.open-cluster-management.io,resources=policies;placementbindings,...
// +kubebuilder:rbac:groups=addon.open-cluster-management.io,resources=managedclusteraddons,...
```

### Hub Cluster RBAC (add)

```
// +kubebuilder:rbac:groups=ramendr.openshift.io,resources=clusterconnections,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=ramendr.openshift.io,resources=clusterconnections/status,verbs=get;update;patch
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch  // For reading kubeconfigs
```

### Managed Cluster RBAC (remove)

```
// DELETE from drclusterconfig_controller.go:
// +kubebuilder:rbac:groups=cluster.open-cluster-management.io,resources=clusterclaims,...
```

### Managed Cluster RBAC

No new RBAC needed on managed clusters. The hub's kubeconfig grants the necessary permissions. The kubeconfig's service account on the managed cluster needs:
- VRG, NetworkFence, MaintenanceMode, DRClusterConfig CRUD
- Namespace create
- Secret create (for S3/VolSync secrets)
- Storage class, snapshot class, replication class read

---

## 11. Implementation Phases

### Phase 1: Foundation (2-3 weeks)

**Goal:** Build the multi-cluster client infrastructure and ClusterConnection CRD without touching existing controllers.

| Task | Description | Files |
|------|-------------|-------|
| 1.1 | Define `ClusterConnection` CRD types | `api/v1alpha1/clusterconnection_types.go` |
| 1.2 | Generate CRD manifests | `make generate manifests` |
| 1.3 | Implement `MultiClusterClientManager` | `internal/controller/multicluster/client_manager.go` |
| 1.4 | Implement `ClusterConnection` reconciler (health probes, capability discovery) | `internal/controller/multicluster/clusterconnection_controller.go` |
| 1.5 | Unit tests for client manager and ClusterConnection controller | `internal/controller/multicluster/*_test.go` |
| 1.6 | Integration test: create ClusterConnection, verify connectivity | `internal/controller/multicluster/suite_test.go` |

**Deliverable:** A working `ClusterConnection` CRD and controller that maintains connections to remote clusters. No existing code is modified.

### Phase 2: Interface Definitions + Direct API Implementations (3-4 weeks)

**Goal:** Implement all interfaces from Section 4 with direct-API backends.

| Task | Description | Files |
|------|-------------|-------|
| 2.1 | Define all interfaces in `interfaces.go` | `internal/controller/multicluster/interfaces.go` |
| 2.2 | Define shared types | `internal/controller/multicluster/types.go` |
| 2.3 | Implement `RemoteResourceManager` | `internal/controller/multicluster/remote_resource_manager.go` |
| 2.4 | Implement `RemoteResourceWatcher` | `internal/controller/multicluster/remote_resource_watcher.go` |
| 2.5 | Implement `NativePlacementManager` | `internal/controller/multicluster/placement_native.go` |
| 2.6 | Implement `ClusterRegistry` | `internal/controller/multicluster/cluster_registry.go` |
| 2.7 | Implement `SecretDistributor` | `internal/controller/multicluster/secret_distributor.go` |
| 2.8 | Implement `AddonDeployer` | `internal/controller/multicluster/addon_deployer.go` |
| 2.9 | Unit tests for each implementation | `internal/controller/multicluster/*_test.go` |

**Deliverable:** All interfaces implemented and tested. No existing code is modified yet.

### Phase 3: Refactor DRClusterConfig Controller (1 week)

**Goal:** Remove ClusterClaim dependency from the cluster-side controller. Easiest controller to refactor — acts as a confidence builder.

| Task | Description | Files |
|------|-------------|-------|
| 3.1 | Remove ClusterClaim CRUD code | `internal/controller/drclusterconfig_controller.go` |
| 3.2 | Remove `clusterv1alpha1` scheme registration from cluster-side | `cmd/main.go` |
| 3.3 | Remove ClusterClaim RBAC markers | `internal/controller/drclusterconfig_controller.go` |
| 3.4 | Update tests | `internal/controller/drclusterconfig_controller_test.go` |

**Deliverable:** DRClusterConfig controller has zero OCM imports.

### Phase 4: Refactor DRCluster Controller (2-3 weeks)

**Goal:** Replace ManifestWork, MCV, and ManagedCluster usage in DRCluster controller.

| Task | Description | Files |
|------|-------------|-------|
| 4.1 | Update struct to accept interfaces | `internal/controller/drcluster_controller.go` |
| 4.2 | Replace ManifestWork calls with `RemoteResourceManager` | `internal/controller/drcluster_controller.go` |
| 4.3 | Replace MCV calls with `RemoteResourceManager.Get` | `internal/controller/drcluster_controller.go` |
| 4.4 | Replace ManagedCluster reads with `ClusterRegistry` | `internal/controller/drcluster_controller.go` |
| 4.5 | Replace ManagedClusterAddOn with `AddonDeployer` | `internal/controller/drcluster_controller.go` |
| 4.6 | Refactor `drcluster_mmode.go` to use interfaces | `internal/controller/drcluster_mmode.go` |
| 4.7 | Update watcher setup to use interface `SetupWatches` | `internal/controller/drcluster_controller.go` |
| 4.8 | Update tests | `internal/controller/drcluster_controller_test.go` |

**Deliverable:** DRCluster controller has zero OCM imports.

### Phase 5: Refactor DRPolicy Controller (2-3 weeks)

**Goal:** Replace ManagedCluster, MCV, and Policy framework usage.

| Task | Description | Files |
|------|-------------|-------|
| 5.1 | Update struct to accept interfaces | `internal/controller/drpolicy_controller.go` |
| 5.2 | Replace ManagedCluster reads with `ClusterRegistry` | `internal/controller/drpolicy_controller.go` |
| 5.3 | Replace MCV storage class queries with `ClusterRegistry.GetClusterInfo` | `internal/controller/drpolicy_peerclass.go` |
| 5.4 | Replace `SecretsUtil` with `SecretDistributor` | `internal/controller/drpolicy_controller.go` |
| 5.5 | Update watcher setup | `internal/controller/drpolicy_controller.go` |
| 5.6 | Update tests | `internal/controller/drpolicy_controller_test.go` |

**Deliverable:** DRPolicy controller has zero OCM imports.

### Phase 6: Refactor DRPC Controller (4-6 weeks)

**Goal:** The largest and most critical refactoring. Replace ManifestWork, MCV, Placement, and PlacementRule in the DRPC controller.

| Task | Description | Files |
|------|-------------|-------|
| 6.1 | Update struct to accept interfaces | `drplacementcontrol_controller.go` |
| 6.2 | Replace all `MWUtil` calls with `RemoteResourceManager` | `drplacementcontrol_controller.go`, `drplacementcontrolvolsync.go` |
| 6.3 | Replace all `MCVGetter` calls with `RemoteResourceManager.Get` | `drplacementcontrol_controller.go`, `drplacementcontrol.go` |
| 6.4 | Replace Placement/PlacementDecision logic with `PlacementManager` | `drplacementcontrol.go` |
| 6.5 | Remove PlacementRule handling | `drplacementcontrol_controller.go` |
| 6.6 | Refactor watcher setup | `drplacementcontrol_watcher.go` |
| 6.7 | Replace VolSync secret propagation | `drplacementcontrolvolsync.go` |
| 6.8 | Make `PlacementRef` optional in DRPC spec | `api/v1alpha1/drplacementcontrol_types.go` |
| 6.9 | Rename OCM-referencing `ProgressionStatus` values | `api/v1alpha1/drplacementcontrol_types.go` |
| 6.10 | Update all tests | `drplacementcontrol_controller_test.go`, etc. |

**This phase should be broken into sub-PRs:**
- 6a: Replace ManifestWork (VRG deployment)
- 6b: Replace MCV (VRG status queries)
- 6c: Replace Placement logic
- 6d: Replace VolSync-specific OCM code
- 6e: Update types and watchers

**Deliverable:** DRPC controller has zero OCM imports.

### Phase 7: Wire Up, Clean Up, Delete OCM Code (2-3 weeks)

**Goal:** Remove all OCM artifacts from the codebase.

| Task | Description | Files |
|------|-------------|-------|
| 7.1 | Update `cmd/main.go` wiring (Section 6.5 of this doc) | `cmd/main.go` |
| 7.2 | Delete `internal/controller/util/mw_util.go` | — |
| 7.3 | Delete `internal/controller/util/mcv_util.go` | — |
| 7.4 | Delete `internal/controller/util/managedcluster.go` | — |
| 7.5 | Delete `internal/controller/util/secrets_util.go` | — |
| 7.6 | Delete `internal/controller/volsync/secret_propagator.go` | — |
| 7.7 | Delete `internal/controller/volsync/deploy_volsync.go` | — |
| 7.8 | Remove all OCM imports from `go.mod` | `go.mod` |
| 7.9 | Run `go mod tidy` | `go.mod`, `go.sum` |
| 7.10 | Remove OCM scheme registrations from `cmd/main.go` | `cmd/main.go` |
| 7.11 | Remove OCM test fixtures and deployers | `e2e/`, `test/` |
| 7.12 | Update Helm charts / deployment manifests | `config/`, `charts/` |
| 7.13 | Verify: `grep -r "open-cluster-management" . --include="*.go"` returns 0 results | — |
| 7.14 | Verify: `grep -r "stolostron" . --include="*.go"` returns 0 results | — |

**Deliverable:** Zero OCM imports anywhere in the codebase. `go.mod` has no OCM dependencies.

### Phase 8: End-to-End Testing (2-3 weeks)

**Goal:** Validate the complete system works without OCM.

| Task | Description |
|------|-------------|
| 8.1 | Set up test environment: 3 plain Kubernetes clusters (kind/k3d), no OCM |
| 8.2 | Create ClusterConnection resources with kubeconfigs |
| 8.3 | Create DRCluster and DRPolicy resources |
| 8.4 | Test initial deployment: create DRPC, verify VRG deployed to preferred cluster |
| 8.5 | Test failover: set action=Failover, verify VRG moves to failover cluster |
| 8.6 | Test relocate: set action=Relocate, verify VRG moves to preferred cluster |
| 8.7 | Test secret propagation: verify S3 secrets appear on managed clusters |
| 8.8 | Test VolSync deployment: verify VolSync is deployed when needed |
| 8.9 | Test cluster disconnect/reconnect: verify ClusterConnection status updates |
| 8.10 | Test on OCM-managed clusters: verify Ramen works alongside OCM (clusters happen to be OCM-managed, but Ramen doesn't use OCM APIs) |

---

## 12. Risk Assessment

### 12.1 Kubeconfig Security and Rotation

**Risk:** Kubeconfig secrets are highly sensitive. If compromised, an attacker gains access to managed clusters.

**Mitigations:**
- Use short-lived service account tokens (bound tokens with expiry)
- Support `exec`-based kubeconfig plugins for dynamic credential providers
- Rotate kubeconfig secrets regularly
- Restrict RBAC on the hub for kubeconfig secret access
- Consider supporting external secret stores (Vault) in a future phase

### 12.2 Network Connectivity Requirements

**Risk:** Direct API access requires network connectivity from hub to all managed clusters. This may not work in air-gapped or firewall-restricted environments.

**Mitigations:**
- Document network requirements clearly
- The pluggable interface design allows a future "agent-pull" backend for restricted environments
- VPN/tunnel solutions can provide connectivity

### 12.3 Watch Scalability

**Risk:** The hub creates Kubernetes watches on remote clusters. At scale (many clusters, many resource types), this could strain both the hub and managed clusters.

**Mitigations:**
- Use informer caching to reduce API server load
- Rate-limit watch reconnections
- Consider watch multiplexing for common resource types
- Benchmark with target cluster count early in Phase 8

### 12.4 DRPC Controller Complexity

**Risk:** The DRPC controller is ~6,000 lines of code tightly coupled to OCM. Phase 6 is the highest-risk phase.

**Mitigations:**
- Break Phase 6 into 5 sub-PRs (6a through 6e)
- Each sub-PR is independently testable
- Maintain a parallel test suite that validates behavior equivalence
- Code review each sub-PR before merging

### 12.5 CRD API Compatibility

**Risk:** Making `PlacementRef` optional and adding `ClusterConnectionRef` to DRCluster changes the API surface.

**Mitigations:**
- Making a required field optional is a non-breaking change for existing resources
- Adding a new optional field is non-breaking
- Both changes are backward compatible
- Use CRD conversion webhooks if needed for version skew

---

## 13. Files Deleted (Complete List)

| File | Lines | Replaced By |
|------|-------|------------|
| `internal/controller/util/mw_util.go` | ~720 | `multicluster.RemoteResourceManager` |
| `internal/controller/util/mcv_util.go` | ~620 | `multicluster.RemoteResourceManager.Get/List` |
| `internal/controller/util/managedcluster.go` | ~118 | `multicluster.ClusterRegistry` |
| `internal/controller/util/secrets_util.go` | ~775 | `multicluster.SecretDistributor` |
| `internal/controller/volsync/secret_propagator.go` | ~346 | `multicluster.SecretDistributor` |
| `internal/controller/volsync/deploy_volsync.go` | ~84 | `multicluster.AddonDeployer` |
| **Total deleted** | **~2,663 lines** | |

---

## 14. Go Module Dependencies Removed

| Module | Version | Purpose (removed) |
|--------|---------|-------------------|
| `open-cluster-management.io/api` | v0.15.0 | ManagedCluster, Placement, ManifestWork |
| `open-cluster-management.io/config-policy-controller` | v0.15.0 | ConfigurationPolicy |
| `open-cluster-management.io/governance-policy-propagator` | v0.16.0 | Policy, PlacementBinding |
| `open-cluster-management.io/multicloud-operators-subscription` | v0.15.0 | ManagedClusterView |
| `github.com/stolostron/multicloud-operators-placementrule` | v1.2.4-... | PlacementRule |

These modules and all their transitive dependencies are removed from `go.mod`.

---

## 15. Estimated Total Effort

| Phase | Scope | Effort |
|-------|-------|--------|
| 1. Foundation | ClusterConnection CRD + client manager | 2-3 weeks |
| 2. Interfaces + implementations | 7 interfaces + direct API backends | 3-4 weeks |
| 3. DRClusterConfig refactor | Remove ClusterClaim (easy win) | 1 week |
| 4. DRCluster refactor | ManifestWork, MCV, ManagedCluster | 2-3 weeks |
| 5. DRPolicy refactor | ManagedCluster, MCV, Policy | 2-3 weeks |
| 6. DRPC refactor | Everything (largest phase) | 4-6 weeks |
| 7. Cleanup + delete | Remove all OCM code and deps | 2-3 weeks |
| 8. E2E testing | Full validation on plain K8s | 2-3 weeks |
| **Total** | | **18-26 weeks** |

---

## 16. Success Criteria

1. **Zero OCM imports:** `grep -r "open-cluster-management\|stolostron" --include="*.go" .` returns no results.
2. **Zero OCM `go.mod` entries:** No `open-cluster-management.io` or `stolostron` modules in `go.mod`.
3. **All tests pass** on plain Kubernetes clusters (no OCM installed).
4. **Works on OCM clusters:** Ramen operates correctly on clusters that happen to be OCM-managed — it just uses kubeconfigs, not OCM APIs.
5. **Feature parity:** Deploy, Failover, and Relocate operations work identically to the OCM-based version.
6. **ClusterConnection CRD** is the only prerequisite for registering remote clusters.
7. **VolSync deployment** is automated via direct API.
8. **Secret propagation** works without the OCM Policy chain.
