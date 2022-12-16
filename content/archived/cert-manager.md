+++
title = "cert-manager"
+++
## Installation

```plaintext
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm show values jetstack/cert-manager > cert-manager-values.yaml
kubectl create ns cert-manager
helm upgrade -i cert-manager -n cert-manager jetstack/cert-manager -f cert-manager-values.yaml
```

## Let's Encrypt Issuer

```plaintext
kubectl apply -f letsencrypt-staging.yaml
kubectl apply -f letsencrypt-production.yaml
```

## Resources

\- [https://cert-manager.io/docs/tutorials/acme/ingress/](https://cert-manager.io/docs/tutorials/acme/ingress/)

## Files

Custom Values: [cert-manager-values.yaml](/Kubernetes/cert-manager-values.yaml)

Let's Encrypt Staging Issuer: [letsencrypt-staging.yaml](/Kubernetes/letsencrypt-staging.yaml)

Let's Encrypt Production Issuer: [letsencrypt-production.yaml](/Kubernetes/letsencrypt-production.yaml)
