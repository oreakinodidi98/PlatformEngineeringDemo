# Demo 2: Cloud-Native Self-Service Setup Guide

## Overview

This guide walks through setting up a **cloud-native self-service platform** using:
- **Crossplane**: Cloud-native infrastructure provisioning
- **ArgoCD**: GitOps-based configuration management  
- **Kubernetes**: Management and downstream application clusters

**Result**: One simple claim provisions a complete AKS cluster with ArgoCD pre-installed.

---

## Prerequisites

✅ **Management Cluster Ready**
- `mgmt-aks` running with Crossplane v1.13+ installed
- Helm and Kubernetes providers installed
- Default ProviderConfig configured for Azure

✅ **Providers Installed**
```bash
kubectl get providers.pkg.crossplane.io
# Should show: provider-azure-management, provider-azure-containerservice, 
#              provider-helm, provider-kubernetes
```

✅ **Function Installed**
```bash
kubectl get functions.pkg.crossplane.io function-patch-and-transform
# Should show: HEALTHY=True
```

---

## File Structure

```
mgmtCluster/bootstrap/control-plane/compositions/
├── staging-cluster-definitions.yaml       ✅ XRD (defines the API)
├── function-patch-and-transform.yaml       ✅ Function (enables provisioning)
└── staging-cluster-comp-final.yaml         ✅ Composition (implements resources)

downstreamInfra/team01/
├── team1-apps.yaml                         ✅ Claim (user request)
└── workloads/ (future: app manifests)

.envrc                                       ✅ Environment variables
```

---

## Step-by-Step Deployment

### Step 1: Install Function

The composition function handles resource provisioning in Pipeline mode.

```bash
kubectl apply -f .\mgmtCluster\bootstrap\control-plane\compositions\function-patch-and-transform.yaml

# Wait for it to become healthy
kubectl get functions.pkg.crossplane.io function-patch-and-transform -w
```

**Status**: Wait until `HEALTHY=True` before proceeding.

### Step 2: Apply XRD (API Definition)

Defines the interface teams use to request clusters.

```bash
kubectl apply -f .\mgmtCluster\bootstrap\control-plane\compositions\staging-cluster-definitions.yaml

# Verify
kubectl get compositeresourcedefinitions
kubectl describe compositeresourcedefinition staging-aks.compute.example.com
```

**What It Does**: Validates that claims include required fields (clustername, teamname, location) and optional fields (repourl, repopath).

### Step 3: Apply Composition

Implements the actual resource provisioning logic.

```bash
kubectl apply -f .\mgmtCluster\bootstrap\control-plane\compositions\staging-cluster-comp-final.yaml

# Verify
kubectl get composition staging-aks
```

**What It Does**: When a claim is submitted, Crossplane uses this to create:
- ResourceGroup in Azure
- KubernetesCluster (AKS) in Azure
- ProviderConfigs to access the new cluster
- Argo Release (installs ArgoCD)
- Argo Application (connects Git repository)

### Step 4: Submit a Claim

Teams submit a simple request to provision a cluster.

```bash
kubectl apply -f .\downstreamInfra\team01\team1-apps.yaml

# Or manually:
kubectl apply -f - <<EOF
apiVersion: compute.example.com/v1alpha1
kind: staging-aks
metadata:
  name: my56app
spec:
  clustername: my56cluster
  teamname: team01
  location: EU
  repourl: https://github.com/oreakinodidi98/PlatformEngineeringDemo
  repopath: workloads/team01
EOF
```

**What This Triggers**: 
1. Crossplane validates inputs against XRD
2. Selects the composition
3. Provisions all 6 resources in sequence

---

## Monitoring Provisioning

### Watch the Claim

```bash
kubectl get staging-aks.compute.example.com/my56app -w
```

Status progression:
```
Synced: False (initial)  →  Synced: True (resources created)
Ready: False (Creating)  →  Ready: True  (all ready)
```

### Watch the Cluster

```bash
kubectl get kubernetescluster.containerservice.azure.upbound.io/my56cluster -w
```

Status progression:
```
SYNCED=True, READY=False  (provisioning in Azure - 5-15 min)
   ↓
SYNCED=True, READY=True   (cluster ready, ArgoCD installing)
```

### View All Resources

```bash
kubectl get managed -o wide
```

Expected output:
```
NAME                                                    SYNCED    READY    AGE
resourcegroup.azure.upbound.io/my56app-xxxxx           True      True     1m
kubernetescluster.containerservice.azure.upbound.io/my56cluster  True  False  5m
release.helm.crossplane.io/my56app-xxxxx               False            2m
object.kubernetes.crossplane.io/my56app-xxxxx          False            2m
```

---

## Expected Timeline

| Phase | Duration | What Happens |
|-------|----------|-------------|
| **Claim Submitted** | 0s | Claim validated, composition selected |
| **ResourceGroup Created** | ~10s | Azure resource group ready |
| **AKS Provisioning** | 5-15m | Azure creates cluster, installs add-ons |
| **ArgoCD Installation** | 2-5m | Helm release deploys ArgoCD to new cluster |
| **Application Sync** | 1-2m | ArgoCD Application syncs workloads from Git |
| **✅ Complete** | **15-25 min** | Cluster ready with applications deployed |

---

## Understanding the Architecture

### Single Repo, Two Zones Pattern

**Management Cluster Zone** (`mgmtCluster/bootstrap/`)
- Hosts Crossplane (infrastructure provisioning)
- Hosts ArgoCD (GitOps orchestration)
- Runs definitions (XRD, Composition)
- **Role**: Control plane for infrastructure

**Workload Zone** (`downstreamInfra/`)
- Contains infrastructure claims (staging-aks requests)
- Contains workload manifests (application configs)
- **Role**: User-facing place to request infrastructure and deploy apps

**Flow**:
1. Team writes a claim in `downstreamInfra/team01/`
2. Claim reaches management cluster via GitOps or `kubectl apply`
3. Crossplane provisions AKS cluster in Azure
4. Crossplane installs ArgoCD on new cluster
5. ArgoCD watches `workloads/team01/` in Git
6. Applications automatically deploy

### Patch-Based Configuration

The Composition uses **patches** to customize resources from claim values:

- `location: EU` → transforms to `Sweden Central` region
- `clustername: my56cluster` → creates cluster named `my56cluster-dns`
- Patches also concatenate values for secret names, provider names, etc.

This allows:
- ✅ Teams provide simple inputs
- ✅ Platform enforces standards
- ✅ No duplicate or conflicting configurations

---

## Troubleshooting

### Claim stuck in `Ready=False (Creating)`

**Cause**: AKS cluster still provisioning (normal for first 10-15 minutes)

**Solution**: Wait and monitor
```bash
kubectl describe kubernetescluster.containerservice.azure.upbound.io/my56cluster
```

### Release/Object showing `SYNCED=False`

**Cause**: Waiting for KubernetesCluster to be `READY=True`

**Solution**: This is expected. Once cluster is ready, these will automatically sync.

### "ProviderConfig not Ready" warning

**Cause**: ProviderConfigs don't follow standard Ready logic

**Solution**: This is expected. We set `readinessChecks: - type: None` to suppress false warnings. They work correctly even though they don't show as Ready.

### Schema validation error on composition

**Cause**: Crossplane version mismatch

**Solution**: Ensure Crossplane v1.13+
```bash
kubectl get deployment crossplane -n crossplane-system -o jsonpath='{.spec.template.spec.containers[0].image}'
```

---

## Cleanup

Delete the provisioned infrastructure:

```bash
# Delete the claim - cascades to all composed resources
kubectl delete staging-aks.compute.example.com/my56app

# Verify deletion (AKS deletion takes 5-10 minutes in Azure)
kubectl get managed -w
```

This removes:
- ✓ Azure ResourceGroup
- ✓ AKS Cluster
- ✓ ProviderConfigs
- ✓ ArgoCD Release
- ✓ Argo Application

---

## Next Steps

**Extend the Platform**:
1. Add more providers (databases, storage, networking)
2. Add Gatekeeper policies for resource governance
3. Add team RBAC for namespace isolation
4. Add pre-defined workload templates in the catalog
5. Connect to Backstage for developer portal

**Automate Claims**:
1. Create CI/CD pipeline to generate claims from templates
2. Add approval workflows before claim submission
3. Add cost estimation and quota validation

---

## Key Files Reference

| File | Purpose | Type |
|------|---------|------|
| `staging-cluster-definitions.yaml` | API schema | XRD |
| `function-patch-and-transform.yaml` | Resource provisioning engine | Function |
| `staging-cluster-comp-final.yaml` | Provisioning logic | Composition |
| `team1-apps.yaml` | Infrastructure request | Claim |
| `notes.md` | Full walkthrough | Documentation |

---

## Important Constraints & Notes

⚠️ **Crossplane Version**: v1.13+ required (Pipeline mode)

⚠️ **ProviderConfig readinessChecks**: Must be `- type: None` (they don't become Ready normally)

⚠️ **Kubeconfig Secret**: Created automatically in `crossplane-system` namespace when AKS cluster is provisioned

⚠️ **Azure Credentials**: Management cluster must have Azure permissions via identity or service principal

⚠️ **Cluster Provisioning Time**: AKS clusters take 5-15 minutes to provision in Azure (not instant)

---

## Resources

- [Crossplane Documentation](https://www.crossplane.io/)
- [function-patch-and-transform](https://marketplace.upbound.io/providers/crossplane-contrib/function-patch-and-transform)
- [ArgoCD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Crossplane Composition Guide](https://docs.crossplane.io/latest/concepts/composition/)
