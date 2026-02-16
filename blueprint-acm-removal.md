# Blueprint: Remove Hard ACM Dependencies from RamenDR (Backward Compatible)

## Executive Summary

This blueprint makes ACM (Advanced Cluster Management) features **optional at runtime** rather than hard requirements. After this work:

- **On ACM hub:** Ramen works **exactly as it does today**. Policy-based secret propagation, PlacementRule support, ManagedClusterAddOn for VolSync — all unchanged.
- **On plain OCM hub (no ACM):** Ramen works using OCM-native alternatives. ManifestWork-based secrets, Placement-only scheduling, ManifestWork-based VolSync deployment.
- **One binary, zero configuration:** Ramen auto-detects whether ACM is available and selects the appropriate code path. No user configuration needed.

### Design Principles

| Principle | Description |
|-----------|-------------|
| **Backward compatible** | Existing ACM deployments continue to work identically |
| **Runtime detection** | ACM availability is detected at startup by checking for ACM CRDs |
| **Interface-based** | ACM-specific operations are behind Go interfaces with two backends |
| **No code deletion** | ACM code moves behind interfaces but is preserved as the ACM backend |
| **Single binary** | Same binary runs on ACM or plain OCM — no build flags, no separate images |

---

## 1. Architecture: Interface-Based ACM Abstraction

```
┌──────────────────────────────────────────────────────────┐
│                    Controller Layer                       │
│  DRPolicy, DRCluster, DRPC controllers                   │
│  (use interfaces, don't care about ACM vs OCM)           │
└────────────┬──────────────────────┬───────────────────────┘
             │                      │
     ┌───────▼─────────┐   ┌───────▼─────────┐
     │  ACM Backend     │   │  OCM Backend     │
     │  (when ACM is    │   │  (when ACM is    │
     │   detected)      │   │   NOT detected)  │
     │                  │   │                  │
     │ Policy chain     │   │ ManifestWork     │
     │ PlacementRule    │   │ Placement only   │
     │ ManagedClusterAO │   │ ManifestWork     │
     └──────────────────┘   └──────────────────┘
```

At operator startup, Ramen checks if ACM CRDs exist on the hub cluster:
- If `policies.policy.open-cluster-management.io` CRD exists → ACM is present → use ACM backend
- Otherwise → use OCM backend

---

## 2. What Changes vs What Stays

### ACM-Specific Features Abstracted (3 interfaces)

| Feature | ACM Backend (existing behavior) | OCM Backend (new) |
|---------|-------------------------------|-------------------|
| Secret propagation | Policy → ConfigurationPolicy → PlacementBinding → PlacementRule | ManifestWork containing Secret |
| Placement support | Both PlacementRule and Placement | Placement only |
| VolSync deployment | ManagedClusterAddOn | ManifestWork-based or prerequisite |

### OCM Features Untouched (no changes)

| Feature | Status |
|---------|--------|
| ManifestWork (VRG, NetworkFence, DRClusterConfig deployment) | Stays as-is |
| ManagedClusterView (cross-cluster queries) | Stays as-is |
| Placement / PlacementDecision | Stays as-is |
| ManagedCluster / ClusterClaim | Stays as-is |
| `mw_util.go`, `mcv_util.go`, `managedcluster.go` | Untouched |

---

## 3. Interface Definitions

All interfaces live in a new package: `internal/controller/acm/`

### 3.1 `SecretPropagator` — Secret Distribution

Abstracts how secrets are distributed from the hub to managed clusters.

```go
// SecretPropagator distributes secrets from the hub to managed clusters.
// The ACM backend uses Policy/ConfigurationPolicy/PlacementBinding/PlacementRule.
// The OCM backend uses ManifestWork containing the Secret directly.
type SecretPropagator interface {
    // PropagateSecretToCluster ensures the named secret is delivered to the
    // specified cluster in the target namespace.
    PropagateSecretToCluster(
        secretName, sourceNamespace, clusterName, targetNamespace string,
        format TargetSecretFormat,
        veleroSecretKeyName string,
    ) error

    // RemoveSecretFromCluster removes the secret from the specified cluster.
    RemoveSecretFromCluster(
        secretName, clusterName string,
        format TargetSecretFormat,
    ) error

    // Cleanup removes all propagation resources for a given secret across all clusters.
    Cleanup(secretName, sourceNamespace string, format TargetSecretFormat) error
}
```

**ACM backend:** Wraps the existing `SecretsUtil` code. `PropagateSecretToCluster` creates Policy/PlacementRule/PlacementBinding.

**OCM backend:** Creates a ManifestWork containing the Secret object. Reads the source secret from the hub, transforms it (Velero format if needed), and wraps it in a ManifestWork.

### 3.2 `VolSyncSecretPropagator` — VolSync PSK Secret Distribution

```go
// VolSyncSecretPropagator distributes VolSync PSK secrets to managed clusters.
type VolSyncSecretPropagator interface {
    // PropagateSecretToClusters propagates a VolSync PSK secret to multiple clusters.
    PropagateSecretToClusters(
        sourceSecret *corev1.Secret,
        ownerObject metav1.Object,
        destClusters []string,
        destSecretName, destSecretNamespace string,
    ) error

    // CleanupSecretPropagation removes all propagation resources for VolSync secrets.
    CleanupSecretPropagation(
        ownerName, ownerNamespace, secretNamespace string,
    ) error
}
```

**ACM backend:** Wraps the existing `volsync.PropagateSecretToClusters` code (Policy chain).

**OCM backend:** Creates ManifestWorks containing the PSK Secret for each target cluster.

### 3.3 `AddonDeployer` — Addon Deployment

```go
// AddonDeployer deploys add-on operators (e.g., VolSync) to managed clusters.
type AddonDeployer interface {
    // DeployAddon ensures the named addon is deployed to the given cluster.
    DeployAddon(ctx context.Context, addonName, clusterName string) error
}
```

**ACM backend:** Wraps the existing `ManagedClusterAddOn` creation code.

**OCM backend:** Checks if VolSync CRDs exist on the cluster (via MCV). If not, deploys VolSync manifests via ManifestWork.

### 3.4 `PlacementAdapter` — Placement Abstraction

```go
// PlacementAdapter handles the difference between PlacementRule (ACM) and
// Placement (OCM) for DRPC's placementRef.
type PlacementAdapter interface {
    // GetPlacementObject returns the referenced placement object (PlacementRule or Placement).
    GetPlacementObject(ctx context.Context, ref corev1.ObjectReference,
        namespace string,
    ) (client.Object, error)

    // SupportsPlacementRule returns true if PlacementRule references are supported.
    SupportsPlacementRule() bool

    // SetupWatches registers watchers for placement resources.
    SetupWatches(builder *ctrl.Builder, handler handler.EventHandler) error
}
```

**ACM backend:** Returns `true` for `SupportsPlacementRule()`, watches both PlacementRule and Placement. Same behavior as today.

**OCM backend:** Returns `false` for `SupportsPlacementRule()`, watches only Placement. Returns a clear error if a PlacementRule is referenced.

---

## 4. Runtime Detection

### ACM Detection at Startup

```go
// DetectACM checks if ACM CRDs are present on the hub cluster.
func DetectACM(ctx context.Context, apiReader client.Reader) (bool, error) {
    // Check for the Policy CRD — if it exists, ACM is installed
    policyCRD := &apiextensionsv1.CustomResourceDefinition{}
    err := apiReader.Get(ctx, client.ObjectKey{
        Name: "policies.policy.open-cluster-management.io",
    }, policyCRD)
    if err != nil {
        if k8serrors.IsNotFound(err) {
            return false, nil // ACM not installed
        }
        return false, err // error checking
    }
    return true, nil // ACM is installed
}
```

### Wiring in `cmd/main.go`

```go
func setupReconcilersHub(mgr ctrl.Manager) {
    // Detect ACM at startup
    acmDetected, err := acm.DetectACM(context.Background(), mgr.GetAPIReader())
    if err != nil {
        setupLog.Error(err, "Failed to detect ACM")
        os.Exit(1)
    }

    var (
        secretPropagator      acm.SecretPropagator
        volSyncSecretProp     acm.VolSyncSecretPropagator
        addonDeployer         acm.AddonDeployer
        placementAdapter      acm.PlacementAdapter
    )

    if acmDetected {
        setupLog.Info("ACM detected — using ACM backends (Policy, PlacementRule, ManagedClusterAddOn)")
        // Register ACM schemes
        utilruntime.Must(plrv1.AddToScheme(scheme))
        utilruntime.Must(cpcv1.AddToScheme(scheme))
        utilruntime.Must(gppv1.AddToScheme(scheme))

        secretPropagator = acm.NewACMSecretPropagator(mgr.GetClient(), mgr.GetAPIReader())
        volSyncSecretProp = acm.NewACMVolSyncSecretPropagator(mgr.GetClient())
        addonDeployer = acm.NewACMManagedClusterAddOnDeployer(mgr.GetClient())
        placementAdapter = acm.NewACMPlacementAdapter(mgr.GetClient())
    } else {
        setupLog.Info("ACM not detected — using OCM-native backends (ManifestWork, Placement)")
        secretPropagator = acm.NewOCMSecretPropagator(mgr.GetClient(), mgr.GetAPIReader())
        volSyncSecretProp = acm.NewOCMVolSyncSecretPropagator(mgr.GetClient())
        addonDeployer = acm.NewOCMAddonDeployer(mgr.GetClient())
        placementAdapter = acm.NewOCMPlacementAdapter(mgr.GetClient())
    }

    // Inject into controllers...
}
```

---

## 5. Package Structure

```
internal/controller/
├── acm/                                # NEW — ACM abstraction layer
│   ├── interfaces.go                   # SecretPropagator, VolSyncSecretPropagator, AddonDeployer, PlacementAdapter
│   ├── detect.go                       # DetectACM() function
│   ├── acm_secret_propagator.go        # ACM backend: wraps existing SecretsUtil code
│   ├── acm_volsync_propagator.go       # ACM backend: wraps existing volsync.PropagateSecretToClusters
│   ├── acm_addon_deployer.go           # ACM backend: wraps existing ManagedClusterAddOn code
│   ├── acm_placement_adapter.go        # ACM backend: supports PlacementRule + Placement
│   ├── ocm_secret_propagator.go        # OCM backend: ManifestWork-based secret distribution
│   ├── ocm_volsync_propagator.go       # OCM backend: ManifestWork-based PSK secret distribution
│   ├── ocm_addon_deployer.go           # OCM backend: ManifestWork-based VolSync deployment
│   └── ocm_placement_adapter.go        # OCM backend: Placement-only
├── util/
│   ├── secrets_util.go                 # STAYS — used by ACM backend
│   ├── mw_util.go                      # STAYS — used by both backends
│   ├── mcv_util.go                     # STAYS — used by both backends
│   ├── managedcluster.go               # STAYS — OCM, untouched
│   └── ...
├── volsync/
│   ├── secret_propagator.go            # STAYS — used by ACM backend
│   ├── deploy_volsync.go               # STAYS — used by ACM backend
│   └── ...
├── drplacementcontrol_controller.go    # MODIFIED — uses PlacementAdapter interface
├── drpolicy_controller.go              # MODIFIED — uses SecretPropagator interface
├── drcluster_controller.go             # MODIFIED — uses AddonDeployer interface
└── ...
```

**Key insight:** No existing files are deleted. The ACM code (`secrets_util.go`, `secret_propagator.go`, `deploy_volsync.go`) stays as-is, wrapped by the ACM backend implementations. The new OCM backend provides alternatives.

---

## 6. Controller Changes

### 6.1 DRPolicy Controller

```go
// Before
type DRPolicyReconciler struct {
    client.Client
    APIReader         client.Reader
    MCVGetter         util.ManagedClusterViewGetter   // OCM — stays
    ObjectStoreGetter ObjectStoreGetter
    // SecretsUtil is created inline in Reconcile()
}

// After
type DRPolicyReconciler struct {
    client.Client
    APIReader          client.Reader
    MCVGetter          util.ManagedClusterViewGetter   // OCM — stays
    SecretPropagator   acm.SecretPropagator            // NEW — injected interface
    ObjectStoreGetter  ObjectStoreGetter
}
```

Change in `Reconcile()`:
```go
// Before:
secretsUtil := &util.SecretsUtil{Client: r.Client, APIReader: r.APIReader, Ctx: ctx, Log: log}
propagateS3Secret(u.object, drclusters, secretsUtil, ramenConfig, u.log)

// After:
propagateS3Secret(u.object, drclusters, r.SecretPropagator, ramenConfig, u.log)
```

### 6.2 DRPC Controller

```go
// Before
type DRPlacementControlReconciler struct {
    client.Client
    MCVGetter      util.ManagedClusterViewGetter   // OCM — stays
    // PlacementRule logic is inline
}

// After
type DRPlacementControlReconciler struct {
    client.Client
    MCVGetter          util.ManagedClusterViewGetter   // OCM — stays
    PlacementAdapter   acm.PlacementAdapter            // NEW — injected interface
    VolSyncSecretProp  acm.VolSyncSecretPropagator     // NEW — injected interface
}
```

Change in placement handling:
```go
// Before:
placementObj, err := getPlacementOrPlacementRule(ctx, client, drpc, log)

// After:
placementObj, err := r.PlacementAdapter.GetPlacementObject(ctx, drpc.Spec.PlacementRef, drpc.Namespace)
```

### 6.3 DRCluster Controller

```go
// Before
type DRClusterReconciler struct {
    client.Client
    MCVGetter         util.ManagedClusterViewGetter   // OCM — stays
    // VolSync deployment is inline call to volsync.DeployVolSyncToCluster
}

// After
type DRClusterReconciler struct {
    client.Client
    MCVGetter          util.ManagedClusterViewGetter   // OCM — stays
    AddonDeployer      acm.AddonDeployer               // NEW — injected interface
}
```

---

## 7. ACM Backend Implementations (Wrapping Existing Code)

The ACM backends are thin wrappers around the existing code. No behavior changes.

### `acm_secret_propagator.go`
```go
type acmSecretPropagator struct {
    client    client.Client
    apiReader client.Reader
}

func (a *acmSecretPropagator) PropagateSecretToCluster(...) error {
    // Delegates to existing SecretsUtil code — zero behavior change
    secretsUtil := &util.SecretsUtil{Client: a.client, APIReader: a.apiReader, ...}
    return secretsUtil.AddSecretToCluster(...)
}
```

### `acm_placement_adapter.go`
```go
type acmPlacementAdapter struct {
    client client.Client
}

func (a *acmPlacementAdapter) GetPlacementObject(...) (client.Object, error) {
    // Delegates to existing getPlacementOrPlacementRule — supports both
    return getPlacementOrPlacementRule(...)
}

func (a *acmPlacementAdapter) SupportsPlacementRule() bool {
    return true // ACM supports PlacementRule
}
```

### `acm_addon_deployer.go`
```go
type acmAddonDeployer struct {
    client client.Client
}

func (a *acmAddonDeployer) DeployAddon(ctx context.Context, addonName, clusterName string) error {
    // Delegates to existing volsync.DeployVolSyncToCluster — zero behavior change
    return volsync.DeployVolSyncToCluster(ctx, a.client, clusterName, log)
}
```

---

## 8. OCM Backend Implementations (New Code)

### `ocm_secret_propagator.go`
```go
type ocmSecretPropagator struct {
    client    client.Client
    apiReader client.Reader
}

func (o *ocmSecretPropagator) PropagateSecretToCluster(
    secretName, sourceNamespace, clusterName, targetNamespace string,
    format TargetSecretFormat, veleroSecretKeyName string,
) error {
    // 1. Read source secret from hub
    sourceSecret := &corev1.Secret{}
    if err := o.client.Get(ctx, types.NamespacedName{
        Name: secretName, Namespace: sourceNamespace,
    }, sourceSecret); err != nil {
        return err
    }

    // 2. Build target secret with format transformation
    targetSecret := buildTargetSecret(sourceSecret, targetNamespace, format, veleroSecretKeyName)

    // 3. Create ManifestWork containing the secret
    mwUtil := &util.MWUtil{Client: o.client, ...}
    mwName := fmt.Sprintf("%s-%s-secret-mw", secretName, format)
    return mwUtil.CreateOrUpdateGenericManifestWork(mwName, clusterName, []interface{}{targetSecret}, nil)
}
```

### `ocm_placement_adapter.go`
```go
type ocmPlacementAdapter struct {
    client client.Client
}

func (o *ocmPlacementAdapter) GetPlacementObject(
    ctx context.Context, ref corev1.ObjectReference, namespace string,
) (client.Object, error) {
    if ref.Kind == "PlacementRule" {
        return nil, fmt.Errorf(
            "PlacementRule is not supported on this hub (ACM not detected). "+
            "Please migrate to Placement (cluster.open-cluster-management.io/v1beta1)")
    }
    // Return Placement object (existing OCM logic)
    placement := &clrapiv1beta1.Placement{}
    if err := o.client.Get(ctx, types.NamespacedName{Name: ref.Name, Namespace: namespace}, placement); err != nil {
        return nil, err
    }
    return placement, nil
}

func (o *ocmPlacementAdapter) SupportsPlacementRule() bool {
    return false
}
```

### `ocm_addon_deployer.go`
```go
type ocmAddonDeployer struct {
    client client.Client
}

func (o *ocmAddonDeployer) DeployAddon(ctx context.Context, addonName, clusterName string) error {
    // Check if VolSync CRDs exist on the cluster via MCV
    // If not, deploy VolSync manifests via ManifestWork
    // If yes, no-op
    mwUtil := &util.MWUtil{Client: o.client, ...}
    // ... ManifestWork-based VolSync deployment
    return nil
}
```

---

## 9. ACM Scheme Registration (Conditional)

Today, ACM schemes are unconditionally registered at startup. After this change, they are registered only when ACM is detected.

```go
// Before (cmd/main.go — unconditional):
if controllers.ControllerType == ramendrv1alpha1.DRHubType {
    utilruntime.Must(plrv1.AddToScheme(scheme))      // always
    utilruntime.Must(cpcv1.AddToScheme(scheme))       // always
    utilruntime.Must(gppv1.AddToScheme(scheme))       // always
    utilruntime.Must(ocmworkv1.AddToScheme(scheme))   // always
    // ...
}

// After (cmd/main.go — conditional):
if controllers.ControllerType == ramendrv1alpha1.DRHubType {
    utilruntime.Must(ocmworkv1.AddToScheme(scheme))   // always (OCM)
    utilruntime.Must(viewv1beta1.AddToScheme(scheme))  // always (OCM)
    utilruntime.Must(clrapiv1beta1.AddToScheme(scheme)) // always (OCM)
    utilruntime.Must(ocmv1.AddToScheme(scheme))        // always (OCM)

    if acmDetected {
        utilruntime.Must(plrv1.AddToScheme(scheme))    // ACM only
        utilruntime.Must(cpcv1.AddToScheme(scheme))     // ACM only
        utilruntime.Must(gppv1.AddToScheme(scheme))     // ACM only
    }
}
```

**Note:** The ACM modules remain in `go.mod` — they're compile-time dependencies. But they're only used at runtime when ACM is detected. This is the same pattern used by many Kubernetes operators that support optional features.

---

## 10. `go.mod` — No Changes

All existing modules stay. The ACM modules are still compiled into the binary so they can be used when ACM is detected:

```
# These ALL stay:
open-cluster-management.io/api v0.15.0                          # OCM
open-cluster-management.io/config-policy-controller v0.15.0     # ACM (used when detected)
open-cluster-management.io/governance-policy-propagator v0.16.0 # ACM (used when detected)
open-cluster-management.io/multicloud-operators-subscription v0.15.0  # OCM
github.com/stolostron/multicloud-operators-placementrule v1.2.4-...   # ACM (used when detected)
```

The `k8s.io/client-go` replace directive also stays (still needed for stolostron).

**Future consideration:** If the go.mod deps themselves are problematic (e.g., transitive dependency conflicts), the ACM backends could be moved to a separate Go module or behind build tags. But that's not required for this phase.

---

## 11. Implementation Phases

### Phase 1: Define Interfaces and ACM Detection (1-2 weeks)

**Goal:** Create the interface layer and detection mechanism without changing any existing behavior.

| Task | Description | Files |
|------|-------------|-------|
| 1.1 | Create `internal/controller/acm/` package with interface definitions | `acm/interfaces.go` |
| 1.2 | Implement `DetectACM()` function | `acm/detect.go` |
| 1.3 | Implement ACM backends (thin wrappers around existing code) | `acm/acm_*.go` |
| 1.4 | Unit tests for detection and ACM backends | `acm/*_test.go` |

**Deliverable:** Interfaces defined. ACM backends wrap existing code with zero behavior changes. Detection works.

### Phase 2: Implement OCM Backends (2-3 weeks)

**Goal:** Build the OCM-native alternatives for each ACM feature.

| Task | Description | Files |
|------|-------------|-------|
| 2.1 | Implement OCM secret propagator (ManifestWork-based) | `acm/ocm_secret_propagator.go` |
| 2.2 | Implement OCM VolSync secret propagator | `acm/ocm_volsync_propagator.go` |
| 2.3 | Implement OCM addon deployer (ManifestWork-based VolSync) | `acm/ocm_addon_deployer.go` |
| 2.4 | Implement OCM placement adapter (Placement only) | `acm/ocm_placement_adapter.go` |
| 2.5 | Unit tests for all OCM backends | `acm/ocm_*_test.go` |

**Deliverable:** All OCM backends implemented and tested in isolation.

### Phase 3: Refactor Controllers to Use Interfaces (2-3 weeks)

**Goal:** Update all controllers to accept interfaces instead of calling ACM code directly.

| Task | Description | Files |
|------|-------------|-------|
| 3.1 | Update DRPolicyReconciler to use `SecretPropagator` | `drpolicy_controller.go` |
| 3.2 | Update DRClusterReconciler to use `AddonDeployer` | `drcluster_controller.go`, `drclusters.go` |
| 3.3 | Update DRPlacementControlReconciler to use `PlacementAdapter` + `VolSyncSecretPropagator` | `drplacementcontrol_controller.go`, `drplacementcontrol.go`, `drplacementcontrol_watcher.go`, `drplacementcontrolvolsync.go` |
| 3.4 | Update `cmd/main.go` wiring with conditional detection | `cmd/main.go` |
| 3.5 | Update tests to work with both backends | Test files |

**Deliverable:** Controllers use interfaces. Detection selects ACM or OCM backend at startup. Existing ACM behavior is preserved.

### Phase 4: Validation (2-3 weeks)

**Goal:** Verify both ACM and OCM paths work correctly.

| Task | Description |
|------|-------------|
| 4.1 | Test on **ACM hub** — verify 100% identical behavior to pre-refactor (regression) |
| 4.2 | Test on **plain OCM hub** — verify ManifestWork secrets, Placement-only, ManifestWork VolSync |
| 4.3 | Test PlacementRule rejection on OCM hub (clear error message) |
| 4.4 | Test PlacementRule still works on ACM hub |
| 4.5 | Test failover / relocate on both hub types |
| 4.6 | Test secret propagation for both S3 and VolSync PSK on both hub types |
| 4.7 | Verify ACM detection log messages at startup |

---

## 12. Estimated Effort

| Phase | Scope | Effort |
|-------|-------|--------|
| 1. Interfaces + ACM detection + ACM backends | Interface layer + wrapping | 1-2 weeks |
| 2. OCM backends | ManifestWork-based alternatives | 2-3 weeks |
| 3. Controller refactoring | Inject interfaces, conditional wiring | 2-3 weeks |
| 4. Validation | E2E on both ACM and OCM hubs | 2-3 weeks |
| **Total** | | **7-11 weeks** |

---

## 13. Risk Assessment

### 13.1 Detection Reliability

**Risk:** ACM CRD detection might give false positives (CRD exists but ACM isn't fully functional) or false negatives (ACM is being installed).

**Mitigations:**
- Check for the Policy CRD specifically — it's the most definitive ACM marker
- Log the detection result clearly at startup
- Add a `ramenConfig.forceACMMode` escape hatch for edge cases
- Recheck periodically or on error (if Policy creation fails, fall back to ManifestWork)

### 13.2 ACM Backend Wrapper Fidelity

**Risk:** The ACM backend wrappers might subtly change behavior by restructuring the call path.

**Mitigations:**
- ACM backends are thin wrappers that delegate directly to existing code
- Existing unit and integration tests validate ACM behavior
- No changes to the existing `secrets_util.go`, `secret_propagator.go`, or `deploy_volsync.go` code

### 13.3 PlacementRule on OCM Hub

**Risk:** Users on a plain OCM hub who have PlacementRule references will get an error.

**Mitigations:**
- Clear error message: "PlacementRule is not supported on this hub (ACM not detected). Please migrate to Placement."
- Migration documentation with exact YAML examples
- This only affects users migrating FROM ACM to OCM — not existing ACM users

### 13.4 ManifestWork Secret Security

**Risk:** ManifestWork payloads contain actual secret data (vs. ACM Policy templates that use hub functions).

**Mitigation:** ManifestWork already carries sensitive resources (VRG with S3 refs). The security model is identical. RBAC on the managed cluster namespace controls access.

---

## 14. Success Criteria

1. **ACM regression:** On ACM hub, product behavior is **100% identical** to pre-refactor — same APIs, same resources created, same flow
2. **OCM works:** On plain OCM hub (no ACM), all DR operations succeed
3. **Auto-detection:** Startup logs show "ACM detected" or "ACM not detected" with correct backend selection
4. **PlacementRule on ACM:** Still works exactly as today
5. **PlacementRule on OCM:** Clear error message with migration guidance
6. **Secret propagation:** Works on both hub types (Policy chain on ACM, ManifestWork on OCM)
7. **VolSync deployment:** Works on both hub types (ManagedClusterAddOn on ACM, ManifestWork on OCM)
8. **No go.mod changes:** All existing dependencies stay. Binary is identical.
9. **No config required:** Zero user configuration needed — detection is automatic

---

## 15. Comparison to Previous Blueprints

| Aspect | This Blueprint (ACM Optional) | Previous ACM Removal | Full OCM Removal |
|--------|------------------------------|---------------------|-----------------|
| ACM on ACM hub | **Works identically** | Broken | Broken |
| ACM code deleted | No — preserved as ACM backend | Yes | Yes |
| go.mod changes | None | 3 modules removed | 5 modules removed |
| Effort | 7-11 weeks | 8-12 weeks | 18-26 weeks |
| New CRDs | None | None | ClusterConnection |
| Risk | Very low (additive, not destructive) | Medium (breaks ACM users) | High (breaks all users) |
| Approach | Interface + runtime detection | Delete + replace | Delete + replace |
