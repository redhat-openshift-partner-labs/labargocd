# Troubleshooting Guide

This guide consolidates troubleshooting information for cluster provisioning issues.

## Quick Diagnosis Commands

```bash
# Check ArgoCD Application status
oc get application -n openshift-gitops

# Check ClusterDeployment status
oc get clusterdeployment -n <cluster-namespace>

# View provision job logs
oc logs -n <namespace> job/<cluster>-provision

# Check Hive controller logs
oc logs -n hive deployment/hive-controllers -c hive-controllers --tail=100
```

## ArgoCD Issues

### Application stuck in "Syncing"

**Symptoms**: Application shows Syncing status indefinitely

**Diagnosis**:
```bash
oc describe application <cluster-name> -n openshift-gitops
oc get application <cluster-name> -n openshift-gitops -o jsonpath='{.status.operationState}'
```

**Solutions**:
- Cancel stuck operation: `oc patch application <cluster-name> -n openshift-gitops --type merge -p '{"operation":null}'`
- Check ArgoCD logs: `oc logs -n openshift-gitops deployment/openshift-gitops-application-controller`

### "Application resource not permitted in project"

**Cause**: ArgoCD project lacks permission for Application resources (needed for app-of-apps).

**Solution**: Add Application to namespaceResourceWhitelist in ArgoCD project:
```yaml
namespaceResourceWhitelist:
  - group: argoproj.io
    kind: Application
```

### "Hive/ACM resources not permitted in project"

**Cause**: ArgoCD project doesn't allow Hive and ACM CRDs.

**Solution**: Ensure `bootstrap/argocd-project.yaml` includes all required resource types in both `clusterResourceWhitelist` and `namespaceResourceWhitelist`.

### Namespace keeps getting deleted

**Cause**: ArgoCD sync failures trigger namespace deletion/recreation cycles.

**Solution**:
1. Cancel sync: `oc patch application <cluster-name> -n openshift-gitops --type merge -p '{"operation":null}'`
2. Disable auto-sync: `oc patch application <cluster-name> -n openshift-gitops --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'`
3. Create namespace: `oc create namespace <cluster-name>`
4. Recreate secrets (see [Secrets Guide](../secrets/README.md))
5. Manual apply: `oc kustomize clusters/<cluster-name> | oc apply -f -`

## Hive Provisioning Issues

### Cluster stuck in "Provisioning"

**Symptoms**: ClusterDeployment remains in Provisioning state for >60 minutes

**Diagnosis**:
```bash
oc get clusterdeployment <cluster> -n <namespace> -o yaml | grep -A 10 conditions
oc logs -n <namespace> job/<cluster>-provision
```

**Common Causes**:
- Insufficient AWS quotas
- Invalid AWS credentials
- DNS zone not found
- Network connectivity issues

### "ssh: no key found" or invalid SSH key

**Cause**: install-config contains placeholder SSH key instead of letting Hive inject it.

**Solution**: Remove `sshKey` field from install-config.yaml. Hive injects it automatically from ClusterDeployment's `sshPrivateKeySecretRef`.

### "ManagedCluster validation webhook denied: url is invalid"

**Cause**: ManagedCluster has empty `managedClusterClientConfigs`.

**Solution**: Remove `managedClusterClientConfigs` from ManagedCluster spec. ACM populates it automatically after provisioning.

### "Hive controller panic: nil pointer dereference" in retrofitMetadataJSON

**Cause**: ClusterDeployment has `clusterMetadata` fields set before provisioning.

**Solution**: Remove `clusterMetadata` section from ClusterDeployment template. Hive populates these automatically.

### "ClusterImageSet not found"

**Cause**: Referenced OpenShift version doesn't have a ClusterImageSet.

**Solution**:
```bash
# Check available versions
oc get clusterimageset

# Update configuration to use available version
```

## AWS Issues

### Quota exceeded

**Symptoms**: Error about EC2 instance limits

**Diagnosis**:
```bash
aws service-quotas list-service-quotas --service-code ec2 --region us-east-1
```

**Solution**:
```bash
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --desired-value 50 \
  --region us-east-1
```

### DNS errors

**Symptoms**: Cannot resolve cluster API endpoint

**Diagnosis**:
```bash
aws route53 list-hosted-zones-by-name --dns-name <base-domain>
aws route53 get-hosted-zone --id <zone-id>
```

**Solution**: Ensure Route53 hosted zone exists and matches `baseDomain`.

### Network CIDR conflicts

**Symptoms**: Cluster provision fails with networking errors

**Solution**: Ensure CIDRs don't conflict with existing VPC or on-premises networks.

## Secret Issues

### "secrets not found" during deprovisioning

**Cause**: Secrets were deleted before uninstall job could read them.

**Symptoms**:
```
error="secrets \"<cluster>-metadata-json\" not found"
```

**Prevention**: Ensure PostSync job adds finalizers. Check:
```bash
oc get secret aws-credentials -n <namespace> -o jsonpath='{.metadata.finalizers}'
oc get secret <cluster>-metadata-json -n <namespace> -o jsonpath='{.metadata.finalizers}'
```

### Secrets deleted by ArgoCD selfHeal

**Cause**: ArgoCD selfHeal deletes Hive-created secrets that aren't in Git.

**Solution**: Disable selfHeal in ArgoCD Application:
```yaml
syncPolicy:
  automated:
    selfHeal: false  # CRITICAL
```

## ACM Import Issues

### Cluster not importing to ACM

**Diagnosis**:
```bash
oc get managedcluster
oc get managedcluster <cluster> -o yaml
oc get secret -n <cluster> <cluster>-import
```

**On provisioned cluster**:
```bash
export KUBECONFIG=<cluster-kubeconfig>
oc get klusterlet -A
```

## Manual Intervention Workflow

When automated sync fails repeatedly:

```bash
# 1. Disable automated sync
oc patch application <cluster> -n openshift-gitops \
  --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 2. Cancel running operations
oc patch application <cluster> -n openshift-gitops \
  --type merge -p '{"operation":null}'

# 3. Ensure namespace exists
oc create namespace <cluster>

# 4. Create/verify secrets
oc get secrets -n <cluster>

# 5. Manually apply resources
oc kustomize clusters/<cluster> | oc apply -f -

# 6. Monitor provision pod
oc get pods -n <cluster> -w
```

## Image Pull Errors

### CronJob or PostSync Job fails with ImagePullBackOff

**Symptoms**: Job pods show `ImagePullBackOff` or `ErrImagePull`

**Cause**: Missing pull secret for `registry.redhat.io/openshift4/ose-cli:v4.14`

**For CronJob** (runs in `openshift-gitops`):
```bash
# Create pull secret
oc create secret generic deprovision-cleanup-pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n openshift-gitops

# Update cronjob to reference it (edit imagePullSecrets in the manifest)
```

**For PostSync Job** (runs in cluster namespace):
```bash
# Verify pull-secret exists in cluster namespace
oc get secret pull-secret -n <cluster-namespace>

# If missing, create it
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <cluster-namespace>
```

## Cleanup After Failed Provision

If a cluster fails to provision and leaves orphaned AWS resources:

```bash
# List EC2 instances
aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/<cluster>,Values=owned" \
  --region <region>

# Check S3 buckets
aws s3 ls | grep <cluster>

# Use openshift-install to clean up
openshift-install destroy cluster --dir=<cluster-dir> --log-level=debug
```
