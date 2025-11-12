# RHAI (Red Hat AI) Extensions

This directory contains RHAI-specific extensions for the Kubeflow Trainer operator.

## Purpose

The `rhai/` package provides midstream-specific features that are not part of upstream Kubeflow:
- **Progression tracking**: Real-time training metrics polling and status updates
- **Custom annotations**: RHAI-specific metadata for training jobs
- **Extended RBAC**: Additional permissions

## Structure

```
pkg/rhai/
├── README.md                       # This file
├── setup.go                        # RHAI feature registration
├── controller/
│   └── progression_controller.go  # Wraps base controller with progression tracking
└── progression/
    ├── progression.go              # Core progression tracking logic
    └── progression_test.go         # Tests for progression tracking
```

## How It Works

### 1. Controller Wrapping

The `ProgressionReconciler` wraps the base `TrainJobReconciler` and adds:
- Metrics polling from training pods
- Progress annotation updates
- Automatic requeuing for ongoing polling

### 2. Progression Tracking

When enabled via annotation `trainer.opendatahub.io/progression-tracking: "enabled"`:
- Controller polls training pod's metrics endpoint (default port: 28080)
- Updates TrainJob annotations with real-time progress
- Captures final metrics on job completion/failure

### 3. Manifest Integration

RHAI-specific manifests in `manifests/rhoai/`:
- `rbac_progression_patch.yaml`: Additional RBAC for pod access
- `manager_config_patch.yaml`: ConfigMap mounting for feature flags

## Enabling RHAI Features

RHAI features are controlled via the `ENABLE_RHAI_FEATURES` environment variable. When enabled, the operator uses a wrapping controller that adds progression tracking to the base upstream functionality.

### Integration

The integration is entirely in `cmd/trainer-controller-manager/main.go` - no changes to `pkg/controller/`:

```go
rhaiEnabled := os.Getenv("ENABLE_RHAI_FEATURES") == "true"
if rhaiEnabled {
    // Setup runtime controllers (same as upstream)
    runtimeRec := controller.NewTrainingRuntimeReconciler(...)
    runtimeRec.SetupWithManager(mgr, ctrlpkg.Options{})
    
    clRuntimeRec := controller.NewClusterTrainingRuntimeReconciler(...)
    clRuntimeRec.SetupWithManager(mgr, ctrlpkg.Options{})
    
    // Create base reconciler and wrap with RHAI features
    baseReconciler := controller.NewTrainJobReconciler(...)
    rhaisetup.SetupWithManager(mgr, baseReconciler)
} else {
    // Use upstream setup
    controller.SetupControllers(mgr, runtimes, ctrlpkg.Options{})
}
```

### Deployment Configuration

**For Kubernetes/OpenShift deployments**, set the environment variable in your deployment manifest:

```yaml
# manifests/rhoai/manager_config_patch.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: manager
        env:
        - name: ENABLE_RHAI_FEATURES
          value: "true"
```

**For local development**, export the variable before running:

```bash
export ENABLE_RHAI_FEATURES=true
go run ./cmd/trainer-controller-manager/main.go
```

**For OLM/CSV deployments**, add to the ClusterServiceVersion:

```yaml
spec:
  install:
    spec:
      deployments:
      - spec:
          template:
            spec:
              containers:
              - env:
                - name: ENABLE_RHAI_FEATURES
                  value: "true"
```

## Usage Example

Create a TrainJob with progression tracking:

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pytorch-example
  annotations:
    trainer.opendatahub.io/progression-tracking: "enabled"
    trainer.opendatahub.io/metrics-port: "28080"           # optional, default: 28080
    trainer.opendatahub.io/metrics-poll-interval: "30s"   # optional, default: 30s
spec:
  # ... your training job spec ...
```

The controller will:
1. Poll the primary pod's metrics endpoint every 30s
2. Update the `trainer.opendatahub.io/trainerStatus` annotation with:
   - Progress percentage
   - Current step/epoch
   - Loss and learning rate
   - Time elapsed/remaining
   - Custom metrics
3. Capture final status when job completes

## Rebase Strategy

This package is designed for **minimal-conflict rebasing** from upstream:

### What Won't Conflict:
- ✅ All files in `pkg/rhai/` (new directory, fully isolated)
- ✅ All files in `pkg/controller/` (completely untouched)
- ✅ Patch files in `manifests/rhoai/` (applied via Kustomize)
- ✅ Environment variable in `manager_config_patch.yaml` (RHAI-specific)

### What Needs Attention During Rebase:
- ⚠️ `cmd/trainer-controller-manager/main.go`: ~35-line RHAI integration block in `setupControllers()` (lines 154-194)
  - If upstream changes controller setup API, the RHAI branch may need similar updates
  - The else branch uses upstream `SetupControllers()` unchanged

### No Upstream Code Modifications:
- ✅ Zero changes to `pkg/controller/` (no files modified)
- ✅ Base controller functions remain unexported
- ✅ RHAI wrapper calls only the public `Reconcile()` method
- ✅ All RHAI logic isolated to `pkg/rhai/` and `main.go`

## Development

### Running Tests

```bash
go test ./pkg/rhai/...
```

### Adding New RHAI Features

1. Create new package under `pkg/rhai/yourfeature/`
2. Add controller wrapper if needed in `pkg/rhai/controller/`
3. Update `pkg/rhai/setup.go` to register the feature
4. Add manifest patches in `manifests/rhoai/`
5. Document in this README

## Maintenance

When rebasing from upstream:
1. Pull upstream changes: `git pull upstream master`
2. Rebase: `git rebase upstream/master`
3. `pkg/rhai/` should auto-merge (no conflicts expected)
4. Review controller integration in `main.go` if base controller changed
5. Run tests: `make test`

