# Implementation Prompts: Make ACM Optional in RamenDR (Backward Compatible)

Each prompt below is a self-contained instruction for implementing one step of the ACM-optional blueprint. They are ordered so each builds on the previous, with no orphaned code. Each prompt stands alone and does not reference other prompts.

---

## Phase 1: Interfaces, Detection, and ACM Backends

### Prompt 1: Create the ACM abstraction package with interface definitions

```text
In the RamenDR project at /ramen, create a new package at internal/controller/acm/ containing interface definitions that abstract the ACM-specific operations in Ramen. These interfaces allow Ramen to use ACM features when available, or fall back to OCM-native alternatives when ACM is not installed.

Create the file internal/controller/acm/interfaces.go with these interfaces:

1. SecretPropagator — distributes secrets from the hub to managed clusters:
   - PropagateSecretToCluster(secretName, sourceNamespace, clusterName, targetNamespace string, format util.TargetSecretFormat, veleroSecretKeyName string) error
   - RemoveSecretFromCluster(secretName, clusterName, namespace string, format util.TargetSecretFormat) error
   - Cleanup(secretName, sourceNamespace string, format util.TargetSecretFormat) error

2. VolSyncSecretPropagator — distributes VolSync PSK secrets:
   - PropagateSecretToClusters(ctx context.Context, sourceSecret *corev1.Secret, ownerObject metav1.Object, destClusters []string, destSecretName, destSecretNamespace string) error
   - CleanupSecretPropagation(ctx context.Context, ownerName, ownerNamespace, secretNamespace string) error

3. AddonDeployer — deploys add-on operators to managed clusters:
   - DeployAddon(ctx context.Context, addonName, clusterName string) error

4. PlacementAdapter — handles PlacementRule (ACM) vs Placement (OCM):
   - GetPlacementObject(ctx context.Context, ref corev1.ObjectReference, namespace string) (client.Object, error)
   - SupportsPlacementRule() bool
   - SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error

Use the types from the existing codebase: util.TargetSecretFormat is in internal/controller/util/secrets_util.go. Import corev1, metav1, client, ctrl, handler from the standard Kubernetes and controller-runtime packages. Follow the SPDX license header convention used throughout the project.
```

### Prompt 2: Implement the ACM detection function

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/detect.go that implements runtime detection of ACM availability on the hub cluster.

The DetectACM function should:
1. Accept a context.Context and a client.Reader (for uncached reads)
2. Check if the CRD "policies.policy.open-cluster-management.io" exists on the hub cluster
3. Return (true, nil) if the CRD exists (ACM is installed)
4. Return (false, nil) if the CRD is not found (ACM not installed)
5. Return (false, err) on any other error

Use k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1 for the CustomResourceDefinition type. Use k8s.io/apimachinery/pkg/api/errors for IsNotFound check. Use sigs.k8s.io/controller-runtime/pkg/client for ObjectKey.

Also add a package-level logger that logs the detection result at Info level:
- "ACM detected on hub cluster — ACM backends will be used"
- "ACM not detected on hub cluster — OCM-native backends will be used"

Follow the SPDX license header and import conventions from other files in the project.
```

### Prompt 3: Implement the ACM secret propagator backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/acm_secret_propagator.go that implements the SecretPropagator interface using the existing ACM Policy chain code.

This is a thin wrapper around the existing util.SecretsUtil struct. It preserves 100% identical behavior to the current code.

Implementation:
1. Struct: acmSecretPropagator with fields: Client client.Client, APIReader client.Reader, Ctx context.Context, Log logr.Logger
2. Constructor: NewACMSecretPropagator(client, apiReader, ctx, log) returns SecretPropagator

3. PropagateSecretToCluster: creates a util.SecretsUtil{Client, APIReader, Ctx, Log} and calls secretsUtil.AddSecretToCluster(secretName, clusterName, sourceNamespace, targetNamespace, format, veleroSecretKeyName)

4. RemoveSecretFromCluster: creates a util.SecretsUtil and calls secretsUtil.RemoveSecretFromCluster(secretName, clusterName, namespace, format)

5. Cleanup: for now, return nil (cleanup is handled by the delete path in the existing code)

The existing code in internal/controller/util/secrets_util.go is NOT modified. The SecretsUtil struct has these methods:
- AddSecretToCluster(secretName, clusterName, namespace, targetNS string, format TargetSecretFormat, veleroNS string) error
- RemoveSecretFromCluster(secretName, clusterName, namespace string, format TargetSecretFormat) error

Import github.com/ramendr/ramen/internal/controller/util for SecretsUtil and TargetSecretFormat.
```

### Prompt 4: Implement the ACM VolSync secret propagator backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/acm_volsync_propagator.go that implements the VolSyncSecretPropagator interface using the existing ACM Policy chain code for VolSync PSK secrets.

This is a thin wrapper around the existing volsync.PropagateSecretToClusters and volsync.CleanupSecretPropagation functions.

Implementation:
1. Struct: acmVolSyncSecretPropagator with field: Client client.Client
2. Constructor: NewACMVolSyncSecretPropagator(client) returns VolSyncSecretPropagator

3. PropagateSecretToClusters: calls volsync.PropagateSecretToClusters(ctx, a.Client, sourceSecret, ownerObject, destClusters, destSecretName, destSecretNamespace, log)

4. CleanupSecretPropagation: calls volsync.CleanupSecretPropagation(ctx, a.Client, ownerName, ownerNamespace, secretNamespace, log)
   Note: The existing CleanupSecretPropagation function signature is:
   func CleanupSecretPropagation(ctx context.Context, k8sClient client.Client, ownerObject metav1.Object, log logr.Logger) error
   Adapt the interface to pass the required parameters.

The existing code in internal/controller/volsync/secret_propagator.go is NOT modified. Import github.com/ramendr/ramen/internal/controller/volsync.
```

### Prompt 5: Implement the ACM addon deployer backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/acm_addon_deployer.go that implements the AddonDeployer interface using the existing ManagedClusterAddOn code.

This is a thin wrapper around the existing volsync.DeployVolSyncToCluster function.

Implementation:
1. Struct: acmAddonDeployer with field: Client client.Client, Log logr.Logger
2. Constructor: NewACMAddonDeployer(client, log) returns AddonDeployer

3. DeployAddon(ctx, addonName, clusterName):
   - If addonName == "volsync", call volsync.DeployVolSyncToCluster(ctx, a.Client, clusterName, a.Log)
   - For other addon names, return an error "unsupported addon"

The existing code in internal/controller/volsync/deploy_volsync.go is NOT modified. Import github.com/ramendr/ramen/internal/controller/volsync.
```

### Prompt 6: Implement the ACM placement adapter backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/acm_placement_adapter.go that implements the PlacementAdapter interface, supporting both PlacementRule and Placement (identical to current behavior).

Implementation:
1. Struct: acmPlacementAdapter with field: Client client.Client
2. Constructor: NewACMPlacementAdapter(client) returns PlacementAdapter

3. GetPlacementObject(ctx, ref, namespace):
   - If ref.Kind == "PlacementRule": get the PlacementRule using plrv1 types and return it
   - If ref.Kind == "Placement": get the Placement using clrapiv1beta1 types and return it
   - Otherwise: return error "unsupported placement kind"

4. SupportsPlacementRule(): return true

5. SetupWatches(builder, handler):
   - Watch both &plrv1.PlacementRule{} and &clrapiv1beta1.Placement{} using the provided handler
   - This replicates the current watcher setup in the DRPC controller

Import:
- plrv1 "github.com/stolostron/multicloud-operators-placementrule/pkg/apis/apps/v1" for PlacementRule
- clrapiv1beta1 "open-cluster-management.io/api/cluster/v1beta1" for Placement
```

---

## Phase 2: OCM-Native Backends

### Prompt 7: Implement the OCM secret propagator backend (ManifestWork-based)

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/ocm_secret_propagator.go that implements the SecretPropagator interface using OCM ManifestWork to deploy secrets directly to managed clusters.

This replaces the ACM Policy/ConfigurationPolicy/PlacementBinding/PlacementRule chain with a single ManifestWork containing the target Secret.

Implementation:
1. Struct: ocmSecretPropagator with fields: Client client.Client, APIReader client.Reader, Ctx context.Context, Log logr.Logger
2. Constructor: NewOCMSecretPropagator(client, apiReader, ctx, log) returns SecretPropagator

3. PropagateSecretToCluster(secretName, sourceNamespace, clusterName, targetNamespace, format, veleroSecretKeyName):
   a. Read the source secret from the hub: client.Get(ctx, {Name: secretName, Namespace: sourceNamespace}, &corev1.Secret{})
   b. Build the target Secret object based on format:
      - SecretFormatRamen: copy all data keys as-is, set namespace to targetNamespace, set name to secretName
      - SecretFormatVelero: combine AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY into a "[default]\naws_access_key_id=...\naws_secret_access_key=..." format under the "cloud" key. Set namespace to veleroSecretKeyName (the Velero namespace), name to util.GenerateVeleroSecretName(secretName)
   c. Create a ManifestWork using util.MWUtil:
      - MWUtil: {Client: o.Client, APIReader: o.APIReader, Ctx: o.Ctx, Log: o.Log, InstName: secretName, TargetNamespace: ""}
      - ManifestWork name: format a unique name like "<secretName>-<format>-secret-mw"
      - Cluster namespace: clusterName (ManifestWork goes in the managed cluster namespace on the hub)
      - Payload: the target Secret object
   d. Use mwUtil.CreateOrUpdateManifestWorkWithNamespace or similar generic method

4. RemoveSecretFromCluster(secretName, clusterName, namespace, format):
   - Delete the ManifestWork for this secret from the cluster namespace
   - Use mwUtil.DeleteManifestWork("<secretName>-<format>-secret-mw", clusterName)

5. Cleanup(secretName, sourceNamespace, format):
   - List and delete all ManifestWorks with the secret-mw naming pattern

Import internal/controller/util for MWUtil. The target Secret must have TypeMeta set (Kind: "Secret", APIVersion: "v1") so ManifestWork can apply it.
```

### Prompt 8: Implement the OCM VolSync secret propagator backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/ocm_volsync_propagator.go that implements the VolSyncSecretPropagator interface using OCM ManifestWork.

This replaces the ACM Policy chain for VolSync PSK secret distribution.

Implementation:
1. Struct: ocmVolSyncSecretPropagator with fields: Client client.Client, Log logr.Logger
2. Constructor: NewOCMVolSyncSecretPropagator(client, log) returns VolSyncSecretPropagator

3. PropagateSecretToClusters(ctx, sourceSecret, ownerObject, destClusters, destSecretName, destSecretNamespace):
   For each cluster in destClusters:
   a. Build a target Secret with:
      - Name: destSecretName
      - Namespace: destSecretNamespace
      - Data: copied from sourceSecret.Data
      - TypeMeta: Kind=Secret, APIVersion=v1
   b. Create a ManifestWork in the cluster namespace on the hub:
      - Use MWUtil to create ManifestWork named "<destSecretName>-volsync-psk-mw"
      - Payload: the target Secret

4. CleanupSecretPropagation(ctx, ownerName, ownerNamespace, secretNamespace):
   - For each cluster that might have the secret, delete the corresponding ManifestWork
   - Use the naming pattern to find and delete ManifestWorks

Import internal/controller/util for MWUtil.
```

### Prompt 9: Implement the OCM addon deployer backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/ocm_addon_deployer.go that implements the AddonDeployer interface using ManifestWork-based deployment instead of ManagedClusterAddOn.

Implementation:
1. Struct: ocmAddonDeployer with fields: Client client.Client, Log logr.Logger
2. Constructor: NewOCMAddonDeployer(client, log) returns AddonDeployer

3. DeployAddon(ctx, addonName, clusterName):
   For "volsync" addon:
   a. Check if VolSync is already deployed on the cluster by querying for the VolSync CRD via ManagedClusterView:
      - Use the existing ManagedClusterViewGetter or MWUtil to create an MCV for CRD "replicationsources.volsync.backube"
      - If the CRD exists, VolSync is already installed — return nil
   b. If not installed, log an informational message:
      "VolSync is not installed on cluster <clusterName>. On ACM hubs, VolSync is deployed automatically via ManagedClusterAddOn. On plain OCM hubs, VolSync must be pre-installed. See documentation for installation instructions."
      Return nil (treat as non-fatal — VolSync may be installed by the user later)

   For other addon names, return error "unsupported addon"

Note: Full ManifestWork-based VolSync deployment (embedding VolSync operator manifests in a ManifestWork) is a future enhancement. For the initial implementation, this backend checks for VolSync and logs a warning if not found, rather than attempting automated deployment. This keeps the scope manageable.
```

### Prompt 10: Implement the OCM placement adapter backend

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/ocm_placement_adapter.go that implements the PlacementAdapter interface, supporting only OCM Placement (no PlacementRule).

Implementation:
1. Struct: ocmPlacementAdapter with field: Client client.Client
2. Constructor: NewOCMPlacementAdapter(client) returns PlacementAdapter

3. GetPlacementObject(ctx, ref, namespace):
   - If ref.Kind == "PlacementRule":
     return nil, fmt.Errorf("PlacementRule is not supported on this hub (ACM not detected). Please migrate to Placement (apiVersion: cluster.open-cluster-management.io/v1beta1, kind: Placement). See Ramen documentation for migration guide.")
   - If ref.Kind == "Placement":
     Get the Placement object using clrapiv1beta1.Placement and return it
   - Otherwise:
     return nil, fmt.Errorf("unsupported placement kind: %s", ref.Kind)

4. SupportsPlacementRule(): return false

5. SetupWatches(builder, handler):
   - Watch only &clrapiv1beta1.Placement{} (NOT PlacementRule since it doesn't exist on this hub)
   - Return nil

Import clrapiv1beta1 "open-cluster-management.io/api/cluster/v1beta1" for Placement.
```

---

## Phase 3: Controller Refactoring

### Prompt 11: Refactor DRPolicy controller to use SecretPropagator interface

```text
In the RamenDR project at /ramen, refactor internal/controller/drpolicy_controller.go and internal/controller/drpolicy.go to use the SecretPropagator interface instead of directly creating util.SecretsUtil.

Changes to drpolicy_controller.go:
1. Add import for "github.com/ramendr/ramen/internal/controller/acm"
2. Add field to DRPolicyReconciler: SecretPropagator acm.SecretPropagator
3. In Reconcile(), remove the inline creation of SecretsUtil:
   - DELETE: secretsUtil := &util.SecretsUtil{Client: r.Client, APIReader: r.APIReader, Ctx: ctx, Log: log}
4. Pass r.SecretPropagator to functions that currently accept *util.SecretsUtil

Changes to drpolicy.go:
1. Update the propagateS3Secret function signature to accept acm.SecretPropagator instead of *util.SecretsUtil
2. Update the drPolicyUndeploy function signature similarly
3. Inside these functions, replace secretsUtil.AddSecretToCluster calls with secretPropagator.PropagateSecretToCluster
4. Replace secretsUtil.RemoveSecretFromCluster calls with secretPropagator.RemoveSecretFromCluster

Do NOT modify internal/controller/util/secrets_util.go — that file is used by the ACM backend and must stay unchanged.

The function propagateS3Secret currently has this signature:
func propagateS3Secret(drpolicy *rmn.DRPolicy, drclusters *rmn.DRClusterList, secretsUtil *util.SecretsUtil, hubOperatorRamenConfig *rmn.RamenConfig, log logr.Logger) error

Change it to:
func propagateS3Secret(drpolicy *rmn.DRPolicy, drclusters *rmn.DRClusterList, secretPropagator acm.SecretPropagator, hubOperatorRamenConfig *rmn.RamenConfig, log logr.Logger) error
```

### Prompt 12: Refactor DRCluster controller to use AddonDeployer interface

```text
In the RamenDR project at /ramen, refactor internal/controller/drcluster_controller.go and internal/controller/drclusters.go to use the AddonDeployer interface instead of directly calling volsync.DeployVolSyncToCluster.

Changes to drcluster_controller.go:
1. Add import for "github.com/ramendr/ramen/internal/controller/acm"
2. Add field to DRClusterReconciler: AddonDeployer acm.AddonDeployer

Changes to drclusters.go:
1. In the drClusterDeploy function, replace the direct call:
   - BEFORE: volsync.DeployVolSyncToCluster(drClusterInstance.ctx, drClusterInstance.client, drcluster.GetName(), drClusterInstance.log)
   - AFTER: drClusterInstance.reconciler.AddonDeployer.DeployAddon(drClusterInstance.ctx, "volsync", drcluster.GetName())

2. Remove the import for "github.com/ramendr/ramen/internal/controller/volsync" from drclusters.go if it was only used for DeployVolSyncToCluster (check if other volsync references exist in the file first)

Do NOT modify internal/controller/volsync/deploy_volsync.go — that file is used by the ACM backend.
```

### Prompt 13: Refactor DRPC controller to use PlacementAdapter interface

```text
In the RamenDR project at /ramen, refactor the DRPC controller to use the PlacementAdapter interface instead of directly handling PlacementRule vs Placement branching.

Target files:
- internal/controller/drplacementcontrol_controller.go
- internal/controller/drplacementcontrol.go
- internal/controller/drplacementcontrol_watcher.go

Changes to drplacementcontrol_controller.go:
1. Add import for "github.com/ramendr/ramen/internal/controller/acm"
2. Add field to DRPlacementControlReconciler: PlacementAdapter acm.PlacementAdapter
3. In SetupWithManager(), replace the direct PlacementRule/Placement watch setup with:
   r.PlacementAdapter.SetupWatches(builder, handler)
   This delegates watch registration to the adapter, which knows whether to watch PlacementRule.

Changes to drplacementcontrol.go and drplacementcontrol_controller.go:
4. Replace calls to getPlacementOrPlacementRule(ctx, client, drpc, log) with:
   r.PlacementAdapter.GetPlacementObject(ctx, drpc.Spec.PlacementRef, drpc.Namespace)
5. Where the code checks if the placement object is a PlacementRule (type assertions like "case *plrv1.PlacementRule:"), use r.PlacementAdapter.SupportsPlacementRule() to gate PlacementRule-specific logic

Note: This is the largest refactoring step because PlacementRule is referenced ~93 times in drplacementcontrol_controller.go. The approach is:
- The existing getPlacementOrPlacementRule function and PlacementRule-specific logic STAYS in the codebase (used by ACM backend)
- The DRPC controller delegates placement decisions through the PlacementAdapter interface
- On ACM hubs, the ACM adapter calls the existing functions
- On OCM hubs, the OCM adapter only handles Placement

Do NOT delete any PlacementRule-related functions — they are used by the ACM placement adapter backend.
```

### Prompt 14: Refactor DRPC controller to use VolSyncSecretPropagator interface

```text
In the RamenDR project at /ramen, refactor internal/controller/drplacementcontrolvolsync.go to use the VolSyncSecretPropagator interface instead of directly calling volsync.PropagateSecretToClusters and volsync.CleanupSecretPropagation.

Changes:
1. Add field to DRPlacementControlReconciler (in drplacementcontrol_controller.go): VolSyncSecretProp acm.VolSyncSecretPropagator

2. In drplacementcontrolvolsync.go, find all calls to:
   - volsync.PropagateSecretToClusters(ctx, client, sourceSecret, owner, destClusters, destName, destNamespace, log)
   Replace with:
   - d.reconciler.VolSyncSecretProp.PropagateSecretToClusters(ctx, sourceSecret, owner, destClusters, destName, destNamespace)

3. Find all calls to:
   - volsync.CleanupSecretPropagation(ctx, client, owner, log)
   Replace with:
   - d.reconciler.VolSyncSecretProp.CleanupSecretPropagation(ctx, ownerName, ownerNamespace, secretNamespace)

Do NOT modify internal/controller/volsync/secret_propagator.go — that file is used by the ACM backend.
```

### Prompt 15: Wire up conditional detection and backend selection in cmd/main.go

```text
In the RamenDR project at /ramen, update cmd/main.go to detect ACM at startup and inject the appropriate backends into controllers.

Changes to setupReconcilersHub():

1. At the top of setupReconcilersHub, add ACM detection:
   acmDetected, err := acm.DetectACM(context.Background(), mgr.GetAPIReader())
   if err != nil {
       setupLog.Error(err, "Failed to detect ACM availability")
       os.Exit(1)
   }

2. Declare interface variables:
   var secretPropagator acm.SecretPropagator
   var volSyncSecretProp acm.VolSyncSecretPropagator
   var addonDeployer acm.AddonDeployer
   var placementAdapter acm.PlacementAdapter

3. If acmDetected:
   - Log "ACM detected — using ACM backends"
   - Create ACM backend instances using New* constructors
   - Keep existing ACM scheme registrations (plrv1, cpcv1, gppv1)

4. If NOT acmDetected:
   - Log "ACM not detected — using OCM-native backends"
   - Create OCM backend instances using New* constructors
   - Do NOT register ACM schemes (plrv1, cpcv1, gppv1)

5. Move the ACM scheme registrations (plrv1.AddToScheme, cpcv1.AddToScheme, gppv1.AddToScheme) from configureController() into the acmDetected branch in setupReconcilersHub()

6. Inject the interface instances into the controller structs:
   - DRPolicyReconciler: add SecretPropagator field
   - DRClusterReconciler: add AddonDeployer field
   - DRPlacementControlReconciler: add PlacementAdapter and VolSyncSecretProp fields

7. Add import for "github.com/ramendr/ramen/internal/controller/acm"

The existing imports for plrv1, cpcv1, gppv1 STAY in cmd/main.go — they are still needed for the ACM backend when ACM is detected.
```

---

## Phase 4: Validation

### Prompt 16: Add unit tests for ACM detection

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/detect_test.go with unit tests for the DetectACM function.

Test cases:
1. "ACM detected when Policy CRD exists" — create a fake client with the Policy CRD, verify DetectACM returns true
2. "ACM not detected when Policy CRD does not exist" — use a fake client without the CRD, verify DetectACM returns false
3. "Error returned on client failure" — use a client that returns an error, verify the error propagates

Use the envtest or fake client from controller-runtime for testing. Follow the existing test patterns in the project (Ginkgo/Gomega if that's the convention, or standard Go testing).
```

### Prompt 17: Add unit tests for OCM secret propagator

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/ocm_secret_propagator_test.go with unit tests for the ManifestWork-based secret propagator.

Test cases:
1. "Propagates secret in Ramen format" — create a source secret on the hub, call PropagateSecretToCluster with SecretFormatRamen, verify a ManifestWork was created in the cluster namespace containing the secret
2. "Propagates secret in Velero format" — same but verify the Velero credential file format
3. "Removes secret ManifestWork" — call RemoveSecretFromCluster, verify the ManifestWork was deleted
4. "Handles missing source secret" — call PropagateSecretToCluster for a non-existent secret, verify error

Use a fake client or envtest. Verify ManifestWork contents by reading the created ManifestWork and checking its spec.workload.manifests contains the expected Secret object.
```

### Prompt 18: Add integration test for dual-backend selection

```text
In the RamenDR project at /ramen, create the file internal/controller/acm/integration_test.go that verifies the correct backend is selected based on ACM availability.

Test scenarios:
1. "With ACM CRDs present, ACM backends are instantiated":
   - Set up a test environment with the Policy CRD registered
   - Call DetectACM, verify it returns true
   - Verify NewACMSecretPropagator returns a working instance
   - Verify the ACM secret propagator delegates to SecretsUtil

2. "Without ACM CRDs, OCM backends are instantiated":
   - Set up a test environment WITHOUT the Policy CRD
   - Call DetectACM, verify it returns false
   - Verify NewOCMSecretPropagator returns a working instance
   - Call PropagateSecretToCluster, verify a ManifestWork is created (not a Policy)

3. "PlacementRule rejected on OCM hub":
   - Create OCM placement adapter
   - Call GetPlacementObject with ref.Kind = "PlacementRule"
   - Verify error contains "PlacementRule is not supported"

4. "PlacementRule accepted on ACM hub":
   - Create ACM placement adapter
   - Create a PlacementRule object
   - Call GetPlacementObject with ref.Kind = "PlacementRule"
   - Verify the PlacementRule is returned successfully

Use envtest for the test environment. Follow existing test patterns in the project.
```
