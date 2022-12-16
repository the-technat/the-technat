+++
title =  "gitlab-runners"
+++
Chart: https://artifacthub.io/packages/helm/gitlab/gitlab-runner

## Preparations

```plaintext
helm repo add gitlab http://charts.gitlab.io/
helm show values gitlab/gitlab-runner > gitlab-runner-values.yaml
kubectl create ns gitlab-runners
```

## Configure

### Cache

Apply the CRD to create the cache:

```plaintext
kubectl apply -f cache-bucket.yaml
```

Get the information from the configmap and fill it into the helm values:

```plaintext
kubectl get configmap gitlab-cache -o yaml
```

For the credentials we have to create a new secret based on the current one that rook-ceph created:

```plaintext
kubectl create secret generic s3access \
   --from-literal=accesskey="" \
   --from-literal=secretkey=""
```

## Install

```plaintext
helm upgrade -i k8s-at-hetzner-runner -n gitlab-runners gitlab/gitlab-runner -f k8s-at-hetzner-runner-values.yaml
```

## Files

Cache Bucket: [cache-bucket.yaml](/posts/cache-bucket.yaml)

Custom Values: [k8s-at-hetzner-runner-values.yaml](/posts/k8s-at-hetzner-runner-values.yaml)
