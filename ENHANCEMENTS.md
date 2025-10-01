# Helm Chart Enhancements

## Overview
This document outlines the enhancements made to the hypershift-helm chart to separate MetalLB deployment and improve the overall architecture.

## Changes Made

### 1. Removed MetalLB Templates
- **Deleted**: `chart/templates/MetalLB.yaml` - MetalLB resources are now deployed separately
- **Deleted**: `chart/templates/PostSync.yaml` - Removed complex post-installation setup that included MetalLB deployment

### 2. Enhanced PatchApiSvc.yaml
- **Improved error handling**: Added proper error checking and logging
- **Added service availability check**: Waits for API service to be available before patching
- **Enhanced configuration**: Added support for custom MetalLB address pool names via `metallb.apiAddressPool` value
- **Added verification**: Verifies the patch was applied successfully
- **Better logging**: Added informative log messages throughout the process

### 3. ExternalSecret Integration for All Secrets
- **Replaced secret creation with ExternalSecrets**: All secrets are now managed through External Secrets Operator
- **Individual BMC ExternalSecrets**: Each BareMetalHost gets its own ExternalSecret by default
- **Shared BMC ExternalSecret option**: Can use shared credentials when `credentialsName` is specified
- **Pull Secret ExternalSecret**: Pull secrets are managed through ExternalSecrets by default
- **Enhanced security**: All secrets are managed securely through External Secrets Operator
- **Cleaner architecture**: No direct secret creation, only ExternalSecret resources

### 5. Updated Configuration
- **Enhanced install-config-example.yaml**: Added MetalLB configuration section with documentation
- **Updated Chart.yaml**: 
  - Updated description to reflect MetalLB separation
  - Bumped version to 0.2.0
- **Updated README.md**: Added comprehensive MetalLB setup documentation and prerequisites

### 6. Updated Test Configurations
- **Added MetalLB configuration** to all test files (`minimal.yaml`, `full.yaml`, `two_pools.yaml`)
- **Regenerated expected results** to match the new chart behavior
- **All tests now pass** with the new configuration

## New Configuration Options

### MetalLB Configuration
```yaml
metallb:
  apiAddressPool: "<cluster-name>-api-address-pool" # Must match the MetalLB IPAddressPool name
```

### BMC Credentials Configuration
```yaml
platform:
  baremetal:
    hosts:
      - name: openshift-worker-0
        bmc:
          address: "redfish-virtualmedia://<bmc_ip>/redfish/v1/Systems/1"
          # Option 1: Use shared ExternalSecret
          credentialsName: "bmc-credentials"
          # Option 2: Individual ExternalSecret (default when no credentialsName)
```

### Pull Secret Configuration
```yaml
# ExternalSecret is created automatically - no configuration needed
# The chart creates ExternalSecret resources that reference your secret store
```

## Prerequisites Update
- MetalLB Operator must be deployed separately on the management cluster
- External Secrets Operator (ESO) for BMC credential management
- Required MetalLB resources must be created before deploying the chart:
  - IPAddressPool for API VIP
  - L2Advertisement for the address pool
- SecretStore resources for BMC credentials and pull secrets
- External Secrets Operator configured to manage all secrets

## Benefits

1. **Separation of Concerns**: MetalLB deployment is now separate from cluster deployment
2. **Improved Maintainability**: Simpler chart with focused responsibility
3. **Better Error Handling**: Enhanced PatchApiSvc with proper error checking
4. **Flexibility**: Support for custom MetalLB address pool names
5. **Enhanced Security**: All secrets managed through External Secrets Operator
6. **Cleaner Architecture**: No direct secret creation, only ExternalSecret resources
7. **Documentation**: Comprehensive setup instructions for MetalLB and ExternalSecrets

## Migration Guide

### For Existing Users
1. Deploy MetalLB separately on your management cluster
2. Create the required MetalLB resources (IPAddressPool and L2Advertisement)
3. Update your values file to include the `metallb.apiAddressPool` configuration
4. Deploy the updated chart

### Required MetalLB Resources
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: <cluster-name>-api-address-pool
  namespace: metallb-system
spec:
  addresses:
    - <api-vip-address>/32
  autoAssign: false
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: <cluster-name>-api-adv
  namespace: metallb-system
spec:
  ipAddressPools:
    - <cluster-name>-api-address-pool
```

## Version Information
- **Previous Version**: 0.1.24
- **New Version**: 0.2.0
- **Breaking Changes**: Yes - MetalLB must be deployed separately
