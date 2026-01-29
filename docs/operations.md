# Cluster Operations

This document covers day-2 operations for provisioned clusters.

## Monitoring Cluster Status

### Check ClusterDeployment

```bash
# Get status
oc get clusterdeployment -n <cluster-namespace>

# Watch for changes
oc get clusterdeployment <cluster> -n <namespace> -w

# Detailed status
oc describe clusterdeployment <cluster> -n <namespace>
```

**Status Indicators**:
- `PROVISIONSTATUS: Provisioning` - Cluster being created
- `PROVISIONSTATUS: Provisioned` - Cluster ready
- `INFRAID` populated - AWS resources created
- Provision pod `Running` - Installation in progress

### Monitor Provisioning

```bash
# List provision jobs
oc get jobs -n <namespace>

# View job logs
oc logs -n <namespace> job/<cluster>-provision -f

# Check pods
oc get pods -n <namespace> -l job-name=<cluster>-provision
```

## Scaling Workers

Update worker replicas in `kustomization.yaml`:

```yaml
- patch: |-
    apiVersion: hive.openshift.io/v1
    kind: MachinePool
    metadata:
      name: cluster-placeholder-worker
    spec:
      replicas: 6  # Scale to 6 workers
```

Commit and push. ArgoCD will sync the change.

### Enable Autoscaling

Replace fixed replicas with autoscaling:

```yaml
spec:
  autoscaling:
    minReplicas: 3
    maxReplicas: 10
  # Remove: replicas: 3
```

## Upgrading OpenShift

1. Check available ClusterImageSets:
   ```bash
   oc get clusterimageset
   ```

2. Update the imageSetRef in kustomization.yaml:
   ```yaml
   provisioning:
     imageSetRef:
       name: img4.15.0-x86-64-appsub
   ```

3. Commit and push

## Hibernation (Cost Savings)

### Hibernate a Cluster

```bash
oc patch clusterdeployment <cluster> -n <namespace> \
  --type merge -p '{"spec":{"powerState":"Hibernating"}}'
```

Hibernation stops EC2 instances but preserves EBS volumes.

### Resume a Cluster

```bash
oc patch clusterdeployment <cluster> -n <namespace> \
  --type merge -p '{"spec":{"powerState":"Running"}}'
```

## Accessing Provisioned Clusters

### Extract Kubeconfig

```bash
# Display kubeconfig
oc extract secret/<cluster>-admin-kubeconfig -n <namespace> --to=-

# Save to file
oc extract secret/<cluster>-admin-kubeconfig -n <namespace> --to=. --confirm

# Use it
export KUBECONFIG=./kubeconfig
oc get nodes
```

### Get Admin Password

```bash
oc extract secret/<cluster>-admin-password -n <namespace> --to=-
```

## Deprovisioning Clusters

**WARNING**: This permanently deletes the cluster and all data.

### Option 1: Remove from Git (Recommended)

1. Delete the cluster directory from Git
2. Commit and push
3. ArgoCD deletes the Application
4. Hive deprovisions AWS resources
5. CronJob cleans up finalizers

### Option 2: Direct Deletion

```bash
# Delete ArgoCD Application
oc delete application <cluster> -n openshift-gitops

# Or delete ClusterDeployment directly
oc delete clusterdeployment <cluster> -n <namespace>
```

### Preserve on Delete

To prevent accidental deletion, set `preserveOnDelete: true`:

```yaml
spec:
  preserveOnDelete: true
```

This prevents Hive from destroying AWS resources when ClusterDeployment is deleted.

## Manual Intervention

When ArgoCD sync fails or causes issues:

```bash
# 1. Disable automated sync
oc patch application <cluster> -n openshift-gitops \
  --type merge -p '{"spec":{"syncPolicy":{"automated":null}}}'

# 2. Cancel running operations
oc patch application <cluster> -n openshift-gitops \
  --type merge -p '{"operation":null}'

# 3. Ensure namespace exists
oc create namespace <cluster>

# 4. Verify/recreate secrets
oc get secrets -n <cluster>

# 5. Manually apply resources
oc kustomize clusters/<cluster> | oc apply -f -

# 6. Monitor provision
oc get pods -n <cluster> -w
```

## Secret Recreation

If namespace was deleted and secrets lost:

```bash
# AWS credentials
oc create secret generic aws-credentials \
  --from-literal=aws_access_key_id=<key> \
  --from-literal=aws_secret_access_key=<secret> \
  -n <cluster>

# Pull secret
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <cluster>

# SSH key
oc create secret generic <cluster>-ssh-key \
  --from-file=ssh-privatekey=<key-file> \
  --from-file=ssh-publickey=<key-file>.pub \
  --type=kubernetes.io/ssh-auth \
  -n <cluster>
```

## Verify AWS Cleanup

After deprovisioning, verify AWS resources are deleted:

```bash
# Check EC2 instances
aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/<cluster>,Values=owned" \
  --region <region>

# Check S3 buckets
aws s3 ls | grep <cluster>
```
