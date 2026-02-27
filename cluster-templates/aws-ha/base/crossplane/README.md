# Crossplane AWS IAM Resources

This folder contains Crossplane manifests to provision AWS IAM credentials for an OpenShift cluster installer user.

## Resource Inventory

| File | Kind | Name | Purpose |
|------|------|------|---------|
| `provider.yaml` | `Provider` + `ProviderConfig` | `provider-aws-iam` / `default` | Installs the Upbound AWS IAM provider (`v1.20.0`) and binds it to the `aws-credentials` secret |
| `iam-user.yaml` | `User` | `ocp-installer` | Creates an AWS IAM user for the OpenShift installer |
| `iam-policy.yaml` | `Policy` | `OpenShift4InstallerPolicy` | IAM policy with EC2, ELB, autoscaling, IAM, S3, Route53, and service-quotas permissions required for OCP 4.20 installation |
| `iam-attachment.yaml` | `UserPolicyAttachment` | `ocp-installer-policy-attachment` | Attaches `OpenShift4InstallerPolicy` to the `ocp-installer` user |
| `iam-access-key.yaml` | `AccessKey` | `ocp-installer-access-key` | Generates an AWS access key and writes the credentials to the `ocp-installer-credentials` secret in the `default` namespace |

## Prerequisites

### 1. Install Crossplane

Crossplane must be installed on your cluster. The recommended way is via Helm:

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace
```

See the [official Crossplane installation docs](https://docs.crossplane.io/latest/get-started/install/) for more options.

### 2. Create the AWS credentials secret (one-time setup)

This secret is used by the `ProviderConfig` in `provider.yaml` to authenticate with AWS:

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

Once the `AccessKey` resource is ready, Crossplane writes the generated AWS credentials to a secret in the `default` namespace:

```bash
kubectl get secret ocp-installer-credentials -n default \
  -o jsonpath='{.data}' | jq 'map_values(@base64d)'
```

The secret contains the `username`, `id` (access key ID), and `secret` (secret access key) fields that can be used for OpenShift cluster installation.

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
