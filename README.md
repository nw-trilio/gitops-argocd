# TVK ArgoCD Example

Example repo: Argo CD deploys TrilioVault for Kubernetes (TVK) to a **source** cluster and a **DR site** cluster.
MySQL on the source cluster is backed up daily to a shared Target, and the site cluster can restore from it.

**Guide:** https://claude.ai/artifact/8ARJMznYDzLNiEbGyEK8hF covers the architecture, what to try, the layout, setup and placeholders.

## Quick start

```sh
for c in source site; do
  kubectl -n argocd apply -f tvk-operator/application-$c.yaml \
                          -f tvk-config/application-$c.yaml
done
kubectl -n argocd apply -f mysql/application.yaml

# License key: set in the cluster, never in git
kubectl --context source-cluster -n trilio-system patch license tvk-license \
  --type merge -p "{\"spec\":{\"key\":\"$(cat tvk-license.key)\"}}"
```

Example only: replace the Target storage, secret store and license key before real use.
