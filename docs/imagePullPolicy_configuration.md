# ImagePullPolicy Configuration for GitOps Operator

## Overview

The GitOps Operator now supports configurable `imagePullPolicy` for all GitOps components, providing administrators with flexibility to optimize network usage and pod startup times based on their environment needs.

## Problem Statement

Previously, GitOps components defaulted to `imagePullPolicy: Always`, which led to:
- Unnecessary image pulls during pod restarts and node maintenance
- Extra network usage in production environments
- Slower pod startup times
- No flexibility for different environments (development vs production)

## Solution

The GitOps Operator now supports configurable `imagePullPolicy` through a three-tier precedence model:

1. **CR-level override** (highest precedence)
2. **Subscription-level environment variable** (middle precedence)  
3. **Default fallback** (lowest precedence - Always)

## Configuration Options

### 1. Subscription-Level Configuration

Set the `IMAGE_PULL_POLICY` environment variable in the GitOps Operator subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  config:
    env:
    - name: IMAGE_PULL_POLICY
      value: "IfNotPresent"  # Options: Always, IfNotPresent, Never
```

### 2. GitOpsService CR Configuration

Override the image pull policy at the CR level:

```yaml
apiVersion: pipelines.openshift.io/v1alpha1
kind: GitopsService
metadata:
  name: cluster
spec:
  imagePullPolicy: IfNotPresent  # Options: Always, IfNotPresent, Never
  runOnInfra: false
  nodeSelector:
    kubernetes.io/os: linux
  tolerations:
  - key: "node-role.kubernetes.io/infra"
    operator: "Exists"
    effect: "NoSchedule"
```

## Precedence Model

The image pull policy is determined using the following precedence:

1. **GitOpsService CR** - If `spec.imagePullPolicy` is specified in the GitOpsService CR, it takes highest precedence
2. **Environment Variable** - If `IMAGE_PULL_POLICY` environment variable is set in the operator subscription, it's used as default
3. **Default Fallback** - If neither is specified, defaults to `Always` for backward compatibility

## Affected Components

The configurable `imagePullPolicy` applies to:

- **GitOps Backend Service** - The backend service deployment
- **Console Plugin** - The OpenShift console plugin deployment
- **Future Components** - Any new GitOps workloads will inherit this configuration

## Valid Values

- `Always` - Always pull the image (default for backward compatibility)
- `IfNotPresent` - Only pull if the image is not already present locally
- `Never` - Never pull the image, only use local images

## Use Cases

### Production Environment
```yaml
# Reduce network usage and improve startup times
imagePullPolicy: IfNotPresent
```

### Development Environment
```yaml
# Always get latest images for testing
imagePullPolicy: Always
```

### Air-gapped Environment
```yaml
# Use only local images
imagePullPolicy: Never
```

## Implementation Details

### Code Changes

1. **Environment Variable Support**: Added `IMAGE_PULL_POLICY` environment variable to operator manager configuration
2. **GitOpsService CRD**: Extended with `imagePullPolicy` field with enum validation
3. **Controller Logic**: Updated controllers to use precedence-based image pull policy resolution
4. **Common Utility**: Added `GetImagePullPolicy()` function to handle precedence logic

### Files Modified

- `api/v1alpha1/gitopsservice_types.go` - Added imagePullPolicy field to GitOpsService spec
- `common/common.go` - Added GetImagePullPolicy utility function
- `controllers/consoleplugin.go` - Updated console plugin to use configurable imagePullPolicy
- `controllers/gitopsservice_controller.go` - Updated backend service to use configurable imagePullPolicy
- `config/manager/manager.yaml` - Added IMAGE_PULL_POLICY environment variable
- `hack/non-olm-install/install-gitops-operator.sh` - Added IMAGE_PULL_POLICY to install script

## Testing

### Verify Configuration

1. Check the GitOpsService CR:
```bash
oc get gitopsservice cluster -o yaml
```

2. Verify deployments are using the correct imagePullPolicy:
```bash
# Check backend service
oc get deployment gitops-backend -n openshift-gitops -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'

# Check console plugin
oc get deployment gitops-plugin -n openshift-gitops -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
```

### Test Scenarios

1. **Default Behavior**: Without any configuration, should default to `Always`
2. **Subscription Override**: Setting `IMAGE_PULL_POLICY` env var should apply to all workloads
3. **CR Override**: GitOpsService `imagePullPolicy` should override subscription setting
4. **Invalid Values**: Should reject invalid values and use default

## Backward Compatibility

- **Default Behavior**: Maintains existing behavior (Always) when no configuration is provided
- **Existing Deployments**: Will be updated to use new imagePullPolicy on next reconciliation
- **API Compatibility**: New field is optional and backward compatible

## Migration Guide

### From Always to IfNotPresent

1. **Test Environment First**: Apply configuration in test environment
2. **Monitor Performance**: Verify improved startup times and reduced network usage
3. **Gradual Rollout**: Apply to production during maintenance window
4. **Rollback Plan**: Can revert by removing configuration or setting back to `Always`

### Example Migration

```bash
# Step 1: Update GitOpsService CR
oc patch gitopsservice cluster --type='merge' -p='{"spec":{"imagePullPolicy":"IfNotPresent"}}'

# Step 2: Verify deployments are updated
oc get deployments -n openshift-gitops -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.template.spec.containers[0].imagePullPolicy}{"\n"}{end}'

# Step 3: Monitor pod restart times
oc get events -n openshift-gitops --sort-by='.lastTimestamp'
```

## Troubleshooting

### Common Issues

1. **Images Not Found**: When using `IfNotPresent` or `Never`, ensure images are available locally
2. **Slow Updates**: With `IfNotPresent`, image updates may be delayed
3. **Network Issues**: In disconnected environments, use `Never` with pre-pulled images

### Debug Commands

```bash
# Check operator logs
oc logs -n openshift-gitops-operator deployment/openshift-gitops-operator-controller-manager

# Check environment variables
oc get deployment openshift-gitops-operator-controller-manager -n openshift-gitops-operator -o jsonpath='{.spec.template.spec.containers[0].env[*]}'

# Verify CRD schema
oc get crd gitopsservices.pipelines.openshift.io -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.imagePullPolicy}'
```

## Future Enhancements

1. **ArgoCD Integration**: Extend to configure imagePullPolicy for ArgoCD components
2. **Rollouts Manager**: Add imagePullPolicy support to RolloutManager CR
3. **Metrics**: Add metrics to track image pull performance
4. **Validation**: Add admission webhooks for enhanced validation

