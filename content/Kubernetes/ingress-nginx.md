---
title: "ingress-nginx"
---
## References

\- [https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)

\- [https://kubernetes.github.io/ingress-nginx/deploy/#using-helm](https://kubernetes.github.io/ingress-nginx/deploy/#using-helm)

## Install

```plaintext
k create ns ingress-nginx
helm upgrade ingress-nginx -i -n ingress-nginx ingress-nginx/ingress-nginx -f helm-ingress-nginx.yaml
```

## External Access

To have external access add port-forwarding rules in Firewall to forward 80/443 to ingress-nginx

## Files

Custom Values: [helm-ingress-nginx.yaml](/Kubernetes/helm-ingress-nginx.yaml)
