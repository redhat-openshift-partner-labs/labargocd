# Crossplane AWS IAM Resources

This folder contains Crossplane manifests to provision AWS IAM credentials for an OpenShift cluster installer user.

## Resource Inventory

| File | Kind | Name | Purpose |
|------|------|------|---------|
| `provider.yaml` | `Provider` + `ProviderConfig` | `provider-aws-iam` / `default` | Installs the Upbound AWS IAM provider (`v1.7.0` - compatible with Crossplane 2.2.0) and binds it to the `aws-credentials` secret in `crossplane-system` namespace |
| `iam-user.yaml` | `User` | `ocp-installer` | Creates an AWS IAM user for the OpenShift installer |
| `iam-policy.yaml` | `Policy` | `OpenShift4InstallerPolicy` | IAM policy with EC2, ELB, autoscaling, IAM, S3, Route53, and service-quotas permissions required for OCP 4.20 installation |
| `iam-attachment.yaml` | `UserPolicyAttachment` | `ocp-installer-policy-attachment` | Attaches `OpenShift4InstallerPolicy` to the `ocp-installer` user |
| `iam-access-key.yaml` | `AccessKey` | `ocp-installer-access-key` | Generates an AWS access key and writes the credentials to the `aws-credentials-raw` secret with keys: `username` (access key ID) and `password` (secret access key) |
| `credentials-transformer-job.yaml` | `Job` + RBAC | `aws-credentials-transformer` | Transforms Crossplane credentials format to Hive-compatible format with keys: `aws_access_key_id` and `aws_secret_access_key` |

## Prerequisites

### 1. Install Crossplane on OpenShift

Crossplane must be installed on your cluster. For **OpenShift**, use these commands to install with proper security contexts:

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm upgrade --install crossplane \
  crossplane-stable/crossplane \
  --namespace crossplane-system \
  --version 2.2.0 \
  --set args='{--enable-usages}' \
  --set securityContextCrossplane.runAsUser=null \
  --set securityContextCrossplane.runAsGroup=null \
  --set securityContextRBACManager.runAsUser=null \
  --set securityContextRBACManager.runAsGroup=null \
  --create-namespace
```

**Note:** The `null` security context settings allow OpenShift to assign UIDs dynamically, which is required for OpenShift's security model.

See the [official Crossplane installation docs](https://docs.crossplane.io/latest/get-started/install/) for more options.

### 2. Grant Security Context Constraints (OpenShift only)

After installing the provider, grant the necessary SCCs to allow provider pods to run on OpenShift:

```bash
# Wait for provider to be installed
kubectl wait provider provider-aws-iam --for=condition=Installed --timeout=120s

# Get the provider revision service account name
PROVIDER_SA=$(kubectl get deployment -n crossplane-system -o name | grep provider-aws-iam | sed 's/deployment.apps\///')

# Grant privileged SCC
oc adm policy add-scc-to-user privileged -z ${PROVIDER_SA} -n crossplane-system

# Restart the provider deployment
kubectl rollout restart deployment ${PROVIDER_SA} -n crossplane-system
```

Repeat for the family-aws provider if using Upbound provider family.

### 3. Create the AWS credentials secret (one-time setup)

This secret is used by the `ProviderConfig` to authenticate with AWS:

```bash
kubectl create secret generic aws-credentials \
  -n crossplane-system \
  --from-literal=credentials="[default]
aws_access_key_id = YOUR_KEY
aws_secret_access_key = YOUR_SECRET"
```

## Usage

### 1. Apply the provider and wait for it to become healthy

`provider.yaml` contains both the `Provider` package install and the `ProviderConfig` that binds it to the `aws-credentials` secret. The provider must be healthy before the IAM resources can be applied:

```bash
kubectl apply -f crossplane/provider.yaml
kubectl wait provider provider-aws-iam --for=condition=Healthy --timeout=120s
```

### 2. Apply the IAM resources

```bash
kubectl apply -f crossplane/iam-user.yaml \
              -f crossplane/iam-policy.yaml \
              -f crossplane/iam-attachment.yaml \
              -f crossplane/iam-access-key.yaml
```

### 3. Verify the resources were reconciled

Check that all resources are synced and ready:

```bash
kubectl get user,policy,userpolicyattachment,accesskey \
  -l app.kubernetes.io/part-of=ocp-installer
```

All resources should show `READY: True` and `SYNCED: True`.

## Retrieving the Generated Credentials

Once the `AccessKey` resource is ready, Crossplane writes the generated AWS credentials to a secret:

```bash
# Crossplane-generated secret (raw format)
kubectl get secret aws-credentials-raw -n <namespace> \
  -o jsonpath='{.data}' | jq 'map_values(@base64d)'
```

The Crossplane secret contains:
- `username` - AWS access key ID
- `password` - AWS secret access key

### Credentials Transformer

The `credentials-transformer-job.yaml` automatically converts the Crossplane credentials format to Hive-compatible format:

**Input (Crossplane format):** `aws-credentials-raw` secret with `username` and `password` keys

**Output (Hive format):** `aws-credentials` secret with `aws_access_key_id` and `aws_secret_access_key` keys

```bash
# Hive-compatible credentials secret
kubectl get secret aws-credentials -n <namespace> \
  -o jsonpath='{.data}' | jq 'map_values(@base64d)'
```

## Cleanup

To delete all IAM resources, apply in reverse order to respect dependencies (access key and attachment before policy and user):

```bash
kubectl delete -f crossplane/iam-access-key.yaml \
               -f crossplane/iam-attachment.yaml \
               -f crossplane/iam-policy.yaml \
               -f crossplane/iam-user.yaml
```

Crossplane will delete the corresponding AWS resources before removing the Kubernetes objects. To also remove the provider:

```bash
kubectl delete -f crossplane/provider.yaml
```

## Troubleshooting

### Provider pods not starting on OpenShift

If provider pods fail with security context errors:

```
Error: pods "provider-aws-iam-xxx" is forbidden: unable to validate against any security context constraint
```

**Solution:** Grant privileged SCC to the provider service account (see step 2 in Prerequisites).

### ProviderConfig not found errors

If IAM resources show:
```
cannot get referenced ProviderConfig: "default": ProviderConfig.aws.upbound.io "default" not found
```

**Solution:** Ensure the `ProviderConfig` resource exists cluster-wide:
```bash
kubectl get providerconfig
```

If missing, apply the provider configuration from the bootstrap directory.

### Credentials transformer fails

If the transformer job shows errors about missing keys:

**Solution:** The job expects Crossplane secret keys named `username` and `password`. Verify the AccessKey resource is creating the secret correctly:

```bash
kubectl get secret aws-credentials-raw -o yaml
```
