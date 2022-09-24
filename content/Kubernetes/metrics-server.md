---
title: "metrics-server"
---

See https://github.com/kubernetes-sigs/metrics-server/blob/master/charts/metrics-server/README.md

## Install

```plaintext
helm upgrade -i metrics-server metrics-server/metrics-server -f metrics-server-values.yaml
```

## Files

Custom Values: [metrics-server-values.yaml](/Kubernetes/metrics-server-values.yaml)
