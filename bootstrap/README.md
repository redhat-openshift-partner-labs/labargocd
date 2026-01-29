# Bootstrap Resources

This directory contains the ArgoCD resources needed to initialize the cluster provisioning system.

## Files

| File | Purpose |
|------|---------|
| `argocd-project.yaml` | Defines the AppProject with permissions for Hive/ACM resources |
| `argocd-app-of-apps.yaml` | App-of-Apps Application that discovers cluster Applications |
| `deprovision-cleanup-cronjob.yaml` | CronJob that removes finalizers after cluster deletion |

## How It Works

### 1. AppProject (`argocd-project.yaml`)

Defines permissions for cluster provisioning:

- **clusterResourceWhitelist**: Allows Hive and ACM cluster-scoped resources
- **namespaceResourceWhitelist**: Allows secrets, jobs, RBAC for per-cluster namespaces
- **sourceRepos**: Git repositories ArgoCD can sync from
- **destinations**: Namespaces ArgoCD can deploy to

### 2. App-of-Apps (`argocd-app-of-apps.yaml`)

Implements the App-of-Apps pattern:

```
App-of-Apps
    │
    ├── scans clusters/*/argocd-application.yaml
    │
    └── creates Application for each cluster found
```

Key configuration:
- `directory.recurse: true` - Scans subdirectories
- `directory.include: '*/argocd-application.yaml'` - Only includes Application manifests
- `prune: true` - Deletes Applications when removed from Git
- `selfHeal: true` - Re-applies Applications if they drift

### 3. Cleanup CronJob (`deprovision-cleanup-cronjob.yaml`)

Runs every 5 minutes to clean up after cluster deletions:

1. Queries secrets with label `deprovision-pending=true`
2. For each secret, checks if the cluster's uninstall job completed
3. If complete, removes the finalizer and label
4. Secrets can then be deleted, allowing namespace termination

## Deployment

Apply bootstrap resources in order:

```bash
# 1. Create the AppProject
oc apply -f bootstrap/argocd-project.yaml

# 2. Create the App-of-Apps
oc apply -f bootstrap/argocd-app-of-apps.yaml

# 3. Create the cleanup CronJob
oc apply -f bootstrap/deprovision-cleanup-cronjob.yaml
```

Or apply all at once:

```bash
oc apply -f bootstrap/
```

## RBAC Requirements

The ArgoCD application controller needs cluster-admin to create Hive and ACM resources:

```bash
oc create clusterrolebinding openshift-gitops-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=openshift-gitops:openshift-gitops-argocd-application-controller
```

## Customization

### Restrict Source Repositories

Change `sourceRepos: ['*']` to specific Git URLs:

```yaml
sourceRepos:
  - 'https://github.com/your-org/your-repo.git'
```

### Restrict Destinations

Limit which namespaces ArgoCD can deploy to:

```yaml
destinations:
  - namespace: 'cluster-*'
    server: https://kubernetes.default.svc
```

### Sync Windows

Add maintenance windows to control when syncs happen:

```yaml
syncWindows:
  - kind: allow
    schedule: '0 6 * * 1-5'  # Weekdays 6 AM
    duration: 12h
    applications:
      - '*'
```
