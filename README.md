# TVK GitOps with Argo CD

GitOps layout for TrilioVault for Kubernetes (TVK) across a **source** cluster and a **DR site** cluster.
The source cluster runs a MySQL app and backs up the `mysql` namespace daily to a shared NFS Target.
The site cluster mounts the same NFS export for restore.

Git holds what is protected, where backups go and how often. TVK runs the backups.
Backups, Snapshots and Restores never go into this repo.

Design notes: *TrilioVault for Kubernetes with Argo CD: Best Practices* (Claude doc).

## Layout

Each cluster gets two Argo CD Applications: one for the operator and one for the config. The source cluster
gets a third, `mysql-source`, for the app being protected. The manifests are plain YAML, one folder per
cluster, read by Argo CD as a directory (`recurse: true`).

```
argocd-cm-patch.yaml           # Target health check + tracking method for argocd-cm
tvk-operator/
├── application-source.yaml
├── application-site.yaml
└── values.yaml                # shared; optional values-<cluster>.yaml overrides
mysql/
├── application.yaml           # mysql-source → source-cluster, namespace mysql
└── manifests/
    ├── namespace.yaml
    ├── mysql-credentials.yaml # ExternalSecret: all MySQL env vars
    ├── service.yaml
    └── statefulset.yaml       # env only via secretKeyRef
tvk-config/
├── application-source.yaml    # path: tvk-config/source
├── application-site.yaml      # path: tvk-config/site
├── source/
│   ├── targets/nfs-target.yaml
│   ├── policies/{daily,keep-7}.yaml
│   └── backupplans/mysql.yaml # whole mysql namespace
└── site/
    ├── namespace.yaml         # mysql namespace for Target + restores
    └── targets/nfs-target.yaml # same Target + enableBrowsing
```

The NFS Target needs no credentials. It appears once per cluster: **keep the two copies identical** (except
`enableBrowsing` on the site), since both clusters must mount the same export. Check with:

```sh
diff tvk-config/source/targets/nfs-target.yaml tvk-config/site/targets/nfs-target.yaml
```

## Secrets

Every MySQL env var (`MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`) is read from the
`mysql-credentials` Secret via `secretKeyRef`. That Secret is created by External Secrets from
`mysql/source` in the `platform-secrets` store, with properties `rootPassword`, `database`, `user`, `password`.
No plain-text values are committed.

Two Applications rather than one: deleting or rebuilding the config never touches the TVK CRDs. Chart upgrades
and config changes sync and roll back separately, and a failing Target marks only the config Degraded.

## Sync order

Nothing makes the config Application wait for the operator. Instead it retries:
`SkipDryRunOnMissingResource=true` gets it past the missing CRDs, and `retry` (limit 10, backoff up to 5m,
about 37 minutes in all) keeps it syncing until TVK's webhooks accept the resources.

Inside each config Application, sync waves order the resources:

| Wave | What |
| --- | --- |
| -1 | `mysql` Namespace (site only; on source it comes from `mysql-source`) |
| 1 | NFS Target; waits for the `Available` health check |
| 2 | Policies, Hooks |
| 3 | BackupPlans |

## Prerequisites

- Argo CD 2.x/3.x with both clusters registered as `source-cluster` and `site-cluster`.
- External Secrets Operator on the source cluster, with a `ClusterSecretStore` named `platform-secrets`.
- An NFS export reachable from the TVK pods on both clusters, writable by TVK.
- A CSI StorageClass with VolumeSnapshot support on the source cluster for the MySQL PVC.
- The keys from `argocd-cm-patch.yaml` merged into your Argo CD install (Helm values `configs.cm`, or a patch).

## Placeholders to replace

| File | Value |
| --- | --- |
| `tvk-*/application-*.yaml`, `mysql/application.yaml` | `repoURL: https://git.example.com/platform/tvk-gitops.git` |
| `tvk-operator/application-*.yaml` | `REPLACE_WITH_PINNED_CHART_VERSION` |
| `tvk-config/*/targets/nfs-target.yaml` | `nfsExport: nfs.example.com:/exports/tvk`, `nfsOptions` |
| `mysql/manifests/mysql-credentials.yaml` | secret store name and `remoteRef.key` |
| `mysql/manifests/statefulset.yaml` | image tag, `storageClassName`, PVC size |

Not scaffolded yet: the TVK License (one per cluster; keep the key out of plain git) and MySQL quiesce Hooks.
Without hooks the daily backup is crash-consistent (InnoDB recovers on restore).

## Bootstrap

```sh
for c in source site; do
  kubectl -n argocd apply -f tvk-operator/application-$c.yaml \
                          -f tvk-config/application-$c.yaml
done
kubectl -n argocd apply -f mysql/application.yaml
```

The Application files are applied by hand, so after editing one (a chart version bump, for example), apply it
again with `kubectl`. Argo CD syncs everything those Applications point at from git.

Start with `prune: false` (the default here). Turn it on in the `tvk-config` Applications once a few syncs and
scheduled backups have run cleanly. Targets and BackupPlans also carry `Prune=false,Delete=false`, so retire
them by hand in reverse order: BackupPlan, then Policies and Hooks, then the Target.

## Adding a protected application

1. Add `tvk-config/source/backupplans/<app>.yaml` (sync wave `3`, `Prune=false,Delete=false`).
2. Open a PR. Argo CD picks up any new file in the folder.

## Adding a cluster

Copy `application-site.yaml` in both `tvk-operator/` and `tvk-config/` to `application-<cluster>.yaml`, change
the name, destination and path, and add `tvk-config/<cluster>/`. Past a few clusters, switch to an
ApplicationSet with a cluster generator.

Check field names against the CRDs for your TVK version (`kubectl explain target.spec`) before the first rollout.
