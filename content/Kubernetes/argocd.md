---
title: "argocd"
---
Chart: [https://artifacthub.io/packages/helm/argo/argo-cd](https://artifacthub.io/packages/helm/argo/argo-cd)

Repo: [https://argo-cd.readthedocs.io/en/stable/getting\_started/#1-install-argo-c://github.com/argoproj/argo-helm](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-c://github.com/argoproj/argo-helm)

## Install

```plaintext
helm upgrade -i argocd argo/argo-cd  -n argocd --create-namespace -f argocd-values.yaml
```

## Local Users

### Create

Add the following to the `argocd-cm` ConfigMap:

```plaintext
accounts.dev: login
accounts.ci: apiKey
accounts.technat: login
accounts.admin.enabled: "false"
```

Or the `argocd.config` key in the helm chart.

### Permissions

Now edit the `argocd-rbac-cm` ConfigMap and add the following to map the technat user admin rights.

```plaintext
data:
 policy.default: role:readonly
 policy.csv: |
   g, technat, role:admin
   p, role:ci, applications, *, */*, allow
   g, ci, role:ci
```

Or the `argocd.rbacConfig` key in the helm chart.

### Password

Using the argocd cli:

```plaintext
argocd login --grpc-web cd.eatthis.ch
argocd account update-password --account dev --new-password ${PASSWORD_HERE} --current-password ${YOUR_PASSWORD}
argocd account update-password --account technat --new-password ${PASSWORD_HERE} --current-password ${YOUR_PASSWORD}
argocd account generate-token --account ci
```

### References

\- [https://itnext.io/argocd-users-access-and-rbac-ddf9f8b51bad](https://itnext.io/argocd-users-access-and-rbac-ddf9f8b51bad)

## Files

Custom ArgoCD values: [argocd-values.yaml](/Kubernetes/argocd-values.yaml)
