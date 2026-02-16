# Implementation Prompts: OCM/ACM Removal from RamenDR

Each prompt below is a self-contained instruction for implementing one step of the OCM removal blueprint. They are ordered so each builds on the previous, with no orphaned code.

---

## Prompt 1: ClusterConnection CRD Types

```text
In the RamenDR project at /ramen, create a new CRD type called ClusterConnection in the api/v1alpha1/ package. This CRD represents a connection to a remote Kubernetes cluster managed by Ramen's hub operator.

Create the file api/v1alpha1/clusterconnection_types.go with:

1. ClusterConnectionSpec containing:
   - KubeconfigSecretRef (corev1.SecretReference, required) — references a Secret with key "kubeconfig"
   - ClusterID (string, optional) — unique stable cluster identifier, auto-discovered if omitted

2. ClusterConnectionStatus containing:
   - Phase (ClusterConnectionPhase enum: Connected, Disconnected, Error)
   - ClusterID (string) — discovered cluster identity
   - LastProbeTime (*metav1.Time) — last successful contact
   - Capabilities (ClusterCapabilities struct) — discovered storage capabilities
   - Conditions ([]metav1.Condition)

3. ClusterCapabilities struct with fields: StorageClasses, VolumeSnapshotClasses, VolumeReplicationClasses, VolumeGroupSnapshotClasses, VolumeGroupReplicationClasses, NetworkFenceClasses (all []string)

4. The CRD should be cluster-scoped (+kubebuilder:resource:scope=Cluster)
5. Register the types with SchemeBuilder.Register in init()
6. Follow the exact same patterns as drcluster_types.go for SPDX header, package declaration, imports
7. The group is ramendr.openshift.io (same as all other Ramen CRDs)
```

## Prompt 2: Multicluster Interface Definitions

```text
In the RamenDR project, create a new package at internal/controller/multicluster/ containing all interface definitions for multi-cluster operations. These interfaces abstract the communication between Ramen's hub and managed clusters.

Create internal/controller/multicluster/interfaces.go with these interfaces:

1. MultiClusterClientManager — manages Kubernetes clients for remote clusters:
   - GetClient(ctx, clusterName) (client.Client, error)
   - GetAPIReader(ctx, clusterName) (client.Reader, error)
   - IsConnected(clusterName) bool

2. RemoteResourceManager — deploys/manages resources on remote clusters (replaces OCM ManifestWork):
   - Apply(ctx, clusterName string, objects []client.Object, opts ...ApplyOption) error
   - Delete(ctx, clusterName string, objects []client.Object) error
   - Get(ctx, clusterName string, key client.ObjectKey, obj client.Object) error
   - List(ctx, clusterName string, list client.ObjectList, opts ...client.ListOption) error
   - IsApplied(ctx, clusterName string, obj client.Object) (bool, error)

3. RemoteResourceWatcher — watches resources on remote clusters:
   - Watch(ctx, clusterName string, obj client.Object, handler handler.EventHandler, predicates ...predicate.Predicate) error
   - StopWatch(ctx, clusterName string, obj client.Object) error

4. PlacementManager — determines workload cluster placement (replaces OCM Placement):
   - GetCurrentDecision(ctx, drpc) (*PlacementDecision, error)
   - SetDecision(ctx, drpc, decision PlacementDecision) error
   - ClearDecision(ctx, drpc) error
   - SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error

5. ClusterRegistry — cluster discovery and health (replaces OCM ManagedCluster):
   - GetClusterInfo(ctx, clusterName) (*ClusterInfo, error)
   - ListClusters(ctx) ([]ClusterInfo, error)
   - IsReady(ctx, clusterName) (bool, error)
   - SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error

6. SecretDistributor — distributes secrets to remote clusters (replaces OCM Policy chain):
   - DistributeSecret(ctx, opts DistributeSecretOptions) error
   - RemoveSecret(ctx, clusterName, targetName, targetNamespace string) error

7. AddonDeployer — deploys add-ons to remote clusters (replaces OCM ManagedClusterAddOn):
   - EnsureAddon(ctx, addonName, clusterName string) (bool, error)
   - RemoveAddon(ctx, addonName, clusterName string) error

Create internal/controller/multicluster/types.go with shared types: PlacementDecision, ClusterInfo, DistributeSecretOptions, SecretFormat, ApplyOption, and the functional option helpers (WithOrphanOnDelete, WithLabels, WithAnnotations).

Use sigs.k8s.io/controller-runtime types. Import ramendrv1alpha1 for DRPC type reference in PlacementManager. Follow the SPDX header convention from other files.
```

## Prompt 3: MultiClusterClientManager Implementation

```text
In the RamenDR project, implement the MultiClusterClientManager in internal/controller/multicluster/client_manager.go.

This is the foundation of the direct-API approach. It manages cached controller-runtime clients for each remote cluster.

Implementation details:
1. Struct: multiClusterClientManager with fields:
   - hubClient client.Client (to read ClusterConnection and Secret resources on the hub)
   - scheme *runtime.Scheme
   - mu sync.RWMutex (protects the cache)
   - clients map[string]*clusterClient (cache of per-cluster clients)

2. clusterClient struct holds: client.Client, client.Reader (uncached), restConfig *rest.Config, lastRefresh time.Time

3. GetClient(ctx, clusterName):
   - Check cache; return if exists and not stale
   - Read ClusterConnection resource from hub
   - Read kubeconfig Secret referenced by ClusterConnection.Spec.KubeconfigSecretRef
   - Parse kubeconfig using clientcmd.RESTConfigFromKubeConfig
   - Create a new controller-runtime client using client.New(restConfig, client.Options{Scheme: scheme})
   - Cache and return

4. GetAPIReader(ctx, clusterName):
   - Same as GetClient but returns an uncached reader

5. IsConnected(clusterName):
   - Returns true if the cluster has a cached client and last probe was recent

6. Constructor: NewMultiClusterClientManager(hubClient, scheme) returns MultiClusterClientManager

Use these imports: k8s.io/client-go/tools/clientcmd for kubeconfig parsing, sigs.k8s.io/controller-runtime/pkg/client for client creation. The ClusterConnection type is in github.com/ramendr/ramen/api/v1alpha1.

Include proper error handling with wrapped errors. Add a method to invalidate cache for a cluster when kubeconfig changes.
```

## Prompt 4: RemoteResourceManager Implementation

```text
In the RamenDR project, implement RemoteResourceManager in internal/controller/multicluster/remote_resource_manager.go.

This replaces OCM ManifestWork. It uses MultiClusterClientManager to get clients for remote clusters and performs direct CRUD operations.

Implementation:
1. Struct: remoteResourceManager with field clientMgr MultiClusterClientManager

2. Apply(ctx, clusterName, objects, opts):
   - Get client for clusterName via clientMgr.GetClient
   - For each object, use server-side apply: client.Patch with client.Apply
   - Apply labels/annotations from options
   - Handle create-or-update semantics

3. Delete(ctx, clusterName, objects):
   - Get client, delete each object. Ignore NotFound errors.

4. Get(ctx, clusterName, key, obj):
   - Get client, call client.Get(ctx, key, obj)

5. List(ctx, clusterName, list, opts):
   - Get client, call client.List(ctx, list, opts...)

6. IsApplied(ctx, clusterName, obj):
   - Get the resource. If it exists, return true. If NotFound, return false.

7. Constructor: NewRemoteResourceManager(clientMgr) returns RemoteResourceManager

Use controllerutil.CreateOrUpdate where appropriate. Ensure proper field manager name for server-side apply (e.g., "ramen-hub").
```

## Prompt 5: ClusterRegistry Implementation

```text
In the RamenDR project, implement ClusterRegistry in internal/controller/multicluster/cluster_registry.go.

This replaces OCM ManagedCluster for cluster discovery and health checks.

Implementation:
1. Struct: clusterRegistry with fields: hubClient client.Client, clientMgr MultiClusterClientManager

2. GetClusterInfo(ctx, clusterName):
   - Read ClusterConnection resource by name from hub
   - Map to ClusterInfo: Name, ID from status, Ready from phase==Connected
   - Capabilities from status.capabilities

3. ListClusters(ctx):
   - List all ClusterConnection resources
   - Map each to ClusterInfo

4. IsReady(ctx, clusterName):
   - Read ClusterConnection, check status.phase == Connected

5. SetupWatches(builder, handler):
   - Watch ClusterConnection resources

6. Constructor: NewClusterRegistry(hubClient, clientMgr) returns ClusterRegistry
```

## Prompt 6: NativePlacementManager Implementation

```text
In the RamenDR project, implement the native PlacementManager in internal/controller/multicluster/placement_native.go.

This is the "Option C" placement backend. It reads cluster decisions directly from DRPC fields and status.

Implementation:
1. Struct: nativePlacementManager with field: hubClient client.Client

2. GetCurrentDecision(ctx, drpc):
   - Read DRPC.Status.PreferredDecision
   - If ClusterName is set, return it as PlacementDecision
   - If empty, return nil (no decision yet)

3. SetDecision(ctx, drpc, decision):
   - Update DRPC.Status.PreferredDecision.ClusterName = decision.ClusterName
   - Update DRPC.Status.PreferredDecision.ClusterNamespace = decision.ClusterName (namespace matches name)
   - Persist via status update

4. ClearDecision(ctx, drpc):
   - Set DRPC.Status.PreferredDecision to empty
   - Persist via status update

5. SetupWatches(builder, handler):
   - No-op. The DRPC controller already reconciles on its own changes.
   - Return nil.

6. Constructor: NewNativePlacementManager(hubClient) returns PlacementManager
```

## Prompt 7: SecretDistributor Implementation

```text
In the RamenDR project, implement SecretDistributor in internal/controller/multicluster/secret_distributor.go.

This replaces the OCM Policy/ConfigurationPolicy/PlacementBinding chain for distributing S3 and VolSync secrets.

Implementation:
1. Struct: secretDistributor with fields: hubClient client.Client, clientMgr MultiClusterClientManager

2. DistributeSecret(ctx, opts):
   - Read source secret from hub cluster using hubClient.Get
   - Transform secret data based on opts.Format:
     - SecretFormatRamen: copy data keys as-is
     - SecretFormatVelero: restructure keys to Velero format (cloud credentials file)
   - Build target corev1.Secret with opts.TargetSecretName, opts.TargetSecretNamespace
   - Get client for remote cluster via clientMgr.GetClient(ctx, opts.ClusterName)
   - CreateOrUpdate the secret on the remote cluster

3. RemoveSecret(ctx, clusterName, targetName, targetNamespace):
   - Get client for remote cluster
   - Delete the secret. Ignore NotFound.

4. Constructor: NewSecretDistributor(hubClient, clientMgr) returns SecretDistributor

Handle the Velero format transformation: combine AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY into a "[default]\naws_access_key_id=...\naws_secret_access_key=..." format stored under the "cloud" key.
```

## Prompt 8: AddonDeployer Implementation

```text
In the RamenDR project, implement AddonDeployer in internal/controller/multicluster/addon_deployer.go.

This replaces OCM ManagedClusterAddOn for deploying VolSync to managed clusters.

Implementation:
1. Struct: addonDeployer with field: clientMgr MultiClusterClientManager

2. EnsureAddon(ctx, addonName, clusterName):
   - Get client for remote cluster
   - For "volsync": check if VolSync CRDs exist on remote cluster (list CRDs, check for volsync group)
   - If VolSync is already deployed (CRDs exist and controller Deployment is Available), return true
   - If not deployed, apply VolSync deployment manifests (namespace, serviceaccount, deployment, CRDs)
   - Return false if still deploying (Deployment not yet Available)

3. RemoveAddon(ctx, addonName, clusterName):
   - Delete the addon deployment resources from the remote cluster

4. For now, implement a simple check-only version: EnsureAddon checks if VolSync CRDs exist and returns true/false. Full automated deployment can be added later.

5. Constructor: NewAddonDeployer(clientMgr) returns AddonDeployer
```

## Prompt 9: ClusterConnection Reconciler

```text
In the RamenDR project, implement the ClusterConnection controller in internal/controller/multicluster/clusterconnection_controller.go.

This controller reconciles ClusterConnection resources, performing health checks and capability discovery.

Implementation:
1. Struct: ClusterConnectionReconciler with fields:
   - client.Client (hub client)
   - clientMgr MultiClusterClientManager
   - Scheme *runtime.Scheme
   - Log logr.Logger

2. Reconcile(ctx, req):
   a. Fetch ClusterConnection resource
   b. Read kubeconfig secret — if missing, set Phase=Error, condition="KubeconfigSecretNotFound"
   c. Attempt to connect to remote cluster via clientMgr.GetClient
   d. If connection fails, set Phase=Disconnected, update conditions
   e. If connection succeeds:
      - Discover cluster ID: read kube-system namespace UID
      - Read DRClusterConfig.Status from remote cluster for capabilities
      - Set Phase=Connected
      - Set Status.ClusterID
      - Set Status.Capabilities from DRClusterConfig.Status
      - Set Status.LastProbeTime = now
      - Update conditions
   f. Requeue after 30 seconds for periodic health checks

3. SetupWithManager(mgr):
   - Watch ClusterConnection resources
   - Watch Secrets (filter to only kubeconfig secrets referenced by ClusterConnections)

4. RBAC markers:
   - ClusterConnections: get, list, watch, update, patch (status)
   - Secrets: get, list, watch

5. Constructor: NewClusterConnectionReconciler(client, clientMgr, scheme, log)
```

## Prompt 10: Wire Phase 1 into cmd/main.go

```text
In the RamenDR project, update cmd/main.go to wire up the ClusterConnection CRD and its controller.

Changes:
1. Add ClusterConnection types to the hub scheme registration (they're already part of ramendrv1alpha1 since we added them to the api package)

2. In setupReconcilersHub(), add:
   - Create MultiClusterClientManager
   - Create and set up ClusterConnectionReconciler

3. The existing hub controllers (DRPolicy, DRCluster, DRPC) remain unchanged for now — they still use OCM. We're just adding the new ClusterConnection controller alongside them.

This ensures Phase 1 deliverables work end-to-end without touching existing functionality.
```

## Prompt 11: Refactor DRClusterConfig Controller — Remove ClusterClaim

```text
In the RamenDR project, refactor internal/controller/drclusterconfig_controller.go to remove all ClusterClaim (OCM) dependencies.

The DRClusterConfig controller currently creates/prunes ClusterClaim resources (OCM API). These are no longer needed because the hub reads capabilities directly from DRClusterConfig.Status via the ClusterConnection controller.

Changes:
1. Remove all import references to cluster.open-cluster-management.io/v1alpha1
2. Remove the ClusterClaim CRUD functions (pruneAllClusterClaims, etc.)
3. Remove ClusterClaim RBAC markers
4. Keep all storage class discovery logic (it populates DRClusterConfig.Status which the hub reads)
5. In cmd/main.go, remove clusterv1alpha1 scheme registration from the cluster-side configuration
6. Update drclusterconfig_controller_test.go to remove ClusterClaim test expectations
```

## Prompt 12: Refactor DRCluster Controller — Replace OCM Dependencies

```text
In the RamenDR project, refactor internal/controller/drcluster_controller.go to use multicluster interfaces instead of OCM types.

Changes:
1. Update DRClusterReconciler struct:
   - Remove MCVGetter field
   - Add RemoteResourceManager, ClusterRegistry, AddonDeployer fields

2. Replace ManifestWork operations:
   - mwUtil.CreateOrUpdateDRClusterConfigManifestWork → remoteResourceMgr.Apply
   - mwUtil.DeleteManifestWork → remoteResourceMgr.Delete
   - All ManifestWork status checks → remoteResourceMgr.IsApplied / Get

3. Replace ManagedClusterView operations:
   - mcvGetter.GetDRClusterConfigFromManagedCluster → remoteResourceMgr.Get
   - mcvGetter.GetMaintenanceModeFromManagedCluster → remoteResourceMgr.Get
   - mcvGetter.GetNetworkFenceFromManagedCluster → remoteResourceMgr.Get

4. Replace ManagedCluster reads:
   - Reading ManagedCluster for validation → clusterRegistry.IsReady

5. Replace ManagedClusterAddOn:
   - VolSync addon deployment → addonDeployer.EnsureAddon

6. Update drcluster_mmode.go similarly

7. Remove all open-cluster-management.io imports from both files

8. Update watcher setup in SetupWithManager — remove ManifestWork/MCV watches, replace with ClusterConnection watches

9. Update cmd/main.go hub wiring to inject the new dependencies

10. Update tests
```

## Prompt 13: Refactor DRPolicy Controller — Replace OCM Dependencies

```text
In the RamenDR project, refactor internal/controller/drpolicy_controller.go and drpolicy_peerclass.go to use multicluster interfaces.

Changes:
1. Update DRPolicyReconciler struct:
   - Remove MCVGetter field
   - Add ClusterRegistry, RemoteResourceManager, SecretDistributor fields

2. Replace ManagedCluster operations:
   - Watching ManagedCluster for cluster changes → ClusterRegistry.SetupWatches
   - Reading ManagedCluster for validation → ClusterRegistry.IsReady / GetClusterInfo

3. Replace ManagedClusterView operations (peer class discovery):
   - mcvGetter.GetSClassFromManagedCluster → Get from ClusterRegistry.GetClusterInfo.capabilities
   - mcvGetter.GetVRClassFromManagedCluster → same pattern
   - All other MCV-based storage class queries → ClusterRegistry capabilities

4. Replace Policy/Secret operations:
   - SecretsUtil calls for S3 secret propagation → SecretDistributor.DistributeSecret/RemoveSecret

5. Remove all open-cluster-management.io imports

6. Update drpolicy_peerclass.go to use ClusterInfo.capabilities instead of MCV queries

7. Update watcher setup in SetupWithManager

8. Update cmd/main.go hub wiring

9. Update tests
```

## Prompt 14: Refactor DRPC Controller Part A — Replace ManifestWork

```text
In the RamenDR project, refactor the DRPC controller to replace all ManifestWork (MWUtil) usage with RemoteResourceManager.

Target files:
- internal/controller/drplacementcontrol_controller.go
- internal/controller/drplacementcontrol.go
- internal/controller/drplacementcontrolvolsync.go

Changes:
1. Update DRPlacementControlReconciler struct: add RemoteResourceManager field

2. Replace all MWUtil calls:
   - CreateOrUpdateVRGManifestWork → remoteResourceMgr.Apply(ctx, cluster, []client.Object{vrg}, ...)
   - DeleteManifestWork → remoteResourceMgr.Delete
   - IsManifestInAppliedState → remoteResourceMgr.IsApplied
   - FindManifestWork → remoteResourceMgr.Get
   - All other MW operations

3. The MWUtil creates VRG objects and wraps them in ManifestWork. In the new model, we just Apply the VRG directly to the remote cluster.

4. Handle namespace creation: MWUtil creates namespace ManifestWorks. Replace with remoteResourceMgr.Apply for namespace objects.

5. Handle RBAC ClusterRole distribution: Replace ClusterRole ManifestWork with direct Apply.

6. Remove MWUtil import and usage from all three files.

7. Update tests to mock RemoteResourceManager instead of MWUtil.
```

## Prompt 15: Refactor DRPC Controller Part B — Replace ManagedClusterView

```text
In the RamenDR project, replace all ManagedClusterView (MCVGetter) usage in the DRPC controller with RemoteResourceManager.Get.

Target files:
- internal/controller/drplacementcontrol_controller.go
- internal/controller/drplacementcontrol.go

Changes:
1. Remove MCVGetter field from DRPlacementControlReconciler

2. Replace all MCVGetter calls:
   - GetVRGFromManagedCluster(name, ns, cluster) → remoteResourceMgr.Get(ctx, cluster, types.NamespacedName{Name: name, Namespace: ns}, &vrg)
   - GetNamespaceFromManagedCluster(name, cluster) → remoteResourceMgr.Get(ctx, cluster, ..., &ns)
   - DeleteVRGManagedClusterView → no-op (no view to delete)
   - DeleteNamespaceManagedClusterView → no-op

3. The MCVGetter creates MCV resources and waits for results. With direct API, we just call Get — it's synchronous.

4. Remove all MCV-related imports and code.

5. Update tests.
```

## Prompt 16: Refactor DRPC Controller Part C — Replace Placement Logic

```text
In the RamenDR project, replace Placement/PlacementDecision/PlacementRule logic in the DRPC controller with PlacementManager interface.

Target files:
- internal/controller/drplacementcontrol_controller.go
- internal/controller/drplacementcontrol.go
- internal/controller/drplacementcontrol_watcher.go

Changes:
1. Add PlacementManager field to DRPlacementControlReconciler

2. Replace Placement operations:
   - getPlacementOrPlacementRule → placementMgr.GetCurrentDecision
   - setDRPCPlacementDecision → placementMgr.SetDecision
   - clearPlacementDecision → placementMgr.ClearDecision
   - getPlacementDecisionClusters → placementMgr.GetCurrentDecision

3. Remove PlacementRule cloning logic (no longer needed)

4. Remove Placement annotation manipulation (experimental-scheduling-disable)

5. Update drplacementcontrol_watcher.go:
   - Remove Placement/PlacementDecision/PlacementRule watches
   - Add placementMgr.SetupWatches call

6. Make PlacementRef optional in api/v1alpha1/drplacementcontrol_types.go:
   - Change PlacementRef from required to optional pointer
   - Remove the immutability validation (or make it conditional)

7. Rename ProgressionStatus values:
   - ProgressionCreatingMW → ProgressionDeployingResources
   - ProgressionUpdatingPlRule → ProgressionUpdatingPlacement

8. Remove all OCM placement imports.

9. Update tests.
```

## Prompt 17: Refactor DRPC Controller Part D — Replace VolSync OCM Code

```text
In the RamenDR project, replace VolSync-specific OCM code in the DRPC controller.

Target files:
- internal/controller/drplacementcontrolvolsync.go
- internal/controller/volsync/secret_propagator.go
- internal/controller/volsync/deploy_volsync.go

Changes:
1. In drplacementcontrolvolsync.go:
   - Replace VolSync secret propagation calls with SecretDistributor
   - Replace VolSync ManifestWork calls with RemoteResourceManager

2. The volsync/secret_propagator.go file will be deleted entirely — its functionality is replaced by SecretDistributor

3. The volsync/deploy_volsync.go file will be deleted entirely — its functionality is replaced by AddonDeployer

4. Update any imports/references.
```

## Prompt 18: Final Cleanup and OCM Deletion

```text
In the RamenDR project, perform the final cleanup to remove all OCM/ACM code and dependencies.

Changes:
1. Delete OCM utility files:
   - internal/controller/util/mw_util.go
   - internal/controller/util/mcv_util.go
   - internal/controller/util/managedcluster.go
   - internal/controller/util/secrets_util.go
   - internal/controller/volsync/secret_propagator.go
   - internal/controller/volsync/deploy_volsync.go

2. Delete associated test files:
   - internal/controller/util/mw_util_test.go
   - internal/controller/util/secrets_util_test.go
   - internal/controller/volsync/secret_propagator_test.go
   - internal/controller/volsync/deploy_volsync_test.go
   - internal/controller/fake_mcv_test.go

3. Remove OCM scheme registrations from cmd/main.go:
   - plrv1, ocmworkv1, viewv1beta1, cpcv1, gppv1, ocmv1, clusterv1alpha1
   - Remove plrv1 import entirely

4. Remove OCM modules from go.mod:
   - open-cluster-management.io/api
   - open-cluster-management.io/config-policy-controller
   - open-cluster-management.io/governance-policy-propagator
   - open-cluster-management.io/multicloud-operators-subscription
   - github.com/stolostron/multicloud-operators-placementrule

5. Remove the replace directive for k8s.io/client-go (was needed for stolostron)

6. Run go mod tidy

7. Verify: grep -r "open-cluster-management\|stolostron" --include="*.go" returns nothing

8. Remove OCM RBAC markers from all controller files

9. Update any remaining doc comments referencing OCM/ACM
```

## Prompt 19: Update Wiring in cmd/main.go

```text
In the RamenDR project, update cmd/main.go to wire all refactored controllers with the new multicluster implementations.

Changes:
1. In setupReconcilersHub():
   - Create MultiClusterClientManager
   - Create all multicluster implementations: RemoteResourceManager, RemoteResourceWatcher, NativePlacementManager, ClusterRegistry, SecretDistributor, AddonDeployer
   - Inject them into DRPolicyReconciler, DRClusterReconciler, DRPlacementControlReconciler
   - Set up ClusterConnectionReconciler

2. In configureController():
   - Hub side: remove all OCM scheme registrations, keep only Ramen + standard K8s + ArgoCD + Recipe
   - Cluster side: remove clusterv1alpha1

3. Remove OCM imports from the file
```

## Prompt 20: End-to-End Validation

```text
In the RamenDR project, create an e2e test setup for validating the OCM-free Ramen.

Create a test environment using kind clusters:
1. Hub cluster: runs Ramen hub operator
2. Managed cluster A: runs Ramen cluster operator
3. Managed cluster B: runs Ramen cluster operator

Test scenarios:
1. Create ClusterConnection resources on hub with kubeconfigs for clusters A and B
2. Verify ClusterConnection.Status shows Connected and capabilities
3. Create DRCluster and DRPolicy resources
4. Create DRPC with preferredCluster=A, verify VRG deployed to cluster A
5. Set action=Failover, failoverCluster=B, verify VRG moves to cluster B
6. Set action=Relocate, preferredCluster=A, verify VRG moves back
7. Verify S3 secrets are distributed to both clusters
8. Verify DRClusterConfig is deployed to managed clusters
```
