+++
title =  "Kubernetes - the kubeadm way"
date = "2024-09-30"
+++

This is my guide / doc how to setup Kubernetes for educational purposes (including CKA/CKS exams) using plain kubeadm.

Here's how the cluster will look like:

- cloud provider of choice is [Hetzner](https://www.hetzner.com/de/)
- all traffic flows through the internet, no private networks
- Dual-stack IPv4/IPv6 networking
- plain ubuntu nodes as base OS
- Cilium as CNI of choice (with kube-proxy replacement)
- manual kubeadm to setup 

## Prerequisites

Before we dive into the details on how to setup, here are some prerequisites to met when you want to follow allong:

- [Create](https://accounts.hetzner.com/login) an account on Hetzner Cloud
- Create a project - call it something meaningful, I called mine `cucumber` (got the irony?)
- Add a cost alert to the project
- Some knowledge about Kubernetes (I'm not going to explain every basic thing in detail)

## Infrastructure

Let's quickly talk about the infrastructure I'm using for this cluster.

### Kubernetes API endpoint

For the Kubernetes API endpoint, you're often told to create a classic load balancer that will balance the traffic between your master nodes. While this is certainly a good idea, it's usually cost-expensive and not required at all for a multi-master setup. There are good alternatives using virtual IPs or even simpler: multiple DNS records.

I'm always bootstraping the cluster with a DNS record as official API endpoint, even if I only have one master node. This allows me to add more master nodes later on and simply create another entry for the same DNS record. This is actually a recommendation from the Kubernetes docs, to not use an IP as control-plane endpoint.

For this guide my endpoint will be `cucumber.technat.dev`

### Servers

Here are the servers I create:

| Location | Image        | Type  | Networks    | Name       | Labels                         |
| -------- | ------------ | ----- | ----------- | ----------- | ------------------------ |
| Helsinki | Ubuntu 24.04 | CAX11 | ipv4,ipv6 | hawk       | cluster=cucumber,role=master |
| Helsinki | Ubuntu 24.04 | CAX31 | ipv4,ipv6 | minion-01    | cluster=cucumber,role=worker |

Some notes **before** creating the servers:
- CAX type stands for Ampere (`arm64`) servers. In my experience, K8s and most containerized applications run fine on ARM, but you might want to change the type to something that uses `amd64`.
- Kubernetes requires you to have an odd number of master nodes. The golden number for most production setups is to have three master nodes. For lab purposes though, I usually only create one master to save cost (one is also an odd number).
- I don't add the servers to any private network, but only have a public IPv4 and IPv6 address for to use. 
- Dual-stack clusters will bring some complexity along that you can easily skip if you don't provision the servers with an IPv6 address.
- Use cloud-init to configure some common things faster (see next section).


#### Cloud-init

The servers are all created using the following cloud-init config, to speed up the configuration that is the same for all servers:

```yaml
#cloud-config <node>

locale: en_US.UTF-8
timezone: UTC
users:
  - name: technat
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL # Allow any operations using sudo
    lock_passwd: True # disable password login
    gecos: "Admin user created by cloud-init"
    shell: /bin/bash
    # use the following lines if you want to add an ssh-key for later use
    # ssh_authorized_keys:
    #  - "ssh-ed25519 ...."

package_update: true # Do a "apt update"
package_upgrade: true # Do a "apt upgrade"
packages: # Install some base-packages 
- vim
- git
- wget
- curl
- gnupg
- lsb-release
- dnsutils
- apt-transport-https
- ca-certificates

runcmd:
  # use the following lines to join the server to a tailnet, instead of using plain ssh 
  # - systemctl mask ssh
  # - curl -fsSL https://tailscale.com/install.sh | sh
  # - tailscale up --ssh --auth-key "<single-use-pre-approved-key>"
```

## Kubernetes OS Preparations

Now that the servers are up and running, we will need to prepare all nodes according to the [install kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) docs. This page links to multiple other pages for things you need to do as well, so here's the summary with all the steps in linear order.

### [Swap](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration)

The first of them is Swap. While you can nowadays use Swap on your nodes, it's still protected with a feature flag and I'm not confident to use it yet. So it must be completely disabled on all nodes. Ubuntu 24.04 on Hetzner does this by default. You can check this with `free -h` where you should see `0B` for total swap listed. 

### [Container Runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

The container runtime must be installed prior to cluster bootstrapping. There are various runtimes that fulfil the [CRI](https://kubernetes.io/docs/concepts/architecture/cri/). I'm using [containerd](https://containerd.io) for obvious reasons.

The steps to install containerd are more or less included in the Kubernetes documentation about container runtimes.

First let's enable IP forwarding in the kernel:

```console
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

Next we need to define which cgroup driver to use. I'm using cgroupv2 with systemd as the driver. Recent Ubuntu versions have cgroupv2 enabled by default and since they all boot using systemd, it's the best to stick with that for the rest as well. For other options and more explanations, take a look [here](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers).

To check whether cgroupv2 is enabeld on the host, do the following

```console
grep cgroup /proc/filesystems
# it should list multiple results, one beeing cgroup2
```

We will tell our Kubernetes components to use systemd as the cgroup driver when we get to the actually bootstrapping.

Now we install containerd using it's APT package from the [docker repository](https://docs.docker.com/engine/install/ubuntu/):

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get update
sudo apt install containerd.io -y
sudo apt-mark hold containerd.io # consider pinning the containerd version until you want to explicitly upgrade
```

Once containerd is installed, we have to tweak it's configuration a bit for it to work with Kubernetes. There are two important toggles:
- The CRI plugin inside containerd is usually disabled, since the containerd APT package is generic
- The systemd cgroup driver must explicitly be set in the config file

To achieve these changes, run the following commands:

```console
sudo sed -i 's/^disabled_plugins \=/\#disabled_plugins \=/g' /etc/containerd/config.toml # comments the line that disables the CRI plugin
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
```

To apply the configuration changes, restart containerd:

```console
sudo systemctl restart containerd
sudo systemctl status containerd
```

### [kubeadm, kubectl, kubelet](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

Next step is to get kubeadm, kubelet and kubectl onto the nodes. Here it's important to think twice which version we are going to install. According to the [version skew-policy](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy) it's the best to install the same version as the version of Kubernetes we are going to install later. In my case this is `v1.31.1`. 

Installing the tools using APT works as follows:

```console
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.31.1-1.1 kubeadm=1.31.1-1.1 kubectl=1.31.1-1.1
sudo apt-mark hold kubelet kubeadm kubectl
```

You may have noticed that the GPG key and repository is specific for Kubernetes 1.31. That means for upgrading Kubernetes, we would first have to get a new GPG key / repo added to APT before we can upgrade kubeadm which in turn can upgrade the cluster for us. I also set the APT packages on hold so that they don't get accidentially patched when we do a system upgrade.

### [kubelet config](https://github.com/hetznercloud/hcloud-cloud-controller-manager?tab=readme-ov-file#deployment)

Our cluster runs on Hetzner. For cloud-functionalities such as volumes and loadbalancers to be provisioned, we would later need to install a [cloud-controller-manager](https://kubernetes.io/docs/concepts/architecture/cloud-controller/) (CCM). This is a component unique to each cloud. But in order for this manager to know which nodes have been created in which cloud, we have to add a flag to the kubelet, that tells it, which cloud-provider it's node is running on.

In case of Hetzner the CCM is called [hcloud-cloud-controller-manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager) and the following drop-in config file for kubelet, instructs it, that it's from a cloud-provider called `external`: 

```console
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service.d/20-hcloud.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--cloud-provider=external"
EOF
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

If you are wondering why the cloud-provider is `external` and not something like `hcloud`, you should read [this blogpost](https://www.cncf.io/blog/2024/09/27/why-kubernetes-is-removing-in-tree-cloud-provider-integration-support-in-v1-31-and-how-it-can-affect-you/). It gives some details on the history of CCMs.

## [Kubernetes control-plane bootstrapping](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

Finally we can bootstrap the control-plane with kubeadm. For this we will create a kubeadm config that tells kubeadm how to bootstarp the cluster. It's important to remember that the control-plane is initalized on the first master node only. So this section is only executed on the first master node!

A default kubeadm config to work with can be generated using this command:

```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

We are going to change some things in this default config and delete most of the default options. All options can be found in the [reference docs](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/).

Here's the config for this guide:


```yaml
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd  # as defined earlier, the kubelet shall use systemd as cgroupv2 driver
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "65.109.160.197" # the public IPv4 address of the first master-node
nodeRegistration:
  kubeletExtraArgs:
    - name: "node-ip"
      value: "65.109.160.197,2a01:4f9:c012:b87a::1" # the public IPv4,IPv6 address of the first master node
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
proxy:
  disabled: true # read below
networking:
  serviceSubnet: 10.111.0.0/16,2001:db8::10:0/108 # read below
  podSubnet: 10.222.0.0/16,2001:db8:42:0::/56 # read below
kubernetesVersion: "v1.31.1" # the exact version to initialize as defined before
controlPlaneEndpoint: cucumber.technat.dev # the api-endpoint to advertise, as discussed in the intro
```

Some explanations to the config:
- `proxy.disabled`: [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) is the default tool to implement Kubernetes services, usually using iptables. I'm skipping the step that installs this tool, as I have a replacement for it that I personally preffer (more when we install our cni-plugin).
- `serviceSubnet`: the CIDR range (v4 and v6) used to assign IPs to Kubernetes Services, must be some private range
- `podSubnet`: the CIDR range (v4 and v6) used to assign IPs to Pods, must be some private range and for the v6 version at least `/64` or bigger.
- `controlPlaneEndpoint`: as discussed earlier, the endpoint where clients can reach the Kubernetes API, in our case, just a DNS record that keeps the topology flexible (could also be the IP of the only master-node we have, or a load-balancer IP)
- `node-ip` & `advertise-address`: kubelet and the kube-apiserver on this particualar node need to know which interface to use for binding to. In our case we only have a public NIC which we can specify. You could also specify another IP (from a private network or tailscale for example), and let Kubernetes use this network for communication.

Once we are happy with our config, we can initalize the cluster using this command on our first master-node:

```console
sudo kubeadm init --upload-certs --config kubeadm-config.yaml 
```


The output will clearly show when you had success initializing your control-plane and when not. Note that it might fail for the first time almost at the end, trying to reach the Kubernetes API using your DNS record and IPv6. That is most linux distros prefer AAAA records over A and the kube-apiserver seems to need a bit more time to be fully reachable on IPv6. If this happens to you, just delete the AAAA record, run `sudo kubeadm reset`, reinitialize and then readd the AAAA record again.

Once it worked, you will see a bunch of instructions how to join additional master and worker nodes, as well as some commands to get the kubeconfig from the master-node to start using kubectl:

```console
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Joining worker-nodes

Since I don't have any additional master nodes, I proceed with joining the worker nodes to my cluster. The instructions for this are almost what kubeadm's output tells you. But we have to tweak some things a bit and thus create a join-config for our worker-node:

```yaml
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd  # as defined earlier, the kubelet shall use systemd as cgroupv2 driver
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
nodeRegistration:
  kubeletExtraArgs:
    - name: "node-ip"
      value: "<ipv4 of machine>,<ipv6 of machine>" # the public IPv4,IPv6 address of our worker node
discovery:
  bootstrapToken: 
    apiServerEndpoint: cucumber.technat.dev:6443 # the defined DNS endpoint clients shall use for the Kubernetes API, port 6443 is the implicit default 
    token: j1d3i2.ji36qbzjw7gc0t0f # read below
    caCertHashes: ["sha256:5856a582eac876282373afb0eeb07f862e4bcb2d2ffe108c70ffcc48d97d1356"] # read below
```

The `token` and `caCertHashes` are the required credentials to join this worker node to the control-plane. To get these informations, you can either look at kubeadm's output on your master-node or if you lost that already, run the following command on the first master-node to get some new credentials:

```console
kubeadm token create --print-join-command
```

This will give you the information you need to join the node. Add the information to the join-config and join the node using:

```console
sudo kubeadm join --config join-config.yaml
```

## Installing a CNI-plugin

By now you might see that a `kubectl get nodes` lists the nodes in a `Not Ready` state. This is due to the missing CNI.

So to install and verify the installation of Cilium, before we continue to add nodes, we need `helm` and the `cilium` cli:

```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

cilium install —set “ipam.mode=kubernetes” —set “ipv6.enabled=true”
```

With those tools we can install Cilium as a helm chart:

```
helm repo add cilium https://helm.cilium.io/
helm upgrade -i cilium cilium/cilium -n kube-system -f cilium-values.yaml
```

See that `-f cilium-values.yaml` at the end of the last command? That's the config for cilium. It looks like that for me:

```yaml
rollOutCiliumPods: true
priorityClassName: "system-node-critical"
annotateK8sNode: true
encryption:
  enabled: true
  type: wireguard
operator:
  replicas: 1
l7Proxy: false # not compatible with kube-proxy replacement (but better double-check if that's still true)
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
    rollOutPods: true
policyEnforcementMode: "always" # you should be using that but it will get you into a lot of work ;)
kubeProxyReplacement: "strict"
k8sServiceHost: "admin.technat.dev"
k8sServicePort: "6443"
```

You can get the default config using `helm show values cilium/cilium` and then customize this to your needs.

### Join master nodes

To join the other master nodes to the cluster, copy the command from our init output and run it on the other master nodes:

```bash
kubeadm join admin.technat.dev:6443 --token blfexx.ei3cp7hozu27ebpg \
       --discovery-token-ca-cert-hash sha256:fd3c9d6595ed7de9cd3f5c66435cbfcbbfddcc782e37c16a4824fff04c23430d \
       --control-plane --certificate-key b5c99a41b7f16ef635619c6e972d29ec4ac921dff85839815b51652ab39447b7 \
       -f kubeadm-config.yaml
```

### Join Workers

Now any number of worker nodes can be joined using the command:

```
sudo kubeadm join admin.technat.dev:6443 --token dt4vt3.c833o3z5fcdmbj37 \
	--discovery-token-ca-cert-hash sha256:dd3d4f4a3bee384d4ebe539d24d38bb11bcfb765a066b03fb8b79676a33cc902
```

Note: the join-token expires after some time. If this happens you need to create a new one using `kubeadm token create --print-join-command`.

## Post-Setup

Bootstraped and now? There are a few things we should think about now.

### kubectl completion

Very very helpful: [kubectl completion](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/)

### etcdctl

The package `etcd-client` provides us with an `etcdctl` that can be used to interact with the etcd cluster that backes the control-plane. However the tool needs some flags to be passed into for usage. I recommend you save yourself the following as an alias:

```
ETCDCTL_API=3 sudo etcdctl --endpoints https://localhost:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```

### Taints

Make sure the master nodes have their taint on them so that only control-plane components are scheduled on them:

```
k taint nodes master-1 node-role.kubernetes.io/master:NoSchedule
```

### type: LoadBalancer

You may have noticied that I installed my K8s cluster on Hetzner. To interact with Hetzner Cloud you would now need the [Hcloud CCM](https://github.com/hetznercloud/hcloud-cloud-controller-manager) in order to be able to provision LoadBalancers for Services.

See their repo for a quickstart how to install the manager and note that the prerequisite with the cloud-provider flag in the kubelet has already been done.
