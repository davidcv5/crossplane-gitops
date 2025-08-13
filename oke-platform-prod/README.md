# Production OKE Platform Resources

This directory contains the production version of OKE platform resources (XRD and Composition) that are separate from the test resources to avoid conflicts.

## Files

- `xrd-okecluster-prod.yaml` - Production OKE cluster XRD (Composite Resource Definition) with unique API group
- `composition-okecluster-prod.yaml` - Production OKE cluster composition

## Key Differences from Test Resources

### XRD Configuration
- **Default VCN CIDR**: `10.1.0.0/16` (vs `10.0.0.0/16` in test)
- **Default Cluster Subnet**: `10.1.1.0/24` (vs `10.0.1.0/24` in test)
- **Default Node Subnet**: `10.1.2.0/24` (vs `10.0.2.0/24` in test)
- **Default Service Subnet**: `10.1.3.0/24` (vs `10.0.3.0/24` in test)
- **Default Pods CIDR**: `10.245.0.0/16` (vs `10.244.0.0/16` in test)
- **Default Services CIDR**: `10.97.0.0/16` (vs `10.96.0.0/16` in test)
- **Default Node Count**: 3 (vs 1 in test)
- **Default Memory**: 32 GB (vs 16 GB in test)
- **Default OCPUs**: 4 (vs 2 in test)
- **Default Cluster Type**: `ENHANCED_CLUSTER` (vs `BASIC_CLUSTER` in test)

### Composition Configuration
- **Composition Name**: `okecluster-prod` (vs `okecluster` in test)
- **DNS Label**: `okeclusterprod` (vs `okecluster` in test)
- **Environment Label**: `production` (vs none in test)
- **Base CIDR Blocks**: Updated to use production ranges
- **Security List Rules**: Updated to use production CIDR ranges
- **Node Pool Configuration**: Production-ready defaults

## Usage

To deploy the production OKE platform:

1. Apply the XRD: `kubectl apply -f xrd-okecluster-prod.yaml`
2. Apply the Composition: `kubectl apply -f composition-okecluster-prod.yaml`
3. Apply the claim from `oke-claims-prod/claim-okecluster-prod.yaml`

## Important Notes

- The XRD uses a unique API group (`oke-prod.crossplane.io`) to avoid conflicts with test resources
- The composition creates resources with production naming conventions
- All network resources use different CIDR blocks to avoid conflicts with test resources
- Enhanced cluster type provides additional features for production workloads
- Production resources are configured for higher availability and performance
- **Route Table Isolation**: Each composition creates its own route tables, preventing cross-references between test and production resources 