+++
title =  "heimdall"
+++
Chart: https://artifacthub.io/packages/helm/k8s-at-home/heimdall

## Installation

```plaintext
helm repo add k8s-at-home https://k8s-at-home.com/charts/
helm repo update
helm show values k8s-at-home/heimdall > heimdall-values.yaml
helm upgrade -i heimdall  --create-namespace -n heimdall k8s-at-home/heimdall -f heimdall-values.yaml
```

## Files

Custom heimdall values: [heimdall-values.yaml](/Kubernetes/heimdall-values.yaml)

Service for Heimdall: [heimdall-lb.yaml](/Kubernetes/heimdall-lb.yaml)
