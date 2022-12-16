+++
title =  "rook-ceph"
+++

Maybe you know rook.io. Rook is a storage operator.

Rook can configure Ceph clusters based on CRDS. Ceph provides Block, File and Object storage.

The workers nodes used in this values file have each an empty 50GB disk attached. By default rook will only take disks that are unused.

## rook-ceph chart

We start by installing the rook-operator which gives us the base to deploy ceph later on:

```plaintext
helm repo add rook-release https://charts.rook.io/release
kubectl create namespace rook-ceph
helm show values rook-release/rook-ceph > rook-values.yaml
```

Let's change the values a bit and make a custom config for it. See the `rook-ceph-values.yaml`

And finally install the rook operator:

```plaintext
helm upgrade -i rook-ceph rook-release/rook-ceph -n rook-ceph -f rook-ceph-values.yaml
```

We ignore the PSP deprication warnings for now.

## rook-ceph-cluster chart

The rook operator manages ceph clusters, so let's create one using the official rook-ceph-cluster chart.

```plaintext
helm show values rook-release/rook-ceph-cluster > original-values-cluster.yaml
```

We modify the values a bit to match our setup. See the `rook-ceph-cluster-values.yaml` file.

Some things to note here:

\- By default rook is configured to keep data on disks when an admin actidentially calls the kubeapi to delete the CephCluster resource. To delete the data as well, confirmation needs to set on the resource, e.q you need to patch the resource first and then delete it.

\- We automatically use any disk on our worker nodes which are not used (that excludes the os diks)

\- if we got an ingress controller we will expose the dashboard using ingress, until then, a loadbalancer will do fine.

\- all storageclasses have the reclaimPolicy of Retain, meaining that PVC's that are deleted will result in PVs laying around until they are manually deleted.

So we can now install the ceph cluster:

```plaintext
helm upgrade -i  rook-ceph-cluster rook-release/rook-ceph-cluster -f rook-ceph-cluster-values.yaml -n rook-ceph
```

Note: This takes some minutes (mine took between 10 and 30 minutes) to get fully up and running.

### Ceph Dashboard

The ceph dashboard is enabled, but we haven't set an Ingress Route yet. So let's create a svc in the meantime:

```plaintext
apiVersion: v1
kind: Service
metadata:
 name: rook-ceph-mgr-dashboard-external-https
 namespace: rook-ceph
 labels:
   app: rook-ceph-mgr
   rook_cluster: rook-ceph
spec:
 ports:
 - name: dashboard
   port: 80
   protocol: TCP
   targetPort: 7000
 selector:
   app: rook-ceph-mgr
   rook_cluster: rook-ceph
 sessionAffinity: None
 type: LoadBalancer
```

The loadbalancer ip: `kubectl -n rook-ceph get svc rook-ceph-mgr-dashboard-external-https -o yaml | grep -A "loadbalancer"`

The password: `kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo`

Then we can reach the dashboard: `https://192.168.100.100:7000`

## Files

Dashboard SVC: [ceph-dashboard-svc.yaml](/posts/ceph-dashboard-svc.yaml)

Custom Cluster Values: [rook-ceph-cluster-values.yaml](/posts/rook-ceph-cluster-values.yaml)

Custom Operator Values: [rook-ceph-values.yaml](/posts/rook-ceph-values.yaml)

OSD-Purge: [osd-purge.yaml](/posts/osd-purge.yaml)
