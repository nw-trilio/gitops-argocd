# TVK ArgoCD Example

How to run TrilioVault for Kubernetes (TVK) with Argo CD: the guidelines first, then a working example.

Guide page: https://claude.ai/artifact/8ARJMznYDzLNiEbGyEK8hF

## Part 1: Guidelines

These apply to any TVK + Argo CD setup.

| Guideline | Why |
| --- | --- |
| **Git holds intent only** | Commit Targets, Policies, Hooks and BackupPlans. Backups, Restores and Snapshots are runtime records created by TVK; in git, Argo CD would recreate or prune them. |
| **Split operator and config** | Use one Application for the operator and one for the config, per cluster. Deleting or rebuilding the config then never removes the CRDs, which would delete every TVK resource. |
| **Pin versions** | Pin the chart version and upgrade through a PR, so an upgrade is reviewed and can be rolled back. |
| **No secrets in git** | Use External Secrets or Sealed Secrets. Values that can't reference a Secret, like the license key, are set in the cluster and ignored by Argo CD. |
| **Protect backup resources** | Put `Prune=false,Delete=false` on Targets and BackupPlans, and start with `prune: false`. A bad commit then can't wipe your backup config. |
| **Order with sync waves** | Target, then Policies and Hooks, then BackupPlans. Each depends on the one before it. |
| **Retry, don't block** | Use `SkipDryRunOnMissingResource=true` and `retry` on config apps, so they sync once the operator's CRDs and webhooks are ready. |
| **Quiesce databases** | Add Hooks to flush and freeze databases. Without them, backups are only crash-consistent. |
| **Test restores** | Restore on the DR cluster on a schedule. A backup you have never restored is untested. |

> **Tip:** reuse with Kustomize. Put shared resources such as the Target in a base, and patch per-cluster
> differences (like `enableBrowsing`) in overlays.

## Part 2: Example

Back up a MySQL app daily on a source cluster and restore it on a DR site cluster, all synced by Argo CD from this repo.

### How it fits together

```
                         Argo CD (syncs from this repo)
                   ┌───────────────┴───────────────┐
source-cluster (runs the app)          site-cluster (DR, restores)
  tvk-operator-source  Helm chart        tvk-operator-site  Helm chart
  tvk-config-source    License, Target,  tvk-config-site    Namespace, Target (browsable)
                       Policies, BackupPlan
  mysql-source         MySQL StatefulSet
          │ writes backups                       ▲ reads backups
          └──────────► Shared Target ────────────┘
                (same backup storage on both clusters)
```

Operator and config are separate Applications, so removing the config never deletes the TVK CRDs.

### What you can try

| Try | How |
| --- | --- |
| Daily backup | The whole `mysql` namespace at 01:00, keeping the last 7. |
| Restore on the DR site | The site cluster browses the same backups through the shared Target. |
| Change schedule or retention | Edit `policies/daily.yaml` or `keep-7.yaml`, then open a PR. |
| Protect another app | Add `backupplans/<app>.yaml`. Argo CD picks it up. |
| Upgrade TVK | Bump `targetRevision` in `tvk-operator/application-*.yaml`. |
| Keep secrets out of git | MySQL passwords come from External Secrets. The license key is set in the cluster. |

### Repo layout

```
argocd-cm-patch.yaml        # Target health check, tracking method
tvk-operator/
  application-{source,site}.yaml
  values.yaml               # shared Helm values
mysql/
  application.yaml          # mysql-source
  manifests/                # namespace, ExternalSecret, service, statefulset
tvk-config/
  application-{source,site}.yaml
  source/
    license.yaml            # placeholder key
    targets/nfs-target.yaml
    policies/{daily,keep-7}.yaml
    backupplans/mysql.yaml
  site/
    namespace.yaml
    targets/nfs-target.yaml # same as source + enableBrowsing
```

### Get started

You need:

- Argo CD with two clusters registered, here `source-cluster` and `site-cluster`. These names are only examples:
  use any names (for example `us-east1` and `eu-west1`) and match `destination.name` in the Applications.
- Backup storage both clusters can reach, used as a shared Target (NFS in this demo)
- External Secrets with a `ClusterSecretStore` named `platform-secrets` (source)
- A CSI StorageClass with VolumeSnapshot support (source)
- `argocd-cm-patch.yaml` merged into `argocd-cm`

**1. Apply the Applications**

```sh
for c in source site; do
  kubectl -n argocd apply -f tvk-operator/application-$c.yaml \
                          -f tvk-config/application-$c.yaml
done
kubectl -n argocd apply -f mysql/application.yaml
```

**2. Set the license key.** Argo CD ignores `spec.key`, so this value stays put.

```sh
kubectl --context source-cluster -n trilio-system patch license tvk-license \
  --type merge -p "{\"spec\":{\"key\":\"$(cat tvk-license.key)\"}}"
```

### Sync order

| Wave | Resource |
| --- | --- |
| -1 | `mysql` Namespace (site) |
| 0 | License (source) |
| 1 | Shared Target, waits until `Available` |
| 2 | Policies |
| 3 | BackupPlans |

### Replace before you use it

| File | Value |
| --- | --- |
| `tvk-config/*/targets/nfs-target.yaml` | Target storage (`nfsExport` is a test IP today) |
| `mysql/manifests/mysql-credentials.yaml` | Secret store and `remoteRef.key` |
| `mysql/manifests/statefulset.yaml` | Image, StorageClass, PVC size |
| `*/application*.yaml` | `repoURL`, if you fork |
| `tvk-config/source/license.yaml` | Key, in the cluster only |

### In this repo

- Keep both Target files identical, apart from `enableBrowsing`.
- `prune: false` to start. Turn it on once backups run cleanly.
- Targets and BackupPlans are never pruned or deleted by Argo CD. Remove them by hand: BackupPlan, then Policies, then Target.
- Check field names against your TVK version: `kubectl explain target.spec`.

---

Example only: adjust before using in production.
