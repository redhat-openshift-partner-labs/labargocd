# Adding New Clusters

This guide walks through adding a new cluster to the GitOps provisioning system.

## Quick Start

```bash
# 1. Create cluster directory
mkdir -p clusters/<cluster-name>/patches

# 2. Copy template files
cp -r clusters/mhillsma-cluster/* clusters/<cluster-name>/

# 3. Update values in all files
# - kustomization.yaml: cluster name, region, instance types
# - patches/install-config.yaml: domain, networking
# - argocd-application.yaml: cluster name, project

# 4. Create secrets in cluster namespace
oc create namespace <cluster-name>
# (see Secret Creation below)

# 5. Commit and push
git add clusters/<cluster-name>/
git commit -m "Add <cluster-name> cluster"
git push
```

## File Structure

Each cluster needs:

```
clusters/<cluster-name>/
├── kustomization.yaml       # Kustomize overlay with patches
├── patches/
│   └── install-config.yaml  # OpenShift install configuration
└── argocd-application.yaml  # ArgoCD Application resource
```

## Configuration Parameters

### Required Values

| Parameter | Description | Example |
|-----------|-------------|---------|
| Cluster name | Unique identifier | `prod-cluster-01` |
| Base domain | DNS domain | `prod.example.com` |
| AWS region | Deployment region | `us-east-1` |
| ClusterImageSet | OpenShift version | `img4.14.0-x86-64-appsub` |

### Instance Types

| Environment | Control Plane | Workers |
|-------------|---------------|---------|
| Development | m5.xlarge | m5.xlarge |
| Production | m5.2xlarge | m5.2xlarge |
| Large Production | m5.2xlarge | m5.4xlarge |

### Replica Counts

| Environment | Control Plane | Workers |
|-------------|---------------|---------|
| Development | 1-3 | 2 |
| Production | 3 | 3-6 |
| Large Production | 3 | 6-12 |

## Secret Creation

Create secrets before ArgoCD sync:

```bash
# Create namespace first
oc create namespace <cluster-name>

# AWS credentials
oc create secret generic aws-credentials \
  --from-literal=aws_access_key_id=AKIA... \
  --from-literal=aws_secret_access_key=... \
  -n <cluster-name>

# Pull secret (from console.redhat.com)
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <cluster-name>

# SSH key
ssh-keygen -t rsa -b 4096 -f <cluster-name>-ssh-key -N ""
oc create secret generic <cluster-name>-ssh-key \
  --from-file=ssh-privatekey=<cluster-name>-ssh-key \
  --from-file=ssh-publickey=<cluster-name>-ssh-key.pub \
  --type=kubernetes.io/ssh-auth \
  -n <cluster-name>
```

## Customization Examples

### Single Availability Zone

For development clusters:

```yaml
# patches/install-config.yaml
controlPlane:
  platform:
    aws:
      zones:
        - us-east-1a
compute:
  - platform:
      aws:
        zones:
          - us-east-1a
```

### Custom Network CIDRs

```yaml
networking:
  clusterNetwork:
    - cidr: 192.168.0.0/16
      hostPrefix: 23
  machineNetwork:
    - cidr: 172.16.0.0/16
  serviceNetwork:
    - 10.254.0.0/16
```

### FIPS Mode

For compliance requirements:

```yaml
# patches/install-config.yaml
fips: true
```

### Private Cluster

No public endpoints (requires VPN):

```yaml
publish: Internal
```

### AWS Tags

For cost tracking:

```yaml
platform:
  aws:
    region: us-east-1
    userTags:
      owner: platform-team
      environment: production
      cost-center: engineering
```

## Verification

After pushing:

```bash
# Check ArgoCD detected the Application
oc get application -n openshift-gitops

# Watch cluster provisioning
oc get clusterdeployment -n <cluster-name> -w

# Check provision job
oc get pods -n <cluster-name>
```

Provisioning takes 30-45 minutes.

## Naming Conventions

- **Cluster name**: lowercase, alphanumeric, hyphens only
- **Namespace**: same as cluster name
- **Secret names**: `<cluster>-ssh-key`, `<cluster>-install-config`
- **MachinePool**: `<cluster>-worker`

## Common Issues

### Sync fails immediately

Check secrets exist:
```bash
oc get secrets -n <cluster-name>
```

### ClusterImageSet not found

Check available versions:
```bash
oc get clusterimageset
```

### AWS quota exceeded

Check quotas before provisioning:
```bash
aws service-quotas list-service-quotas \
  --service-code ec2 --region <region>
```

See [troubleshooting.md](troubleshooting.md) for more issues.
