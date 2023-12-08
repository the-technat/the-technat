## Base OS

Flashed Ubuntu Server (64-bit) onto an SD card (32 GB) and booted up. So far I created my own user and run [chezmoi](https://chezmoi.io) over it as well as joined the tailnet.

## K3s

### Prerequisites

The docs have a section about [Raspberry Pi's](https://docs.k3s.io/advanced#raspberry-pi). The important part from there is that you need some extra kernel modules:

```bash
sudo apt install linux-modules-extra-raspi
```

### Install

K3s is so easy to install, just simply run:

```bash
k3sup install --cluster --local --k3s-extra-args '--disable=traefik --disable=servicelb'
```

## External-secrets

When K3s is up & running, the next thing is to get [ESO](https://external-secrets.io) ready:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm upgrade -i external-secrets external-secrets/external-secrets --create-namespace -n external-secrets
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: akeyless-secret-creds
type: Opaque
stringData:
  accessId: "p-XXXX"
  accessType:  api_key
  accessTypeParam: "access_secret"
EOF
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: akeyless-secret-store
spec:
  provider:
    akeyless:
      akeylessGWApiURL: "https://api.akeyless.io"
      authSecretRef:
        secretRef:
          accessID:
            name: akeyless-secret-creds
            key: accessId
            namespace: external-secrets
          accessType:
            name: akeyless-secret-creds
            key: accessType
            namespace: external-secrets
          accessTypeParam:
            name: akeyless-secret-creds
            key: accessTypeParam
            namespace: external-secrets
EOF
```

## Argo CD

When ESO is ready, you can kick off GitOps:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm upgrade -i argocd --create-namespace -n argocd argo/argo-cd
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  project: default
  source:
    path: definitions
    repoURL: https://github.com/alleaffengaffen/l-t
    targetRevision: HEAD
  syncPolicy:
    automated:
      prune: true 
      selfHeal: true 
      allowEmpty: false 
    syncOptions:
      - ServerSideApply=true
EOF
```
