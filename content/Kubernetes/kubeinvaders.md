---
title: "kubeinvaders"
---

## Preparations

```plaintext
helm repo add kubeinvaders https://lucky-sideburn.github.io/helm-charts
```

## Install

```plaintext
helm upgrade -i kubeinvaders -n kubeinvaders --create-namespace kubeinvaders/kubeinvaders -f kubeinvaders-values.yaml
```

## Files

Custom Values: [kubeinvaders-values.yaml](/Kubernetes/kubeinvaders-values.yaml)
