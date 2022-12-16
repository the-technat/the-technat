+++
title =  "external-dns"
+++
## Prepare

See https://github.com/kubernetes-sigs/external-dns/tree/master/charts/external-dns

## Install

```plaintext
kubectl create ns external-dns
helm upgrade -i external-dns -n external-dns external-dns/external-dns --set env.0.value=<APITOKEN_HERE> -f external-dns-values.yaml
```

## Files

Custom Values: [external-dns-values.yaml](/posts/external-dns-values.yaml)
