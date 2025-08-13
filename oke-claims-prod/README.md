# Production OKE Cluster Resources

This directory contains the production version of OKE cluster resources that are separate from the test resources to avoid conflicts.

## Files

- `claim-okecluster-prod.yaml` - Production OKE cluster claim

## Key Differences from Test Resources

### Network Configuration
- **VCN CIDR**: `10.1.0.0/16` (vs `10.0.0.0/16` in test)
- **Cluster Subnet**: `10.1.1.0/24` (vs `10.0.1.0/24` in test)
- **Node Subnet**: `10.1.2.0/24` (vs `10.0.2.0/24` in test)
- **Service Subnet**: `10.1.3.0/24` (vs `10.0.3.0/24` in test)

### Cluster Configuration
- **Cluster Name**: `prodcluster` (vs `testcluster` in test)
- **Cluster Type**: `ENHANCED_CLUSTER` (vs `BASIC_CLUSTER` in test)
- **Pods CIDR**: `10.245.0.0/16` (vs `10.244.0.0/16` in test)
- **Services CIDR**: `10.97.0.0/16` (vs `10.96.0.0/16` in test)

### Node Pool Configuration
- **Node Pool Name**: `prodworkernodes` (vs `workernodes` in test)
- **Node Count**: 3 (vs 1 in test)
- **Memory**: 32 GB (vs 16 GB in test)
- **OCPUs**: 4 (vs 2 in test)

### Resource Names
- **Claim Name**: `prod-oke-cluster` (vs `test-oke-cluster` in test)
- **Network Prefix**: `prod` (vs `test` in test)

### Tags
- **Environment**: `production` (vs `development` in test)

## Usage

To deploy the production OKE cluster:

1. Apply the XRD: `kubectl apply -f ../oke-platform-prod/xrd-okecluster-prod.yaml`
2. Apply the Composition: `kubectl apply -f ../oke-platform-prod/composition-okecluster-prod.yaml`
3. Apply this claim: `kubectl apply -f claim-okecluster-prod.yaml`

## Important Notes

- These resources use different CIDR blocks to avoid conflicts with test resources
- Production resources are configured for higher availability and performance
- Enhanced cluster type provides additional features compared to basic cluster
- Increased node count and resources for production workloads
- **OIDC Multi-Issuer Support**: The cluster is configured with OIDC multi-issuer support for enhanced authentication

## OIDC Configuration

The production cluster supports OIDC multi-issuer authentication using configuration files. This approach allows you to configure multiple OIDC providers through a base64-encoded configuration file.

### Configuration Steps

1. **Create your OIDC configuration file** with the required OIDC provider settings
2. **Base64 encode the configuration file**:
   ```bash
   base64 -i your-oidc-config.yaml
   ```
3. **Update the claim** with the base64-encoded configuration:
   ```yaml
   cluster:
     configurationFile: "base64-encoded-config-content"
   ```

### Example OIDC Configuration File

```yaml
apiVersion: v1
kind: Config
clusters:
- name: oke-cluster
  cluster:
    server: https://your-cluster-endpoint
contexts:
- name: oke-context
  context:
    cluster: oke-cluster
    user: oidc-user
current-context: oke-context
users:
- name: oidc-user
  user:
    auth-provider:
      name: oidc
      config:
        client-id: your-client-id
        client-secret: your-client-secret
        id-token: your-id-token
        idp-issuer-url: https://your-oidc-provider.com
        refresh-token: your-refresh-token
```

### Supported OIDC Providers
- **Okta**: Enterprise identity provider
- **Azure AD**: Microsoft identity platform
- **Google Workspace**: Google identity
- **Custom OIDC**: Any OIDC-compliant provider

### OIDC Provider Setup

#### Okta Setup
1. Create an OAuth 2.0 application in Okta
2. Configure redirect URIs: `https://your-cluster-endpoint/oauth2/callback`
3. Note the Client ID and Client Secret
4. Update the configuration file with Okta details

#### Azure AD Setup
1. Register an application in Azure AD
2. Configure redirect URIs: `https://your-cluster-endpoint/oauth2/callback`
3. Note the Client ID and Client Secret
4. Update the configuration file with Azure AD details

### Security Considerations
- Store client secrets securely (consider using Kubernetes secrets)
- Use HTTPS for all OIDC endpoints
- Configure appropriate scopes and claims
- Regularly rotate client secrets
- Keep configuration files secure and restrict access 