+++
title =  "Kubernetes - the kubeadm way"
date = "2022-11-29"
+++
This is my current approach on how to setup Kubernetes for labing purposes. I'm not a fan of automation when It comes to trying things out, so my setup is completely manual using kubeadm.

Here are some other aspects of my setup that will be part of this guide:

- My Cloud Provider of choice is currently [Hetzner](https://www.hetzner.com/de/)
- Public facing K8s - no traffic going over private networks
- My CNI of choice is currently Cilium with pod to pod encryption using wireguard

## Prerequisites

Before we dive into the details on how to setup, here are some prerequisites to met when you want to follow allong:

- [Create](https://accounts.hetzner.com/login) an Account on Hetzner Cloud
- Create a Project - call it something meaningful (like cucumber)
- Add a ssh-key to the project and mark it as the default key

## Infrastructure

Let's quickly talk about the infrastructure I'm using for this guide.

First start with everything you need outside the servers:

- A DNS record pointing to all of your master nodes
- Placement Groups: one for workers, one for masters
- Firewall Rules: one for masters, one for workers using the following rules:
  - Note: this list is not final by any way, kubeadm requires more ports to be open
  - masters:

  | Type     | Source          | Protocol  | Port    |
  | -------- | --------------- | --------- | --------|
  | Incoming | 0.0.0.0/0, ::/0 | TCP       | 59245   |
  | Incoming | 0.0.0.0/0, ::/0 | ICMP      | -       |
  | Incoming | 0.0.0.0/0, ::/0 | TCP       | 6443    |
  | Incoming | masters         | TCP      | 2379-2380 |
  | Incoming | workers         | TCP      | 2379-2380 |
  | Incoming | masters         | TCP       | 10250 |
  | Incoming | masters         | TCP       | 10259 |
  | Incoming | masters         | TCP       | 10257 |
  | Incoming | masters         | TCP       | 4240 |
  | Incoming | workers         | TCP       | 4240 |
  | Incoming | workers         | UDP       | 8472 |
  | Incoming | masters         | UDP       | 8472 |
  | Incoming | workers         | UDP       | 51871 |
  | Incoming | masters         | UDP       | 51871 |

  - workers:

  | Type     | Source          | Protocol  | Port    |
  | -------- | --------------- | ----------| --------|
  | Incoming | 0.0.0.0/0, ::/0 | TCP       | 59245   |
  | Incoming | 0.0.0.0/0, ::/0 | ICMP      | -       |
  | Incoming | 0.0.0.0/0, ::/0 | TCP       | 30000 - 32768 |
  | Incoming | 0.0.0.0/0, ::/0 | UDP       | 30000 - 32768 |
  | Incoming | masters         | TCP       | 10250 |
  | Incoming | workers         | TCP       | 4240 |
  | Incoming | masters         | TCP       | 4240 |
  | Incoming | masters         | UDP       | 8472 |
  | Incoming | workers         | UDP       | 8472 |
  | Incoming | workers         | UDP       | 51871 |
  | Incoming | masters         | UDP       | 51871 |

Kubernetes requires you to have an odd number of master nodes, for true HA. Also many applications you may install require three replicas that are spread accross different nodes, so at least three worker nodes are ideal too. So here's a list of all the servers I'm going to create:

| Location | Image        | Type  | Networks    | Placement Group | Backups | Name       | Labels                         |
| -------- | ------------ | ----- | ----------- | --------------- | ------- | ----------- | ------------------------ |
| Helsinki | Ubuntu 22.04 | CPX11 | k8s, ipv4 | masters         | true    | lion       | cluster=technat_ch,master=true |
| Helsinki | Ubuntu 22.04 | CPX11 | k8s, ipv4 | masters         | true    | giraffe    | cluster=technat_ch,master=true |
| Helsinki | Ubuntu 22.04 | CPX11 | k8s, ipv4 | masters         | true    | gorilla    | cluster=technat_ch,master=true |
| Helsinki | Ubuntu 22.04 | CX21  | k8s, ipv4 | workers         | false   | burro-01   | cluster=technat_ch,worker=true |
| Helsinki | Ubuntu 22.04 | CX21  | k8s, ipv4 | workers         | false   | burro-02   | cluster=technat_ch,worker=true |
| Helsinki | Ubuntu 22.04 | CX21  | k8s, ipv4 | workers         | false   | burro-03   | cluster=technat_ch,worker=true |

Some notes:
- All my servers are initalized with a [cloud-init](https://technat.ch/linux/cloud_init) config to speed up.
- Expect the size and number of workers to change over time.
- That's also the reason why I don't add DNS records for my nodes. They should be as ephemeral as possible.
- However you should have a DNS record for your kubeapi where you have added all master node IPs as valid answers (e.g three records for the same domain name). This is the simplest way to avoid an external load balancer in front of your control plane (of course you could do that too and just forward port 6443 to all the master nodes)

Oh and a word about networking too: After trying different szenarious, I realized that dual-stack isn't an option, as it's just too complicated and error prone. IPv6 only would be my desired setup, but unfortunately that currently doesn't work (not even Github can be rechached over IPv6). So the only solution I found reliable was IPv4 only.

## OS Preparations

Now that the servers are up and running, we will need to prepare all nodes according to the [install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) docs.

### Swap

The first of them is Swap. Swap must be completly disabled on all nodes. Mine already had that, make sure yours have it disabled too.

### Container Runtime

The [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) must be installed previous to the cluster bootstraping. There are various runtimes that fulfil the CRI. I'm using Containerd as its simple and minimal.

The steps to install can be checked in the linked documentation or here.

Frist let's load the br_netfilter module and set some forwarding settings:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

then we need to add the docker repository in order to install containerd:

```bash
sudo apt-get update
sudo apt-get install \
  ca-certificates \
  curl \
  gnupg \
  lsb-release -y

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


sudo apt update
sudo apt install containerd -y
```

Now containerd should be installed, you can check with `ctr version` to see if you can access it (some errors are fine, you just want to see a version number)

### Cgroups

The [docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers) tell you a lot about cgroups and cgroup drivers. Take a look there if you want to understand why it's needed and what it's doing. For the cluster setup, it's just important to agree on one variant and enforce this in all components. I'm using the `systemd` driver and cgroup v2. To enforce this on Ubunut, we need to add a kernel parameter in the `GRUB_CMDLINE_LINUX` directive of `/etc/default/grub`:

```bash
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"
```

And run a `sudo update-grub` followed by a reboot. Then we are sure, the system uses the correct cgroup driver and version.

We repeat the procedure and tell containerd to use systemd's cgroup as well by modifiying `/etc/containerd/config.toml`:

```bash
cat <<EOF | sudo tee -a /etc/containerd/config.toml
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
EOF

sudo sed -i 's/^disabled_plugins \=/\#disabled_plugins \=/g' /etc/containerd/config.toml
```

Note: it's possible that now all systems have the default configuration already in place for containerd. If you get an error saying the directory doesn't exist, try the following and repeat the above commands:

```bash
sudo mkdir -p /etc/containerd
```

And do a final restart of containerd:

```
sudo systemctl restart containerd
sudo systemctl status containerd
```

### kubeadm, kubectl, kubelet

Next step is to get [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) (cluster bootstraping tool), kubectl and kubelet (cluster agent on systems) installed using package managers.

But to start we need to ensure some network settings are given. They should actually already be set when you used the above commands to install containerd.

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

then we can add the necessary repository:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Note: We mark kubelet, kubeadm and kubectl on hold so that they don't get updated when updating other system packages. This is because they have to follow the versioning of Kubernetes and have some versions of room to differ. So it's best to keep them at the same version as Kubernetes itself and update them when you update Kubernetes.

Make sure you set the `cgroupDriver: systemd` when bootstraping the cluster so that the kubelet uses the same cgroup driver as the container runtime.

### CNI considerations

One last thing before we can bootstrap our cluster is to choose a CNI for kubernetes. Although we could wait with that until kubeadm init is started we may need to set some flags in the init config to work with our CNI.

As said I'm using cilium. Cilium has two options for IPAM, one beeing kubernetes and one beeing cluster-scoped cilium mode. I'm using the later and also use the kube-proxy replacement of cilium which means I can completly ignore the flags for service and pod Subnets in kubeadm.

## Bootstraping

Finally we can [bootstrap](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) our cluster with kubeadm. For this we will create a kubeadm config to customize our installation:

Let's get the default config with:

```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

We are going to change some things in this default config. All options can be found in the [reference docs](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3/#kubeadm-k8s-io-v1beta3). My file usually looks like that:

```yaml
+++
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
clusterName: technat.k8s
networking:
  dnsDomain: admin.alleaffengaffen.k8s # not necessary: custom internal dns domain
controlPlaneEndpoint: admin.alleaffengaffen.ch:6443 # DNs record pointing to your master nodes
+++
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd # must match the value you set for containerd
```

Some of the options are reasonable, some are only cosmetics and some are performance tweacks.
If you think the config looks good for you, you can start the initial bootstrap using the following command (remove the `--skip-phase=addon/kube-proxy` argument if you're not using cilium):

```bash
sudo kubeadm init --upload-certs --config kubeadm-config.yaml --skip-phases=addon/kube-proxy
```

The output will clearly show when you had success initializing your control-plane and when not. Expect this to fail the first time. In my experience custom configs always lead to an error in the first place. But fortunately the commands shows you where to start debugging.

If it worked, you can then get the kubeconfig from the kubernetes directory to your regular user and start kubeing:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

For the root user, you can also just export the location of the kubeconfig:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

### Deploy CNI

By now you might see that a `kubectl get nodes` lists the first master in a `Not Ready` state. This is due to the missing CNI.

So to install and verify the installation of cilium, before we continue to add nodes, we need `helm` and the `cilium` cli:

```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

With those tools we can install cilium as a helm chart:

```
helm repo add cilium https://helm.cilium.io/
helm upgrade -i cilium cilium/cilium -n kube-system -f cilium-values.yaml
```

See that `-f cilium-values.yaml` at the end of the last command? That's the config for cilium. It looks like that for me:

```yaml
rollOutCiliumPods: true

priorityClassName: "system-node-critical"

annotateK8sNode: true

containerRuntime:
  integration: containerd
  socketPath: /var/run/containerd/containerd.sock

encryption:
  enabled: true
  type: wireguard

operator:
  replicas: 1

l7Proxy: false

hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    rollOutPods: true

policyEnforcementMode: "none"

kubeProxyReplacement: "strict"
k8sServiceHost: "admin.alleaffengaffen.ch"
k8sServicePort: "6443"
```

You can get the default config using `helm show values cilium/cilium` and then customize this to your needs.


### Join master nodes

To join the other master nodes to the cluster, copy the command from our init output and run it on the other master nodes:

```bash
kubeadm join admin.alleaffengaffen.ch:6443 --token blfexx.ei3cp7hozu27ebpg \
       --discovery-token-ca-cert-hash sha256:fd3c9d6595ed7de9cd3f5c66435cbfcbbfddcc782e37c16a4824fff04c23430d \
       --control-plane --certificate-key b5c99a41b7f16ef635619c6e972d29ec4ac921dff85839815b51652ab39447b7 \
       -f kubeadm-config.yaml
```

### Join Workers

Now any number of worker nodes can be joined using the command:

```
sudo kubeadm join admin.alleaffengaffen.ch:6443 --token dt4vt3.c833o3z5fcdmbj37 \
	--discovery-token-ca-cert-hash sha256:dd3d4f4a3bee384d4ebe539d24d38bb11bcfb765a066b03fb8b79676a33cc902
```

## Post-Setup

Bootstraped and now? There are a few things we should think about now.

### kubectl completion

Very very helpful: [kubectl completion](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/)

### etcdctl

The package `etcd-client` provides us with a `etcdctl` that can be used to interact with the etcd cluster that backes the control-plane. However the tool needs some flags to be passed into for usage. I recommend you save yourself the following as an alias:

```
ETCDCTL_API=3 sudo etcdctl --endpoints https://192.168.112.51:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

### Taints

Make sure the master nodes have their taint on them so that only control-plane components are scheduled on them:

```
k taint nodes lion node-role.kubernetes.io/master:NoSchedule
```

### type: LoadBalancer

You may have noticied that I installed my K8s cluster on Hetzner. But as you might now, the cloud controller manager of K8s does not support interacting with Hetzner to provision LoadBalancers for Services.

This is the point where the [hcloud-cloud-controller-manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager) comes into play. This is a simple controller that talks to the Hetzner API and enriches your nods with some labels about the typology and provisions LBs if desired.

See the repo for a quickstart how to install the manager.