# AWS HA Cluster Template

This template provisions a highly available OpenShift cluster on AWS with 3 control plane nodes and customizable worker nodes.

## ⚠️ CRITICAL: ArgoCD Configuration Requirement

**If using ArgoCD for cluster provisioning, you MUST disable selfHeal to prevent cluster deletion failures.**

### The Problem

Hive creates critical secrets during cluster provisioning:
- `<cluster>-metadata-json` - Required for AWS resource cleanup during deletion
- `<cluster>-admin-kubeconfig` - Admin access to provisioned cluster
- `<cluster>-admin-password` - Admin password

These secrets are **NOT defined in Git**. When ArgoCD's `selfHeal: true` is enabled, ArgoCD reconciles ALL resources in the namespace against Git. Since these secrets aren't in Git, **selfHeal deletes them**, causing deprovisioning to fail with:
```
error="secrets \"<cluster>-metadata-json\" not found"
```

### Required ArgoCD Application Configuration

```yaml
spec:
  syncPolicy:
    automated:
      prune: false
      selfHeal: false  # CRITICAL: Must be false to preserve Hive-created secrets
      allowEmpty: false

  # Also configure ignoreDifferences to prevent drift detection
  ignoreDifferences:
    - group: hive.openshift.io
      kind: ClusterDeployment
      jsonPointers:
        - /status
        - /spec/clusterMetadata  # Prevents drift in ClusterDeployment spec
        - /spec/installed
    - group: cluster.open-cluster-management.io
      kind: ManagedCluster
      jsonPointers:
        - /status
        - /spec/managedClusterClientConfigs
```

### Why ignoreDifferences Alone Doesn't Work

`ignoreDifferences` only prevents ArgoCD from detecting drift in the **ClusterDeployment spec fields**. It does NOT prevent ArgoCD from deleting the **actual secret resources** during namespace reconciliation. SelfHeal reconciles all resources in the namespace, not just the ClusterDeployment.

See the [main README troubleshooting section](../../README.md) for more details.

## Architecture

- **Control Plane**: 3 nodes across 3 availability zones
- **Workers**: Configurable count across 3 availability zones
- **Load Balancers**: AWS ELB for API and Ingress
- **Storage**: EBS gp3 volumes with configurable IOPS
- **Networking**: OVN-Kubernetes CNI

## Components

### Base Resources

- **namespace.yaml**: Cluster-specific namespace
- **clusterdeployment.yaml**: Hive ClusterDeployment resource
- **machinepool-worker.yaml**: Worker node MachinePool
- **managedcluster.yaml**: ACM ManagedCluster and KlusterletAddonConfig
- **install-config-secret.yaml**: OpenShift installation configuration

### Required Secrets (Created separately)

- **aws-credentials**: AWS access credentials
- **pull-secret**: Red Hat registry pull secret
- **<cluster-name>-ssh-key**: SSH key for node access

## Configuration Parameters

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `clusterName` | Name of the cluster | `dev-cluster-01` |
| `baseDomain` | DNS base domain | `dev.example.com` |
| `region` | AWS region | `us-east-1` |
| `openshiftVersion` | ClusterImageSet name | `openshift-v4.14.0` |

### Control Plane Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `controlPlaneInstanceType` | EC2 instance type | `m5.xlarge` |
| `controlPlaneReplicas` | Number of control plane nodes | `3` |

Supported control plane instance types:
- `m5.xlarge` (4 vCPU, 16 GB RAM) - Minimum for production
- `m5.2xlarge` (8 vCPU, 32 GB RAM) - Recommended for production
- `m5.4xlarge` (16 vCPU, 64 GB RAM) - Large production clusters

### Worker Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `workerInstanceType` | EC2 instance type | `m5.2xlarge` |
| `workerReplicas` | Number of worker nodes | `3` |
| `workerZones` | Comma-separated AZ list | `us-east-1a,us-east-1b,us-east-1c` |

Supported worker instance types:
- `m5.large` (2 vCPU, 8 GB RAM) - Development only
- `m5.xlarge` (4 vCPU, 16 GB RAM) - Small workloads
- `m5.2xlarge` (8 vCPU, 32 GB RAM) - Standard workloads
- `m5.4xlarge` (16 vCPU, 64 GB RAM) - Large workloads
- `m5.8xlarge` (32 vCPU, 128 GB RAM) - Very large workloads

### Networking Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `clusterNetworkCIDR` | Pod network CIDR | `10.128.0.0/14` |
| `serviceNetworkCIDR` | Service network CIDR | `172.30.0.0/16` |
| `machineNetworkCIDR` | Machine network CIDR | `10.0.0.0/16` |

## AWS Prerequisites

### 1. IAM Permissions

The AWS credentials must have permissions to:
- Create and manage EC2 instances, security groups, and VPCs
- Create and manage ELB/ALB load balancers
- Create and manage Route53 DNS records
- Create and manage S3 buckets
- Create and manage IAM roles and instance profiles

See [Hive AWS Permissions](https://github.com/openshift/hive/blob/master/docs/using-hive.md#aws-credentials) for the full IAM policy.

### 2. DNS Zone

A Route53 hosted zone must exist for your base domain:

```bash
# Check if zone exists
aws route53 list-hosted-zones-by-name --dns-name dev.example.com

# Create zone if needed
aws route53 create-hosted-zone --name dev.example.com --caller-reference $(date +%s)
```

### 3. Service Quotas

Ensure your AWS account has sufficient quotas:

| Resource | Minimum Required | Recommended |
|----------|------------------|-------------|
| VPCs per region | 1 | 5 |
| EC2 instances (m5.xlarge) | 6 | 20 |
| Elastic IPs | 3 | 10 |
| EBS volumes (gp3) | 10 | 50 |

Check quotas:
```bash
aws service-quotas list-service-quotas --service-code ec2 --region us-east-1
```

## Sizing Recommendations

### Development Environment

```yaml
controlPlaneInstanceType: m5.xlarge
controlPlaneReplicas: "3"
workerInstanceType: m5.xlarge
workerReplicas: "2"
```

**Capacity**: ~50 pods, light workloads
**Monthly Cost**: ~$500-600 USD

### Production Environment

```yaml
controlPlaneInstanceType: m5.2xlarge
controlPlaneReplicas: "3"
workerInstanceType: m5.2xlarge
workerReplicas: "6"
```

**Capacity**: ~300-400 pods, standard workloads
**Monthly Cost**: ~$2,000-2,500 USD

### Large Production Environment

```yaml
controlPlaneInstanceType: m5.2xlarge
controlPlaneReplicas: "3"
workerInstanceType: m5.4xlarge
workerReplicas: "9"
```

**Capacity**: ~600-800 pods, large workloads
**Monthly Cost**: ~$4,000-5,000 USD

## Customization Examples

### Example 1: Single AZ Deployment (Not Recommended for Production)

Modify the zones in your overlay:

```yaml
patches:
  - target:
      kind: MachinePool
    patch: |-
      - op: replace
        path: /spec/platform/aws/zones
        value:
          - us-east-1a
```

### Example 2: Custom Network CIDRs

Useful when integrating with existing VPC infrastructure:

```yaml
# cluster-config.yaml
data:
  clusterNetworkCIDR: 192.168.0.0/16
  serviceNetworkCIDR: 10.254.0.0/16
  machineNetworkCIDR: 172.16.0.0/16
```

### Example 3: FIPS Mode

For government/compliance requirements:

```yaml
patches:
  - target:
      kind: Secret
      name: cluster-placeholder-install-config
    patch: |-
      - op: replace
        path: /stringData/install-config.yaml
        value: |
          # ... existing config ...
          fips: true
```

### Example 4: Proxy Configuration

For environments requiring HTTP proxy:

```yaml
# Add to install-config Secret
proxy:
  httpProxy: http://proxy.example.com:8080
  httpsProxy: https://proxy.example.com:8080
  noProxy: .cluster.local,.svc,127.0.0.1,localhost,169.254.169.254,10.0.0.0/16
```

### Example 5: Custom Instance Store

For workloads requiring NVMe instance storage:

```yaml
workerInstanceType: m5d.2xlarge  # 'd' variants include NVMe SSD
```

### Example 6: Spot Instances (Development Only)

Reduce costs for non-production environments:

```yaml
# Note: Requires additional Hive configuration
platform:
  aws:
    type: m5.2xlarge
    spotMarketOptions:
      maxPrice: "0.5"
```

## Advanced Configuration

### Autoscaling Worker Nodes

Enable autoscaling instead of fixed replica count:

```yaml
# In machinepool-worker.yaml
spec:
  autoscaling:
    minReplicas: 3
    maxReplicas: 10
  # Remove or comment out:
  # replicas: 3
```

### Custom AMI

Use a custom RHCOS AMI:

```yaml
platform:
  aws:
    amiID: ami-0123456789abcdef
    region: us-east-1
```

### Additional Tags

Add AWS resource tags for cost tracking:

```yaml
platform:
  aws:
    region: us-east-1
    userTags:
      owner: platform-team
      environment: production
      cost-center: engineering
      project: openshift-platform
```

### Private Cluster

Deploy a private cluster (no public endpoints):

```yaml
# In install-config Secret
publish: Internal
```

Note: This requires VPN or Direct Connect to access the cluster.

## Monitoring Deployment

### Check ClusterDeployment Status

```bash
# Get ClusterDeployment status
oc get clusterdeployment <cluster-name> -n <namespace>

# Watch for completion
oc get clusterdeployment <cluster-name> -n <namespace> -w

# Detailed status
oc describe clusterdeployment <cluster-name> -n <namespace>
```

### Monitor Provision Job

```bash
# List provision jobs
oc get jobs -n <namespace>

# View job logs
oc logs -n <namespace> job/<cluster-name>-provision -f

# Check job pods
oc get pods -n <namespace> -l job-name=<cluster-name>-provision
```

### View Hive Logs

```bash
# Hive controller logs
oc logs -n hive deployment/hive-controllers -c hive-controllers --tail=100 -f
```

## Troubleshooting

### Issue: Cluster stuck in "Provisioning"

**Symptoms**: ClusterDeployment remains in Provisioning state for >60 minutes

**Diagnosis**:
```bash
# Check provision job logs
oc logs -n <namespace> job/<cluster-name>-provision

# Check for AWS API errors
oc get clusterdeployment <cluster-name> -n <namespace> -o yaml | grep -A 10 conditions
```

**Common Causes**:
- Insufficient AWS quotas
- Invalid AWS credentials
- DNS zone not found
- Network connectivity issues

### Issue: AWS quota exceeded

**Symptoms**: Error message about EC2 instance limits

**Solution**:
```bash
# Request quota increase
aws service-quotas request-service-quota-increase \
  --service-code ec2 \
  --quota-code L-1216C47A \
  --desired-value 50 \
  --region us-east-1
```

### Issue: DNS errors

**Symptoms**: Cannot resolve cluster API endpoint

**Diagnosis**:
```bash
# Check if Route53 zone exists
aws route53 list-hosted-zones-by-name --dns-name <base-domain>

# Verify zone is public (not private)
aws route53 get-hosted-zone --id <zone-id>
```

**Solution**: Ensure Route53 hosted zone exists and matches `baseDomain`

### Issue: Network CIDR conflicts

**Symptoms**: Cluster provision fails with networking errors

**Solution**: Ensure CIDRs don't conflict with existing VPC or on-premises networks

## Resource Cleanup

When deprovisioning, Hive automatically cleans up:
- EC2 instances
- Security groups
- Load balancers
- Route53 records
- S3 buckets
- VPC resources

To verify cleanup:

```bash
# List EC2 instances
aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/<cluster-name>,Values=owned" \
  --region <region>

# Check S3 buckets
aws s3 ls | grep <cluster-name>
```

## Cost Optimization

1. **Use Spot Instances for dev**: Save up to 90%
2. **Enable hibernation**: Stop clusters outside business hours
3. **Right-size instances**: Monitor actual usage and adjust
4. **Use gp3 instead of gp2**: Better price/performance ratio
5. **Clean up failed provisions**: Delete stuck clusters to avoid costs
6. **Use Reserved Instances**: For long-running production clusters

## Further Reading

- [AWS OpenShift Best Practices](https://docs.openshift.com/container-platform/latest/installing/installing_aws/preparing-to-install-on-aws.html)
- [Hive ClusterDeployment API](https://github.com/openshift/hive/blob/master/docs/clusterpools.md)
- [AWS EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [OpenShift Networking](https://docs.openshift.com/container-platform/latest/networking/understanding-networking.html)
