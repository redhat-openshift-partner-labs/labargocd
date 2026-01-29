# Advanced Features

This document covers advanced features for scaling and optimizing cluster provisioning.

## ApplicationSets

For managing many similar clusters, use ArgoCD ApplicationSets:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-provisioning-set
  namespace: openshift-gitops
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/opl-argocd.git
        revision: main
        directories:
          - path: clusters/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: cluster-provisioning
      source:
        repoURL: https://github.com/your-org/opl-argocd.git
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: false
          selfHeal: false
        syncOptions:
          - CreateNamespace=true
      ignoreDifferences:
        - group: hive.openshift.io
          kind: ClusterDeployment
          jsonPointers:
            - /status
            - /spec/clusterMetadata
            - /spec/installed
```

This automatically creates Applications for each cluster directory.

## ClusterPools

Pre-provision clusters for faster availability:

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterPool
metadata:
  name: aws-dev-pool
  namespace: hive
spec:
  baseDomain: dev.example.com
  imageSetRef:
    name: img4.14.0-x86-64-appsub
  platform:
    aws:
      credentialsSecretRef:
        name: aws-credentials
      region: us-east-1
  pullSecretRef:
    name: pull-secret
  size: 2  # Keep 2 clusters ready
  maxSize: 5
  runningCount: 1  # Keep 1 running, hibernate rest
```

### Claiming from Pool

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterClaim
metadata:
  name: my-cluster
  namespace: hive
spec:
  clusterPoolName: aws-dev-pool
  lifetime: 8h  # Auto-delete after 8 hours
```

## Custom Manifests

Install additional manifests during provisioning:

```yaml
# Create ConfigMap with manifests
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-manifests
data:
  99_custom-namespace.yaml: |
    apiVersion: v1
    kind: Namespace
    metadata:
      name: custom-app
  99_custom-rbac.yaml: |
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    ...
```

Reference in ClusterDeployment:

```yaml
spec:
  provisioning:
    manifestsConfigMapRef:
      name: custom-manifests
```

## Spot Instances (Development Only)

Reduce costs with spot instances:

```yaml
# In MachinePool
spec:
  platform:
    aws:
      type: m5.2xlarge
      spotMarketOptions:
        maxPrice: "0.50"  # Maximum hourly price
```

**Warning**: Spot instances can be terminated with 2-minute notice.

## Multi-Region Deployment

### Cross-Region Strategy

```
clusters/
├── us-east-1/
│   ├── prod-cluster-use1/
│   └── dev-cluster-use1/
├── us-west-2/
│   ├── prod-cluster-usw2/
│   └── dev-cluster-usw2/
└── eu-west-1/
    └── prod-cluster-euw1/
```

### Region-Specific Secrets

Create separate AWS credentials per region:

```bash
oc create secret generic aws-credentials-use1 \
  --from-literal=aws_access_key_id=... \
  -n <cluster>
```

## Autoscaling Configuration

### Cluster Autoscaler

Enable on provisioned clusters:

```yaml
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    delayAfterDelete: 5m
    unneededTime: 5m
  resourceLimits:
    maxNodesTotal: 20
```

### MachinePool Autoscaling

```yaml
apiVersion: hive.openshift.io/v1
kind: MachinePool
spec:
  autoscaling:
    minReplicas: 3
    maxReplicas: 10
```

## Backup Strategy

### Velero for Cluster Backups

Install Velero on provisioned clusters:

```yaml
# custom-manifests ConfigMap
99_velero-namespace.yaml: |
  apiVersion: v1
  kind: Namespace
  metadata:
    name: velero
```

### etcd Backup

Schedule etcd backups via CronJob on provisioned clusters.

## Cost Optimization

| Strategy | Savings | Use Case |
|----------|---------|----------|
| Spot instances | Up to 90% | Development |
| Hibernation | ~70% | Non-production hours |
| Right-sizing | Variable | After usage analysis |
| Reserved instances | 30-60% | Long-running production |
| ClusterPools | Faster provisioning | Ephemeral environments |

### Hibernation Schedule

Automate hibernation with CronJobs:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hibernate-dev-clusters
spec:
  schedule: "0 20 * * 1-5"  # 8 PM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hibernate
              image: quay.io/openshift/origin-cli
              command:
                - /bin/sh
                - -c
                - |
                  oc patch clusterdeployment dev-cluster -n dev-cluster \
                    --type merge -p '{"spec":{"powerState":"Hibernating"}}'
```

## Security Hardening

### Private Clusters

```yaml
publish: Internal
```

### FIPS Mode

```yaml
fips: true
```

### Network Policies

Add via custom manifests:

```yaml
99_network-policies.yaml: |
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: default-deny
    namespace: default
  spec:
    podSelector: {}
    policyTypes:
      - Ingress
      - Egress
```

## Monitoring Integration

### Prometheus Federation

Configure provisioned clusters to federate metrics to hub.

### Alertmanager

Route alerts from provisioned clusters to central alerting.

## Further Reading

- [Hive ClusterPool Documentation](https://github.com/openshift/hive/blob/master/docs/clusterpools.md)
- [ArgoCD ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [AWS Spot Instances Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html)
