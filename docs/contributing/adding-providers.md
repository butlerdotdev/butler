---
title: Adding a New Provider
sidebar_position: 1
---

# Add a New Infrastructure Provider

This guide covers what you need to implement, where to make changes, and what reviewers look for when adding a new infrastructure provider to Butler.

For architecture context on how providers fit into the platform, see [Concepts: Providers](../concepts/providers.md).

**Reference implementations:**
- On-prem: [butler-provider-harvester](https://github.com/butlerdotdev/butler-provider-harvester) (~1,800 LOC)
- On-prem: [butler-provider-nutanix](https://github.com/butlerdotdev/butler-provider-nutanix) (~1,900 LOC)
- Cloud: [butler-provider-aws](https://github.com/butlerdotdev/butler-provider-aws), [butler-provider-gcp](https://github.com/butlerdotdev/butler-provider-gcp), [butler-provider-azure](https://github.com/butlerdotdev/butler-provider-azure)

## Provider Interface Contract

A provider controller watches `MachineRequest` CRDs filtered by provider type, creates VMs on the target infrastructure, polls for IP addresses, and reports status back.

### MachineRequest Lifecycle

Every provider must implement these phase transitions:

```
Pending --> Creating --> Running --> Deleting --> Deleted
                   \              \
                    --> Failed      --> Failed
```

| Phase | Provider Responsibility |
|-------|----------------------|
| `Pending` | Validate config, add finalizer, call provider API to create VM |
| `Creating` | Poll for VM IP address (requeue every 10s) |
| `Running` | Report IP in status. Optional: periodic health check |
| `Failed` | Set `failureReason` and `failureMessage` in status |
| `Deleting` | Call provider API to delete VM |
| `Deleted` | Remove finalizer |

### Critical Requirements

1. **Filter by provider type.** The controller MUST skip MachineRequests that reference a different provider type. Without this, multiple provider controllers fight over the same resources.

2. **Finalizer-gated deletion.** Add a finalizer on first reconcile. Remove it only after the VM is deleted. This prevents orphaned VMs.

3. **Idempotent operations.** Creating a VM that already exists must not fail. Deleting a VM that does not exist must not fail. The controller will be retried on errors.

4. **Event recording.** Record Kubernetes events for VM creation, running, deletion, and failures. These appear in `kubectl describe machinerequest`.

### MachineRequest Spec (Input)

```yaml
apiVersion: butler.butlerlabs.dev/v1alpha1
kind: MachineRequest
metadata:
  name: cluster-cp-0
  namespace: butler-system
spec:
  providerRef:
    name: provider-config
    namespace: butler-system
  machineName: cluster-cp-0
  role: control-plane       # control-plane | worker
  cpu: 4
  memoryMB: 8192
  diskGB: 50
  userData: |
    <Talos machine config or cloud-init>
```

### MachineRequest Status (Output)

```yaml
status:
  phase: Running
  providerID: "proxmox://node1/qemu/100"
  ipAddress: "10.40.1.50"
  ipAddresses: ["10.40.1.50"]
  macAddress: "BC:24:11:AA:BB:CC"
  conditions:
    - type: Ready
      status: "True"
      reason: VMRunning
```

## Required Changes Across Repos

Adding a provider touches 5 repositories. Make changes in this order:

### 1. butler-api

Add the provider type and config struct.

| File | Change |
|------|--------|
| `api/v1alpha1/providerconfig_types.go` | Add `ProviderType{Name}` constant and provider-specific config struct |
| `api/v1alpha1/zz_generated.deepcopy.go` | Run `make generate` |
| `config/crd/bases/` | Run `make manifests` |

Existing provider configs serve as templates. The config struct holds provider-specific fields (API endpoint, network name, image reference, storage class). Reference credentials via `spec.credentialsRef` pointing to a Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: provider-credentials
  namespace: butler-system
type: Opaque
stringData:
  kubeconfig: |        # Harvester/Nutanix: provider kubeconfig or API credentials
    <provider credentials>
```

The Secret key names are provider-specific. See an existing provider's `getCredentials()` function for the expected format.

### 2. butler-provider-{name} (New Repository)

Create the provider controller repository.

| Directory | Purpose |
|-----------|---------|
| `internal/controller/` | MachineRequestReconciler |
| `internal/{provider}/` | Provider API client (SDK wrapper) |
| `cmd/` | Main entrypoint |
| `Dockerfile` | Multi-stage build with `CGO_ENABLED=0` |
| `.github/workflows/` | CI (lint, test, build) and release (image push) |

Scaffold with kubebuilder. The controller watches MachineRequest, the provider client wraps the infrastructure SDK.

### 3. butler-charts

Add a Helm chart for the provider controller.

| File | Change |
|------|--------|
| `charts/butler-provider-{name}/` | New chart directory |
| `charts/butler-provider-{name}/values.yaml` | Image, replicas, resources, RBAC |
| `charts/butler-provider-{name}/templates/rbac.yaml` | MachineRequest and ProviderConfig read/write permissions |
| `charts/butler-crds/hack/sync-crds.sh` | Add CRD mapping if butler-api added new CRDs |

Copy an existing provider chart (e.g., butler-provider-harvester) and adapt.

### 4. butler-cli

Wire the new provider into the bootstrap CLI.

| File | Change |
|------|--------|
| `internal/adm/bootstrap/cmd_{name}.go` | New `butleradm bootstrap {name}` cobra command |
| `internal/adm/bootstrap/orchestrator/` | Add provider to orchestrator switch |
| `manifests/controllers/` | Embed provider controller Deployment + RBAC manifest (copy from an existing provider in this directory) |
| `configs/examples/` | Example bootstrap config file |

The bootstrap command creates a KIND cluster, deploys the provider controller, and creates a ClusterBootstrap CR. Follow the pattern in an existing provider command.

### 5. butler-umbrella (docs)

Add a provider guide to getting-started/.

| File | Change |
|------|--------|
| `docs/getting-started/{name}.md` | Bootstrap guide for the new provider |
| `docs/concepts/providers.md` | Add to the supported providers table |

## Minimal Reconciler Skeleton

The core of a provider controller is a phase-based switch in the `Reconcile` function. This skeleton shows the essential structure:

```go
func (r *MachineRequestReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    mr := &butlerv1alpha1.MachineRequest{}
    if err := r.Get(ctx, req.NamespacedName, mr); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Fetch and validate ProviderConfig
    pc, err := r.getProviderConfig(ctx, mr)
    if err != nil {
        return ctrl.Result{}, err
    }
    if pc.Spec.Provider != butlerv1alpha1.ProviderTypeProxmox { // replace with your provider type
        return ctrl.Result{}, nil // Not our provider
    }

    // Handle deletion
    if !mr.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, mr, pc)
    }

    // Add finalizer
    if !controllerutil.ContainsFinalizer(mr, finalizerName) {
        controllerutil.AddFinalizer(mr, finalizerName)
        return ctrl.Result{Requeue: true}, r.Update(ctx, mr)
    }

    // Phase-based reconciliation
    switch mr.Status.Phase {
    case "", butlerv1alpha1.MachinePhasePending:
        return r.reconcilePending(ctx, mr, pc)   // Create VM
    case butlerv1alpha1.MachinePhaseCreating:
        return r.reconcileCreating(ctx, mr, pc)   // Poll for IP
    case butlerv1alpha1.MachinePhaseRunning:
        return r.reconcileRunning(ctx, mr, pc)    // Health check
    default:
        return ctrl.Result{}, nil
    }
}
```

Each phase handler follows the same pattern: call provider API, update MachineRequest status, return requeue interval. See butler-provider-harvester for the complete implementation.

## Testing Requirements

### Required Tests

| Test Type | What to Cover |
|-----------|--------------|
| Unit: provider filter | Controller skips MachineRequests for other providers |
| Unit: VM create | Pending -> Creating transition, idempotency on existing VM |
| Unit: VM status poll | Creating -> Running transition when IP is reported |
| Unit: VM delete | Finalizer removal after VM deletion, idempotency on missing VM |
| Unit: error handling | Failed phase on API errors, event recording |
| Integration: envtest | Full MachineRequest lifecycle with mocked provider client |
| Manual: real infra | End-to-end bootstrap on actual infrastructure |

### What Reviewers Look For

**Architectural alignment:**
- Phase-based reconciliation following the MachineRequest contract
- Provider type filter prevents cross-provider interference
- Finalizer-gated deletion prevents orphaned VMs
- Clean separation between controller logic and provider SDK client

**Backward compatibility:**
- No breaking changes to butler-api (additive only)
- Existing providers and CRDs unaffected
- ProviderConfig changes are backward-compatible

**Code quality:**
- `slog` for structured logging
- Error messages are actionable (include VM ID, API error)
- No sensitive data in logs (credentials, tokens)
- `CGO_ENABLED=0` in Makefile and Dockerfile

## Contribution Checklist

Use this checklist in your PR description:

```markdown
### butler-api
- [ ] ProviderType constant added
- [ ] Provider config struct added to providerconfig_types.go
- [ ] `make generate && make manifests` passes
- [ ] No changes to existing provider configs

### butler-provider-{name}
- [ ] MachineRequestReconciler implements full phase lifecycle
- [ ] Provider type filter (skips non-matching MachineRequests)
- [ ] Finalizer add/remove implemented
- [ ] Provider API client with create/status/delete operations
- [ ] Unit tests pass (`go test ./...`)
- [ ] Dockerfile builds (`docker build .`)
- [ ] CI workflows (lint, test, build, release)

### butler-charts
- [ ] Helm chart created in charts/butler-provider-{name}/
- [ ] RBAC grants MachineRequest and ProviderConfig access
- [ ] `helm lint` passes

### butler-cli
- [ ] `butleradm bootstrap {name}` command added
- [ ] Controller manifest embedded in manifests/controllers/
- [ ] Example config file added
- [ ] Orchestrator deploys new provider to KIND

### Documentation
- [ ] Provider getting-started guide in butler-umbrella
- [ ] Provider table updated in concepts/providers.md
- [ ] README in the provider repository
```

## Review Process

1. Open a draft PR in each affected repository
2. Complete the checklist above in the butler-provider-{name} PR
3. Request review from a Butler maintainer
4. Iterate on feedback
5. Merge in order: butler-api -> butler-charts -> butler-provider-{name} -> butler-cli -> butler-umbrella

## See Also

- [Concepts: Providers](../concepts/providers.md) -- Two-layer architecture and provider overview
- [MachineRequest CRD](../reference/crds/machinerequest.md) -- Full spec reference
- [ProviderConfig CRD](../reference/crds/providerconfig.md) -- Provider configuration reference
- [Development Setup](development-setup.md) -- Local development environment
