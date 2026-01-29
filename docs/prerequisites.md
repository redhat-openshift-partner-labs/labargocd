# Prerequisites

This document covers all prerequisites for cluster provisioning.

## Hub Cluster Requirements

| Requirement | Version |
|-------------|---------|
| OpenShift | 4.12+ |
| Red Hat ACM | 2.8+ |
| OpenShift GitOps | Latest |
| Hive | Installed with ACM |

## Install OpenShift GitOps

```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for operator
oc wait --for=condition=Ready pods -l control-plane=controller-manager \
  -n openshift-gitops --timeout=300s
```

## Install ACM

```bash
# Create namespace
oc create namespace open-cluster-management

oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: advanced-cluster-management
  namespace: open-cluster-management
spec:
  channel: release-2.9
  name: advanced-cluster-management
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for operator
oc wait --for=condition=Ready pods -l app=multiclusterhub-operator \
  -n open-cluster-management --timeout=300s

# Create MultiClusterHub
oc apply -f - <<EOF
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
EOF
```

## AWS Requirements

### IAM Permissions

The AWS credentials need permissions to:
- Create/manage EC2 instances, security groups, VPCs
- Create/manage ELB/ALB load balancers
- Create/manage Route53 DNS records
- Create/manage S3 buckets
- Create/manage IAM roles and instance profiles

See [Hive AWS Permissions](https://github.com/openshift/hive/blob/master/docs/using-hive.md#aws-credentials) for the full IAM policy.

### DNS Zone

A Route53 hosted zone must exist for your base domain:

```bash
# Check if zone exists
aws route53 list-hosted-zones-by-name --dns-name example.com

# Create if needed
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)
```

### Service Quotas

Ensure sufficient AWS quotas:

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| VPCs per region | 1 | 5 |
| EC2 instances (m5.xlarge) | 6 | 20 |
| Elastic IPs | 3 | 10 |
| EBS volumes (gp3) | 10 | 50 |

Check quotas:
```bash
aws service-quotas list-service-quotas \
  --service-code ec2 --region us-east-1
```

## Access Requirements

| Requirement | Description |
|-------------|-------------|
| AWS Account | With sufficient permissions |
| Red Hat Pull Secret | From console.redhat.com |
| Git Repository | Access to this repository |
| DNS Domain | Configured in Route53 |

## Verify ClusterImageSets

Check available OpenShift versions:

```bash
oc get clusterimageset
```

The hub must have ClusterImageSets for versions you want to deploy.

## ArgoCD RBAC

Grant ArgoCD cluster-admin for Hive/ACM resources:

```bash
oc create clusterrolebinding openshift-gitops-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=openshift-gitops:openshift-gitops-argocd-application-controller
```
