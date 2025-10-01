# Intro
See a blog post about this solution [here](https://cloud.redhat.com/blog/using-gitops-to-deploy-bare-metal-openshift-hosted-control-plane-clusters).

This is a fork of the original [loganmc10/hypershift-helm](https://github.com/loganmc10/hypershift-helm) repository with enhanced features including ExternalSecret support and separated MetalLB deployment.

## 🚀 New in v0.2.0

- **Unified ExternalSecret Configuration**: Simplified list-based approach for managing all external secrets
- **Type-based Organization**: Separate configuration for `pull-secret` and `bmc-secret` types
- **Improved UX**: Much cleaner and more maintainable configuration structure
- **Flexible Secret Management**: Support for both ExternalSecret and existing secret references
- **Published Helm Repository**: Chart available via `https://yantra-vinyasa.github.io/hypershift-helm/`

# Prerequisites
* OpenShift GitOps Operator v1.8+
* MetalLB Operator (deployed separately on management cluster)
* External Secrets Operator (ESO) - for BMC credential management
* ACM 2.7+ or MCE 2.2+
* Enable the Hosted Control Plane feature: [RHACM Docs - Enable Hypershift Add-On](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/clusters/index#hypershift-addon-intro)
* Enable CIM: [RHACM Docs - Enable CIM](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.7/html-single/clusters/index#enable-cim)

## MetalLB Setup
MetalLB must be deployed separately on the management cluster before deploying this chart. The chart will automatically patch the API service to use the specified MetalLB address pool.

### Required MetalLB Resources
You need to create the following MetalLB resources on your management cluster:

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

## Secret Management with External Secrets

This chart uses External Secrets Operator (ESO) for managing all secrets securely through a unified configuration approach. All external secrets are configured in a single `externalSecrets` list in `values.yaml`.

### Unified External Secret Configuration

The chart supports a simplified, list-based configuration for all external secrets:

```yaml
externalSecrets:
  # Pull secret for container registry authentication
  - name: "assisted-deployment-pull-secret"
    type: "pull-secret"
    # Use existing secret instead of ExternalSecret
    # ref: "my-existing-pull-secret"
    secretStore:
      name: "pull-secret-store"
      kind: "SecretStore"
    remoteKey: "pull-secret"
    remoteProperty: "dockerconfigjson"
  
  # BMC credentials for bare metal hosts
  - name: "bmc-credentials"
    type: "bmc-secret"
    secretStore:
      name: "bmc-secret-store"
      kind: "SecretStore"
    remoteKey: "bmc-credentials"
    data:
      - secretKey: "username"
        remoteProperty: "username"
      - secretKey: "password"
        remoteProperty: "password"
```

### Secret Types

#### Pull Secret (type: "pull-secret")
Creates an ExternalSecret for container registry authentication:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: assisted-deployment-pull-secret
  namespace: <cluster-name>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: pull-secret-store
    kind: SecretStore
  target:
    name: assisted-deployment-pull-secret
    creationPolicy: Owner
  data:
  - secretKey: .dockerconfigjson
    remoteRef:
      key: pull-secret
      property: dockerconfigjson
```

#### BMC Secret (type: "bmc-secret")
Creates individual ExternalSecrets for each BareMetalHost's BMC credentials:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: openshift-worker-0-bmc-secret
  namespace: <cluster-name>
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: bmc-secret-store
    kind: SecretStore
  target:
    name: openshift-worker-0-bmc-secret
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: bmc-credentials
      property: openshift-worker-0-username
  - secretKey: password
    remoteRef:
      key: bmc-credentials
      property: openshift-worker-0-password
```

### Using Existing Secrets

You can reference existing secrets instead of creating ExternalSecrets by using the `ref` field:

```yaml
externalSecrets:
  - name: "assisted-deployment-pull-secret"
    type: "pull-secret"
    ref: "my-existing-pull-secret"  # Use existing secret
```

### Required Secret Stores

You need to create SecretStore resources for the ExternalSecrets to reference:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: bmc-secret-store
  namespace: <cluster-name>
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "bmc-credentials-role"
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: pull-secret-store
  namespace: <cluster-name>
spec:
  provider:
    vault:
      server: "https://vault.example.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "pull-secret-role"
```

I have written an Ansible Playbook that can set up the management cluster as required, see [here](https://github.com/loganmc10/openshift-edge-installer/tree/main/provisioning).

## CI/CD Workflows

This repository includes GitHub Actions workflows for:

- **Build & Test**: Runs on every push and PR to validate chart syntax and run tests
- **Release**: Automatically packages and releases charts when tags are pushed

### Chart Repository

The Helm charts are automatically published as GitHub releases and available via Helm repository. You can install the chart in multiple ways:

#### Option 1: Using Helm Repository (Recommended)
```bash
# Add the repository
helm repo add hypershift-helm https://yantra-vinyasa.github.io/hypershift-helm/
helm repo update

# Install the chart
helm install my-cluster hypershift-helm/deploy-cluster --values your-values.yaml
```

#### Option 2: Direct Download from GitHub Releases
```bash
# Install directly from GitHub releases
helm install my-cluster https://github.com/yantra-vinyasa/hypershift-helm/releases/download/deploy-cluster-0.2.0/deploy-cluster-0.2.0.tgz --values your-values.yaml
```

#### Option 3: Local Development
```bash
# Clone and install locally
git clone https://github.com/yantra-vinyasa/hypershift-helm.git
cd hypershift-helm
helm install my-cluster ./chart --values your-values.yaml
```

#### Available Chart Versions
- **Latest**: `0.2.0` - Unified ExternalSecret configuration with improved UX
- **Repository**: `https://yantra-vinyasa.github.io/hypershift-helm/`

# DNS for the Hosted Cluster
You need to create DNS entries for `api.<hosted-cluster-name>.<domain>` and `*.apps.<hosted-cluster-name>.<domain>`, just like you would for a standalone cluster.
## API
This Helm chart utilizes MetalLB (layer 2) on the **management** cluster in order to serve the API for the hosted cluster. This means that the IP address you choose for the Hosted Cluster API needs to be in the same subnet as the management cluster.
## Ingress
This Helm chart utilizes MetalLB (layer 2) on the **hosted** cluster for the Ingress. This means that the IP address you choose for the Hosted Cluster Ingress needs to be in the same subnet as the hosted cluster workers.
# Configuration
The Values file for the Helm chart is an OpenShift [Install Config](https://docs.openshift.com/container-platform/latest/installing/installing_bare_metal_ipi/ipi-install-installation-workflow.html#additional-resources_config) with an extra `hypershift` section. See [the example](install-config-example.yaml). Hosted Control Plane (HyperShift) clusters do not have any control plane nodes, therefore only worker nodes should be specified in the Install Config.
# Example
Example ArgoCD application:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hypershift-cluster
  namespace: openshift-gitops
spec:
  destination:
    server: https://kubernetes.default.svc
  project: default
  sources:
  - repoURL: 'https://yantra-vinyasa.github.io/hypershift-helm'
    chart: deploy-cluster
    targetRevision: 0.2.0
    helm:
      valueFiles:
      - $values/install-config.yaml
  - repoURL: 'https://git.example.com/org/install-configs.git'
    targetRevision: main
    ref: values
```
