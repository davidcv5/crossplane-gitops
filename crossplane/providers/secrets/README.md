# Manual Secret Deployment for POC

This directory contains **instructions** for manually deploying secrets. **No actual secrets are stored in Git.**

## Why Manual Deployment?

For POC environments, this approach provides:

- ✅ **Security**: No credentials in Git repository
- ✅ **Simplicity**: No external secret management setup required
- ✅ **Speed**: Quick deployment for testing
- ✅ **Flexibility**: Easy to update secrets during development

## Required Secrets

### 1. OCI API Credentials (`oci-creds`)

Deploy with your actual OCI API credentials:

```bash
kubectl create secret generic oci-creds \
  --namespace=crossplane-system \
  --from-literal=credentials='{
    "tenancy_ocid": "ocid1.tenancy.oc1..aaaaaaaaef7jcdwjzimictktlzdajwcruyywj4ejg5evqjmzaxdawkbvwcva",
    "user_ocid": "ocid1.user.oc1..aaaaaaaalwtjpgfvdjpfipuof4rb7pv7gvvzep4hdaq3i5lt2vfdznkr7inq",
    "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQ...\n-----END PRIVATE KEY-----\n",
    "fingerprint": "08:6a:dc:0e:f4:99:4b:a6:9c:ea:56:20:95:45:ef:a8",
    "region": "us-ashburn-1"
  }'
```

### 2. OCIR Pull Secret (`ocir-secret`)

Deploy with your OCIR credentials:

```bash
kubectl create secret docker-registry ocir-secret \
  --namespace=crossplane-system \
  --docker-server=iad.ocir.io \
  --docker-username=idwsb3nvxazj/<your-oci-username> \
  --docker-password=<your-oci-auth-token>
```

## Deployment Script

For convenience, create a local deployment script (keep this outside Git):

```bash
#!/bin/bash
# deploy-secrets.sh - Keep this file LOCAL, do not commit to Git!

set -e

echo "Deploying Crossplane secrets..."

# Ensure namespace exists
kubectl create namespace crossplane-system --dry-run=client -o yaml | kubectl apply -f -

# Deploy OCI API credentials
kubectl create secret generic oci-creds \
  --namespace=crossplane-system \
  --from-literal=credentials='{
    "tenancy_ocid": "REPLACE_WITH_YOUR_TENANCY_OCID",
    "user_ocid": "REPLACE_WITH_YOUR_USER_OCID",
    "private_key": "REPLACE_WITH_YOUR_PRIVATE_KEY",
    "fingerprint": "REPLACE_WITH_YOUR_FINGERPRINT",
    "region": "REPLACE_WITH_YOUR_REGION"
  }' \
  --dry-run=client -o yaml | kubectl apply -f -

# Deploy OCIR credentials
kubectl create secret docker-registry ocir-secret \
  --namespace=crossplane-system \
  --docker-server=iad.ocir.io \
  --docker-username=idwsb3nvxazj/REPLACE_WITH_YOUR_USERNAME \
  --docker-password=REPLACE_WITH_YOUR_AUTH_TOKEN \
  --dry-run=client -o yaml | kubectl apply -f -

echo "✅ Secrets deployed successfully!"
echo "You can now deploy the GitOps applications."
```

## Security Notes

- 🔒 **Never commit** the deployment script with real credentials
- 🔒 **Store credentials** in a password manager or secure notes
- 🔒 **Rotate secrets** regularly
- 🔒 **Delete secrets** when POC is complete: `kubectl delete secret oci-creds ocir-secret -n crossplane-system`

## Verification

After deploying secrets, verify they exist:

```bash
kubectl get secrets -n crossplane-system
kubectl describe secret oci-creds -n crossplane-system
kubectl describe secret ocir-secret -n crossplane-system
```
