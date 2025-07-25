# Crossplane OCI Provider with ArgoCD GitOps

A proof-of-concept for managing Oracle Cloud Infrastructure (OCI) resources using Crossplane, ArgoCD, and Composite Resource Definitions (XRDs) on Oracle Kubernetes Engine (OKE).

## 🎯 What This POC Demonstrates

- **Infrastructure as Code**: Declarative OCI resource management through Kubernetes APIs
- **GitOps Deployment**: ArgoCD-driven continuous delivery for infrastructure
- **XRD Abstraction**: Simplified APIs for complex infrastructure (complete OKE clusters)
- **Security-First**: Manual secret deployment keeping credentials out of Git.

## 📊 Architecture

```
┌─────────────────────────────────────┐
│         ArgoCD Applications         │  Wave 1: Crossplane Core
├─────────────────────────────────────┤  Wave 2: OCI Provider
│    Crossplane XRDs & Compositions   │  Wave 3: Platform APIs
├─────────────────────────────────────┤  Wave 4: User Claims
│       Crossplane Core v1.18.0       │
├─────────────────────────────────────┤
│  OCI Provider (Terraform-based)     │  ⚠️ CPU: 2000m minimum!
├─────────────────────────────────────┤
│   Oracle Cloud Infrastructure       │
└─────────────────────────────────────┘
```

## 🚀 Quick Start

### Prerequisites

- OKE cluster with ArgoCD installed
- kubectl configured for cluster access
- OCI API key and OCIR auth token
- Basic understanding of Kubernetes and GitOps

### Step 1: Deploy Secrets (Manual - Keep Out of Git!)

```bash
# Create namespace
kubectl create namespace crossplane-system

# OCI API credentials
kubectl create secret generic oci-creds \
  --namespace=crossplane-system \
  --from-literal=credentials='{
    "tenancy_ocid": "ocid1.tenancy.oc1..aaaaa...",
    "user_ocid": "ocid1.user.oc1..aaaaa...",
    "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----",
    "fingerprint": "aa:bb:cc:dd:ee:ff:00:11:22:33:44:55:66:77:88:99",
    "region": "us-ashburn-1",
    "auth": "ApiKey"
  }'

# OCIR pull secret
kubectl create secret docker-registry ocir-secret \
  --namespace=crossplane-system \
  --docker-server=iad.ocir.io \
  --docker-username=YOUR_TENANCY/YOUR_USERNAME \
  --docker-password=YOUR_AUTH_TOKEN
```

### Step 2: Deploy Infrastructure Components

```bash
# 1. Deploy Crossplane
kubectl apply -f argocd/applications/crossplane.yaml

# 2. Wait for readiness
kubectl wait --for=condition=ready pod -l app=crossplane \
  -n crossplane-system --timeout=300s

# 3. Deploy OCI Provider
kubectl apply -f argocd/applications/crossplane-provider-oci.yaml

# 4. Verify deployment
kubectl get providers
kubectl get providerconfig.oci
```

### Step 3: Test with Simple Resources

```bash
# Create a storage bucket
kubectl apply -f examples/bucket.yaml

# Check status
kubectl get bucket
kubectl describe bucket example-bucket
```

### Step 4: Deploy OKE Cluster (Advanced)

```bash
# Deploy XRD platform
kubectl apply -f oke-platform/xrd-okecluster.yaml
kubectl apply -f oke-platform/composition-okecluster.yaml

# Create cluster (update claim-okecluster.yaml first!)
kubectl apply -f oke-claims/claim-okecluster.yaml

# Monitor progress
kubectl get okeclusterclaim
kubectl get vcn,subnet,cluster,nodepool
```

## ⚙️ Critical Configuration

### Provider Resources - MUST INCREASE

The default 500m CPU is **insufficient**. You MUST increase resources:

```yaml
# In crossplane/providers/oci-provider.yaml
resources:
  limits:
    cpu: 2000m      # Minimum for stable operation
    memory: 2Gi
  requests:
    cpu: 1000m
    memory: 1Gi
```

**Why?** The Terraform-based provider forks processes for each reconciliation, causing CPU exhaustion at default limits.

### OCI Naming Requirements

OCI rejects DNS labels with hyphens. Use only alphanumeric names:

- ✅ `prodcluster`, `workernodes`, `prod`
- ❌ `prod-cluster`, `worker-nodes`, `prod-env`

### ArgoCD Sync Configuration

Add this to applications deploying XRD resources:

```yaml
syncPolicy:
  syncOptions:
    - SkipDryRunOnMissingResource=true  # Critical for CRDs
```

## 🛠️ Troubleshooting Guide

### Provider Issues

**Symptoms**: GRPC errors, "plugin did not respond", timeouts

```bash
# Check CPU usage (should be well below limit)
kubectl top pod -n crossplane-system | grep provider-oci

# View logs
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=oci-provider --tail=100

# Restart provider if needed
kubectl delete pod -n crossplane-system -l pkg.crossplane.io/provider=oci-provider
```

### Resource Stuck Creating

**Symptom**: Resource shows "Creating" indefinitely

```bash
# Check if resource exists in OCI Console
# If yes, annotate with the OCID:
kubectl annotate <resource> crossplane.io/external-name=ocid1.xxx... --overwrite

# Force reconciliation
kubectl annotate <resource> crossplane.io/reconcile-timestamp="$(date -u)" --overwrite
```

### ArgoCD Sync Failures

**Symptom**: "The Kubernetes API could not find <CRD>"

**Solution**: Ensure proper sync waves and `SkipDryRunOnMissingResource=true`

### Authentication Errors

**401 Errors**: Check secret format - use standard API keys, not session tokens

**403 Errors**: Verify OCIR credentials and namespace format

### Common kubectl Commands

```bash
# Get all Crossplane managed resources
kubectl get managed

# Check specific resource details
kubectl describe <resource-type> <name>

# View events
kubectl get events --sort-by='.lastTimestamp' | grep <resource>

# Check provider configuration
kubectl describe providerconfig.oci default
```

## 📁 Repository Structure

```
crossplane-gitops/
├── argocd/applications/         # ArgoCD app definitions
├── crossplane/providers/        # Provider configuration
├── oke-platform/                # XRD and Composition for OKE
├── oke-claims/                  # Example cluster claims
└── examples/                    # Simple resource examples
```

## 🎓 Key Lessons from This POC

### What Worked Well

- GitOps approach provides good auditability
- XRDs successfully abstract infrastructure complexity
- Manual secrets keep credentials secure

### Major Challenges Encountered

1. **Provider CPU Limits** - Default 500m causes failures; 2000m minimum required
2. **ArgoCD CRD Timing** - Requires careful sync wave configuration
3. **OKE Networking** - Enhanced clusters need 4 subnets; Basic clusters simpler
4. **DNS Naming** - No hyphens allowed in OCI resource names
5. **SSH Key for Nodess** - Currently require plain text in claims (security concern)

## 🏗️ XRD Pattern Example

The included OKE XRD creates a complete cluster environment:

```yaml
apiVersion: platform.example.com/v1alpha1
kind: OKEClusterClaim
metadata:
  name: production
spec:
  parameters:
    compartmentId: "ocid1.compartment.oc1..aaaaa..."
    cluster:
      name: prod              # No hyphens!
      kubernetesVersion: v1.33.1
      type: BASIC_CLUSTER     # Start with Basic
    nodePool:
      nodeCount: 3
      nodeShape: VM.Standard.E4.Flex
      sshPublicKey: "ssh-rsa AAAAB3..."  # Plain text required
```

This creates: VCN, subnets, gateways, route tables, security lists, OKE cluster, and node pool.

## ⚠️ Important Notes

### Security Considerations

- Never commit secrets to Git
- SSH keys in claims are visible (production concern)
- Rotate credentials regularly
- Delete secrets after POC: `kubectl delete secret oci-creds ocir-secret -n crossplane-system`

### Known Limitations

- Terraform providers are CPU-intensive at scale
- Some manual intervention may be needed for stuck resources
- Provider may lose track of resources during failures
- ArgoCD health checks need custom configuration for XRDs

## 🧹 Cleanup

```bash
# Delete claims first
kubectl delete -f oke-claims/
kubectl delete -f examples/

# Remove ArgoCD apps
kubectl delete -f argocd/applications/

# Clean up secrets
kubectl delete secret oci-creds ocir-secret -n crossplane-system
```

## 📚 Additional Resources

- [Crossplane Docs](https://docs.crossplane.io)
- [OCI Provider Issues](https://github.com/oracle-samples/crossplane-provider-oci/issues)
- [ArgoCD Sync Options](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [OKE Documentation](https://docs.oracle.com/en-us/iaas/Content/ContEng/home.htm)
