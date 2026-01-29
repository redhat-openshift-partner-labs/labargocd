# Cluster Instances

This directory contains configuration for each provisioned cluster. Each subdirectory represents one cluster.

## Directory Structure

```
clusters/
├── README.md                     # This file
├── mhillsma-cluster/            # Example cluster
│   ├── kustomization.yaml       # Kustomize overlay
│   ├── patches/
│   │   └── install-config.yaml  # OpenShift install configuration patch
│   └── argocd-application.yaml  # ArgoCD Application resource
└── <new-cluster>/               # Add new clusters here
```

## Creating a New Cluster

### 1. Create cluster directory

```bash
mkdir -p clusters/<cluster-name>/patches
```

### 2. Create kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <cluster-name>

resources:
  - ../../cluster-templates/aws-ha/base

patches:
  - path: patches/install-config.yaml
  - patch: |-
      apiVersion: hive.openshift.io/v1
      kind: ClusterDeployment
      metadata:
        name: cluster-placeholder
      spec:
        clusterName: <cluster-name>
        baseDomain: <your-domain.com>
        platform:
          aws:
            region: <aws-region>
        provisioning:
          imageSetRef:
            name: <clusterimageset-name>
  - patch: |-
      apiVersion: hive.openshift.io/v1
      kind: MachinePool
      metadata:
        name: cluster-placeholder-worker
      spec:
        clusterDeploymentRef:
          name: <cluster-name>
        replicas: <worker-count>
        platform:
          aws:
            type: <instance-type>
  - patch: |-
      apiVersion: cluster.open-cluster-management.io/v1
      kind: ManagedCluster
      metadata:
        name: cluster-placeholder
        labels:
          environment: <environment>
  - patch: |-
      apiVersion: agent.open-cluster-management.io/v1
      kind: KlusterletAddonConfig
      metadata:
        name: cluster-placeholder
      spec:
        clusterName: <cluster-name>
        clusterNamespace: <cluster-name>
```

### 3. Create patches/install-config.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-placeholder-install-config
  namespace: <cluster-name>
stringData:
  install-config.yaml: |
    apiVersion: v1
    baseDomain: <your-domain.com>
    metadata:
      name: <cluster-name>

    controlPlane:
      name: master
      platform:
        aws:
          type: m5.xlarge
          rootVolume:
            iops: 4000
            size: 120
            type: gp3
          zones:
            - <region>a
      replicas: 3

    compute:
      - name: worker
        platform:
          aws:
            type: m5.2xlarge
            rootVolume:
              iops: 2000
              size: 100
              type: gp3
            zones:
              - <region>a
              - <region>b
              - <region>c
        replicas: 3

    networking:
      clusterNetwork:
        - cidr: 10.128.0.0/14
          hostPrefix: 23
      machineNetwork:
        - cidr: 10.0.0.0/16
      serviceNetwork:
        - 172.30.0.0/16
      networkType: OVNKubernetes

    platform:
      aws:
        region: <region>

    fips: false
    publish: External
```

### 4. Create argocd-application.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <cluster-name>
  namespace: openshift-gitops
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: <project-name>
  source:
    repoURL: <git-repo-url>
    targetRevision: <branch>
    path: clusters/<cluster-name>
  destination:
    server: https://kubernetes.default.svc
    namespace: <cluster-name>
  syncPolicy:
    automated:
      prune: false
      selfHeal: false  # CRITICAL: Must be false
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
  ignoreDifferences:
    - group: hive.openshift.io
      kind: ClusterDeployment
      jsonPointers:
        - /status
        - /spec/clusterMetadata
        - /spec/installed
    - group: cluster.open-cluster-management.io
      kind: ManagedCluster
      jsonPointers:
        - /status
        - /spec/managedClusterClientConfigs
```

### 5. Create secrets

Before ArgoCD syncs, create required secrets in the cluster namespace:

```bash
# Create namespace
oc create namespace <cluster-name>

# AWS credentials
oc create secret generic aws-credentials \
  --from-literal=aws_access_key_id=<key> \
  --from-literal=aws_secret_access_key=<secret> \
  -n <cluster-name>

# Pull secret
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n <cluster-name>

# SSH key
oc create secret generic <cluster-name>-ssh-key \
  --from-file=ssh-privatekey=<key-file> \
  --from-file=ssh-publickey=<key-file>.pub \
  --type=kubernetes.io/ssh-auth \
  -n <cluster-name>
```

### 6. Commit and push

```bash
git add clusters/<cluster-name>/
git commit -m "Add <cluster-name> cluster"
git push
```

ArgoCD will detect the new Application and begin provisioning.

## Naming Conventions

- Cluster names should be lowercase, alphanumeric with hyphens
- Match namespace name to cluster name
- Use consistent naming for secrets: `<cluster>-ssh-key`, `<cluster>-install-config`

## Deleting a Cluster

To delete a cluster:

1. Remove the cluster directory from Git
2. Commit and push
3. ArgoCD will delete the Application
4. Hive will deprovision AWS resources (10-15 minutes)
5. CronJob will remove finalizers
6. Namespace deletion completes

See [cluster-deletion-workflow.md](../cluster-deletion-workflow.md) for details.
