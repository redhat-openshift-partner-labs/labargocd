# Cluster Deletion Workflow

This diagram visualizes the complete cluster deletion process from trigger to completion.

```mermaid
sequenceDiagram
    autonumber
    participant User
    participant Git
    participant ArgoCD as ArgoCD Controller
    participant K8s as Kubernetes API
    participant Hive as Hive Controller
    participant AWS
    participant CronJob as Cleanup CronJob

    Note over User,CronJob: Phase 1: Deletion Trigger

    User->>Git: Remove clusters/<cluster>/argocd-application.yaml
    User->>Git: git commit && git push
    Git-->>ArgoCD: Webhook/Poll detects change

    Note over User,CronJob: Phase 2: ArgoCD Application Deletion

    ArgoCD->>ArgoCD: App-of-Apps detects missing Application manifest
    ArgoCD->>K8s: Delete Application: <cluster-name>
    K8s-->>ArgoCD: Application deletion initiated

    Note over User,CronJob: Phase 3: Resource Deletion Requests

    ArgoCD->>K8s: Delete KlusterletAddonConfig (sync-wave: 2)
    ArgoCD->>K8s: Delete ManagedCluster (sync-wave: 1)
    ArgoCD->>K8s: Delete MachinePool (sync-wave: 0)
    ArgoCD->>K8s: Delete ClusterDeployment (sync-wave: 0)

    rect rgb(255, 248, 220)
        Note over K8s,Hive: ClusterDeployment blocked by Hive finalizer
        K8s->>K8s: ClusterDeployment enters Terminating state
        K8s->>K8s: Blocked by finalizer: hive.openshift.io/deprovision
        Note over K8s: ClusterDeployment persists until Hive completes cleanup
    end

    Note over User,CronJob: Phase 4: Namespace Terminating (parallel with Hive)

    ArgoCD->>K8s: Delete Namespace: <cluster> (via Application finalizer)
    K8s->>K8s: Namespace enters Terminating state

    rect rgb(255, 240, 240)
        Note over K8s: Namespace deletion cascades to secrets
        K8s->>K8s: Cascade delete Secret: aws-credentials
        K8s->>K8s: Blocked by finalizer: openshiftpartnerlabs.com/deprovision
        K8s->>K8s: Cascade delete Secret: <cluster>-metadata-json
        K8s->>K8s: Blocked by finalizer: openshiftpartnerlabs.com/deprovision
        Note over K8s: Namespace blocked until all finalized resources complete
    end

    Note over User,CronJob: Phase 5: Hive Uninstall Process

    K8s-->>Hive: ClusterDeployment Terminating state detected
    Hive->>K8s: Create ClusterDeprovision resource
    Hive->>K8s: Create Job: <cluster>-uninstall

    Note over Hive,AWS: Uninstall job reads protected secrets (still accessible)

    Hive->>K8s: Read Secret: aws-credentials
    K8s-->>Hive: Return AWS access key, secret key
    Hive->>K8s: Read Secret: <cluster>-metadata-json
    K8s-->>Hive: Return infraID, clusterID, AWS metadata

    Note over User,CronJob: Phase 6: AWS Resource Cleanup (10-15 minutes)

    Hive->>AWS: Delete EC2 instances
    Hive->>AWS: Delete Load Balancers
    Hive->>AWS: Delete Security Groups
    Hive->>AWS: Delete VPC resources
    Hive->>AWS: Delete Route53 records
    Hive->>AWS: Delete S3 buckets
    Hive->>AWS: Delete EBS volumes
    Hive->>AWS: Delete IAM resources
    AWS-->>Hive: Resources deleted

    Hive->>K8s: Update Job: <cluster>-uninstall status.succeeded=1
    Hive->>K8s: Delete ClusterDeprovision
    Hive->>K8s: Remove finalizer from ClusterDeployment
    K8s->>K8s: ClusterDeployment fully deleted

    Note over User,CronJob: Phase 7: Cleanup CronJob (runs every 5 minutes)

    CronJob->>K8s: Query secrets with label: deprovision-pending=true
    K8s-->>CronJob: Return: aws-credentials, <cluster>-metadata-json

    CronJob->>CronJob: Extract cluster name from secret namespace
    CronJob->>K8s: Get Job: <cluster>-uninstall
    K8s-->>CronJob: Job exists with status.succeeded=1

    rect rgb(240, 255, 240)
        Note over CronJob,K8s: Uninstall verified complete - safe to release secrets
        CronJob->>K8s: Remove finalizer from aws-credentials
        CronJob->>K8s: Remove label deprovision-pending from aws-credentials
        CronJob->>K8s: Remove finalizer from <cluster>-metadata-json
        CronJob->>K8s: Remove label deprovision-pending from <cluster>-metadata-json
    end

    Note over User,CronJob: Phase 8: Final Cleanup

    K8s->>K8s: Secret: aws-credentials deleted (finalizer removed)
    K8s->>K8s: Secret: <cluster>-metadata-json deleted (finalizer removed)
    K8s->>K8s: Remaining secrets deleted (namespace cascade)
    K8s->>K8s: Namespace <cluster> deletion completes

    Note over User,CronJob: Deletion Complete
```

## Key Components

| Component | Role |
|-----------|------|
| **Git Repository** | Source of truth for cluster state |
| **ArgoCD App-of-Apps** | Discovers and manages cluster Applications |
| **ArgoCD Controller** | Syncs desired state, handles deletion |
| **Hive Controller** | Manages ClusterDeployment lifecycle |
| **Uninstall Job** | Destroys AWS infrastructure |
| **Cleanup CronJob** | Removes finalizers after uninstall completes |

## Finalizer Protection

Two finalizer mechanisms work together to ensure proper cleanup:

### Secret Finalizers (`openshiftpartnerlabs.com/deprovision`)
Applied to `aws-credentials` and `<cluster>-metadata-json` secrets:
1. Secrets enter Terminating state but persist until finalizer is removed
2. Uninstall job can read secrets while they are in Terminating state
3. CronJob removes finalizers only after uninstall job succeeds

### ClusterDeployment Finalizer (`hive.openshift.io/deprovision`)
Applied by Hive to ClusterDeployment:
1. ClusterDeployment enters Terminating state when deleted
2. Hive creates uninstall job and waits for AWS cleanup
3. Hive removes finalizer only after AWS resources are deleted

## Timing

| Phase | Description | Duration |
|-------|-------------|----------|
| 1-2 | ArgoCD detection and Application deletion | 1-3 minutes |
| 3-5 | Resource deletion requests and Hive uninstall start | < 1 minute |
| 6 | AWS resource cleanup | 10-15 minutes |
| 7 | CronJob finalizer removal | Up to 5 minutes (next scheduled run) |
| 8 | Final namespace cleanup | < 1 minute |
| **Total** | | **15-25 minutes** |
