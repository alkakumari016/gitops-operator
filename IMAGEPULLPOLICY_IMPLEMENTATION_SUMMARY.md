# ImagePullPolicy Configuration Implementation Summary

## Overview

This implementation adds configurable `imagePullPolicy` support to the GitOps Operator, providing administrators with flexibility to optimize network usage and pod startup times based on their environment needs.

## Key Questions Answered

### 1. What are the current defaults?
- **Before**: All GitOps components defaulted to `imagePullPolicy: Always` (hardcoded)
- **After**: Configurable with fallback to `Always` for backward compatibility

### 2. Is there an existing knob at CR or Subscription level?
- **Before**: No existing configuration options
- **After**: 
  - ✅ **GitOpsService CR**: `spec.imagePullPolicy` field
  - ✅ **Subscription level**: `IMAGE_PULL_POLICY` environment variable

### 3. Precedence model adopted
✅ **Three-tier precedence model implemented**:
1. **CR override** (highest precedence) - `GitOpsService.spec.imagePullPolicy`
2. **Subscription default** (middle precedence) - `IMAGE_PULL_POLICY` env var
3. **Fallback Always** (lowest precedence) - Default for backward compatibility

### 4. Default in disconnected environment
- **Configurable**: Administrators can set `imagePullPolicy: Never` for air-gapped environments
- **Flexible**: Can use `IfNotPresent` to reduce network usage while allowing fallback to local images

## Implementation Details

### Files Modified

1. **`api/v1alpha1/gitopsservice_types.go`**
   - Added `ImagePullPolicy corev1.PullPolicy` field to GitOpsServiceSpec
   - Added kubebuilder validation for enum values (Always, IfNotPresent, Never)

2. **`common/common.go`**
   - Added `ImagePullPolicyEnvVar` constant
   - Added `GetImagePullPolicy()` function implementing precedence logic

3. **`controllers/consoleplugin.go`**
   - Updated `getPluginPodSpec()` to accept imagePullPolicy parameter
   - Updated `pluginDeployment()` to use configurable imagePullPolicy
   - Updated `reconcileDeployment()` to use precedence-based policy resolution

4. **`controllers/gitopsservice_controller.go`**
   - Updated `newBackendDeployment()` to accept imagePullPolicy parameter
   - Updated backend deployment creation to use configurable imagePullPolicy

5. **`config/manager/manager.yaml`**
   - Added `IMAGE_PULL_POLICY` environment variable with default value "Always"

6. **`hack/non-olm-install/install-gitops-operator.sh`**
   - Added `IMAGE_PULL_POLICY` environment variable support for non-OLM installations

7. **CRD Manifests**
   - Generated updated GitOpsService CRD with imagePullPolicy field and enum validation
   - Updated bundle manifests

### Components Affected

✅ **GitOps Backend Service** - The backend service deployment
✅ **Console Plugin** - The OpenShift console plugin deployment
✅ **Future Components** - Any new GitOps workloads will inherit this configuration

### ArgoCD and Rollouts Manager Analysis

- **ArgoCD CRD**: Already supports `imagePullPolicy` at individual component level, no global field needed
- **Rollouts Manager CR**: Already supports `imagePullPolicy` at container level in existing CRDs
- **GitOpsService CR**: Now supports global `imagePullPolicy` for operator-managed workloads

## Usage Examples

### 1. Subscription-Level Configuration
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
```

## Environment-Specific Configurations

### Production Environment
```yaml
imagePullPolicy: IfNotPresent  # Reduce network usage and improve startup times
```

### Development Environment
```yaml
imagePullPolicy: Always  # Always get latest images for testing
```

### Air-gapped Environment
```yaml
imagePullPolicy: Never  # Use only local images
```

## Testing

### Unit Tests
- ✅ `common/common_test.go` - Tests precedence logic and edge cases
- ✅ All test scenarios pass

### Verification Commands
```bash
# Check GitOpsService CR
oc get gitopsservice cluster -o yaml

# Verify backend service imagePullPolicy
oc get deployment cluster -n openshift-gitops -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'

# Verify console plugin imagePullPolicy
oc get deployment gitops-plugin -n openshift-gitops -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
```

## Backward Compatibility

✅ **Maintains existing behavior** - Defaults to `Always` when no configuration is provided
✅ **API compatibility** - New field is optional and backward compatible
✅ **Existing deployments** - Will be updated on next reconciliation

## Benefits

1. **Network Optimization**: Reduce unnecessary image pulls in production
2. **Faster Startup Times**: Use cached images when appropriate
3. **Air-gapped Support**: Configure for disconnected environments
4. **Flexibility**: Different policies for different environments
5. **Operational Control**: Fine-grained control over image pull behavior

## Migration Path

1. **Test Environment**: Apply configuration and verify behavior
2. **Monitor Performance**: Confirm improved startup times and reduced network usage
3. **Production Rollout**: Apply during maintenance window
4. **Rollback Plan**: Remove configuration or set back to `Always` if needed

## Future Enhancements

The implementation provides a foundation for:
1. **ArgoCD Integration**: Extend to configure imagePullPolicy for ArgoCD components
2. **Rollouts Manager**: Add imagePullPolicy support to RolloutManager CR
3. **Additional Components**: Apply to any future GitOps workloads
