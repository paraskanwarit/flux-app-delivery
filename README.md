# Flux App Delivery Repository

This repository contains GitOps configuration manifests for deploying applications to Kubernetes using FluxCD. It serves as the "delivery" layer in a GitOps architecture, defining how applications should be deployed without containing the application code itself.

## 🏗️ Repository Architecture

This repository follows the GitOps pattern where:
- **Infrastructure** is managed separately (gke-gitops-infra repository)
- **Application code & charts** are in separate repositories (sample-app-helm-chart)
- **Deployment configuration** is managed here (flux-app-delivery)

## 📁 Repository Structure

```
flux-app-delivery/
├── README.md                           # This documentation
├── kustomization.yaml                  # Kustomize configuration for Flux
├── namespaces/
│   └── sample-app-namespace.yaml       # Kubernetes namespace definition
├── helmrelease/
│   ├── sample-app-helmrepository.yaml  # GitRepository source definition
│   └── sample-app-helmrelease.yaml     # HelmRelease deployment definition
└── diagrams/                           # Architecture diagrams
    ├── diagram-delivery.md
    ├── diagram-end-to-end.md
    └── diagram-change-workflow.md
```

## 🔧 Kubernetes Objects Explained

### 1. Kustomization (`kustomization.yaml`)

**What it is:** A Kustomize configuration file that defines which Kubernetes manifests to include.

**Why we use it:** Flux uses Kustomize to organize and apply multiple Kubernetes resources as a single unit. This ensures all related objects are deployed together.

### 2. Namespace (`namespaces/sample-app-namespace.yaml`)

**What it is:** A Kubernetes namespace that provides resource isolation and organization.

**Why we use it:** 
- Isolates application resources from other applications
- Provides security boundaries through RBAC
- Enables resource quotas and limits per application
- Makes it easier to manage and clean up application resources

### 3. GitRepository (`helmrelease/sample-app-helmrepository.yaml`)

**What it is:** A Flux Custom Resource that defines a Git repository as a source for Kubernetes manifests or Helm charts.

**Why we use it:**
- Tells Flux where to find the Helm chart source code
- Enables automatic polling for changes in the chart repository
- Provides authentication and access control for private repositories
- Caches chart content locally for faster deployments

**Configuration:**
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: sample-app-helm-chart        # Reference name used by HelmRelease
  namespace: flux-system              # Must be in flux-system namespace
spec:
  interval: 1m0s                      # Check for changes every minute
  url: https://github.com/paraskanwarit/sample-app-helm-chart.git
  ref:
    branch: main                      # Track the main branch
```

### 4. HelmRelease (`helmrelease/sample-app-helmrelease.yaml`)

**What it is:** A Flux Custom Resource that defines how to deploy a Helm chart to Kubernetes.

**Why we use it:**
- Provides declarative Helm chart deployments
- Enables GitOps workflow for Helm-based applications
- Supports automatic upgrades when chart or values change
- Allows environment-specific value overrides
- Provides rollback capabilities and deployment history

**Configuration:**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: sample-app2                   # Name of this HelmRelease
  namespace: sample-app               # Target namespace for deployment
spec:
  interval: 5m                        # How often to reconcile
  chart:
    spec:
      chart: charts/sample-app        # Path to chart in repository
      version: "0.1.2"               # Specific chart version to deploy
      sourceRef:
        kind: GitRepository           # Reference to GitRepository object
        name: sample-app-helm-chart   # Name of GitRepository
        namespace: flux-system        # Namespace where GitRepository exists
  values: {}                          # Override chart default values
```

## 🔄 How GitOps Works Here

### 1. Initial Deployment Flow
1. **Flux Source Controller** monitors this repository for changes
2. **Kustomization** tells Flux which manifests to apply
3. **Namespace** is created first to provide deployment target
4. **GitRepository** object tells Flux where to find the Helm chart
5. **HelmRelease** object tells Flux how to deploy the chart
6. **Flux Helm Controller** fetches chart and deploys application

### 2. Change Detection & Updates
1. **Chart Updates**: When developers update the Helm chart repository, Flux detects changes and redeploys
2. **Configuration Updates**: When this repository is updated, Flux applies the new configuration
3. **Automatic Reconciliation**: Flux continuously ensures deployed state matches desired state

## 🚀 Usage Instructions

### Making Changes

#### Option 1: Update Chart Repository
Update values in `sample-app-helm-chart/charts/sample-app/values.yaml`:
```yaml
image:
  tag: "1.22"  # Change image version
replicaCount: 3  # Change replica count
```

#### Option 2: Override via HelmRelease
Update `helmrelease/sample-app-helmrelease.yaml`:
```yaml
spec:
  values:
    image:
      tag: "1.22"
    replicaCount: 3
    resources:
      limits:
        memory: "512Mi"
```

## 🔍 Monitoring & Troubleshooting

### Check Flux Status
```bash
# Check GitRepository status
kubectl get gitrepository -n flux-system

# Check HelmRelease status
kubectl get helmrelease -n sample-app

# View Flux logs
kubectl logs -n flux-system deployment/source-controller
kubectl logs -n flux-system deployment/helm-controller
```

## 🎯 Benefits of This Approach

### Separation of Concerns
- **Infrastructure**: Managed by platform team
- **Application Code**: Managed by development team  
- **Deployment Config**: Managed by DevOps/SRE team

### GitOps Advantages
- **Declarative**: Desired state defined in Git
- **Auditable**: All changes tracked in Git history
- **Recoverable**: Easy rollback via Git revert
- **Secure**: No direct cluster access needed

## 📚 Related Repositories

- **fluxcd-gitops**: Infrastructure and Flux bootstrap
- **sample-app-helm-chart**: Application Helm charts and templates

## 🔗 Useful Commands

```bash
# Watch Flux reconciliation
flux get sources git
flux get helmreleases

# Force reconciliation
flux reconcile source git sample-app-helm-chart
flux reconcile helmrelease sample-app2

# Suspend/resume reconciliation
flux suspend helmrelease sample-app2
flux resume helmrelease sample-app2
```
