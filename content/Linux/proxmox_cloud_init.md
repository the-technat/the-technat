---
title: "proxmox_cloud_init"
---
This page describes how to create a cloud-init ready vm template in proxmox

## Create initial VM
| Key | Value |
|-----|-------|
| ID | 9000 |
| Name | debian-11-generic |
| ISO | debian-11.0.0-adm64.netinst.iso |
| Qemu-Agent | 1|
| BIOS | OVMF (UEFI) |
| EFI Disk | 1, local-zfs |
| Machine type | q35 |
| Graphic card | SPICE |
| OS Disk | local-zfs, 32gb, io-thread=1,discard=1 |
| Cores | 2 |
| Memory | 2048MB |
| Net0 | vmbr4, firewall=0 |

### K8S
| Key | Value |
|-----|-------|
| ID | 9008 |
| Name | debian-11-k8s |
| ISO | debian-11.0.0-adm64.netinst.iso |
| Qemu-Agent | 1 |
| Graphic card | SPICE |
| BIOS | OVMF (UEFI) |
| EFI Disk | 1, local-zfs |
| Machine type | q35 |
| OS Disk | local-zfs, 32gb, io-thread=1,discard=1 |
| Cores | 2 |
| Memory | 4096MB |
| Net0 | vmbr4, firewall=0 |


## Initial setup
Values used when installing the OS for the first time:

| Key | Value |
|-----|-------|
| Lanugage | English |
| Location | Europe/Finland |
| Locale | en_US-UTF-8 |
| Keyboard | American english |
| IP Type | Manual |
| IP | 192.168.114.2/24 |
| Gateway | 192.168.114.1 |
| DNS | 192.168.111.11 |
| Hostname | debian-11-generic |
| Domain Name | technat.lab |
| root PW | random, remember it |
| Default admin | technat |
| Default admin pw | random, remember it |
| Disk Paritioning | manual, /dev/sda1=efi 500mb,/dev/sda2=lvm-pv |
| VG | os, use os-partition |
| LV-root | 10G,ext4,mount=/ |
| LV-var | 5G,ext4,mount=/var |
| LV-tmp | 5G,ext4,mount=/tmp |
| LV-home | 10G,ext4,mount=/home |
| LV-swap | 2G,swap |
| Software selection | only ssh |

### K8S
Values used for the k8s template:
| Key | Value |
|-----|-------|
| Lanugage | English |
| Location | Europe/Finland |
| Locale | en_US-UTF-8 |
| Keyboard | American english |
| IP Type | Manual
| IP | 192.168.114.3/24 |
| Gateway | 192.168.114.1 |
| DNS | 192.168.111.11 |
| Hostname | debian-11-k8s |
| Domain Name | technat.lab |
| root PW | random, remember it |
| Default admin | technat |
| Default admin pw | random, remember it |
| Disk Paritioning | manual, /dev/sda1=efi 500mb,/dev/sda2=lvm-pv |
| VG | os, use os-partition |
| LV-root | 10G,ext4,mount=/ |
| LV-var | 10G,ext4,mount=/var |
| LV-tmp | 7G,ext4,mount=/tmp |
| LV-home | 5G,ext4,mount=/home |
| Swap | None (ignore) |
| Software selection | only ssh |

Reboot the machine and ssh into it with the ip and admin user set during installation.

## System Preparation
As mentioned in the [Cloud-Init FAQ](https://pve.proxmox.com/wiki/Cloud-Init_FAQ) of Proxmox it's best practise to configure your system before continuing with cloud-init. There are several things mentioned and there are some things I like to do. So let's go through some things

### Predictable Network Interface Names
The docs say that for compatibility it's preferred to have NIC names `eth0`, `eth1` and so on instead of naming them according to their actual physical hardware.

Read more about the feature [here](https://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/).

We disable the names with the kernel parameter `net.ifnames=0`. Set it in `/etc/default/grub` and update the config:
```bash
su - root # Must be root if sudo is not installed to run this command
sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet"/GRUB_CMDLINE_LINUX_DEFAULT="net.ifnames=0 quiet"/g' /etc/default/grub
update-grub
reboot
```

After the reboot you should see in the output of `ip link` that the names have changed.
You might need to reapply your network config as nic names are hardcoded in `/etc/network/interfaces`.

### Default Packages
Next is to install some default packages:
```bash
su - root
apt install vim bash-completion sudo curl wget git stow qemu-guest-agent resolvconf -y
```

Note: resolvconf is important as it configures dns according to `/etc/network/interfaces`.

### Admin User
During the installation we created the user `technat`. This will be our admin user we login in the future. But the user needs some configs.

First do the following:
```bash
su - root
echo "technat ALL=(ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers
usermod -aG sudo technat
exit
bash
```

Now we are able to run commands using sudo without beeing prompted for our password.

### APT Mirrors
Hetnzer provides some [APT Mirrors](https://docs.hetzner.com/robot/dedicated-server/operating-systems/hetzner-aptitude-mirror/) for Debian. As we can reach them much faster than others we replace the content of `/etc/apt/sources.list` with the following:
```
#Packages and Security Updates from the Hetzner Debian Mirror
deb https://mirror.hetzner.com/debian/packages  bullseye           main contrib non-free
deb https://mirror.hetzner.com/debian/packages  bullseye-updates   main contrib non-free
deb https://mirror.hetzner.com/debian/security  bullseye-security  main contrib non-free
deb https://mirror.hetzner.com/debian/packages  bullseye-backports main contrib non-free

deb http://deb.debian.org/debian               bullseye          main non-free contrib
deb http://deb.debian.org/debian               bullseye-updates  main non-free contrib
deb http://security.debian.org/debian-security bullseye-security main contrib non-free
```

### SSH
We adjust the ssh config `/etc/ssh/sshd_config` a bit:
```
PermitRootLogin no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
```

Restart the sshd service.

### Env Vars
Two env vars to set in `/etc/environment`:

```
EDITOR=vim
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games"
```

### NTP
Configure ntp and the timezone:
```bash
sudo timedatectl set-ntp true
sudo timedatectl set-timezone Europe/Helsinki
```

### Misc
- Run `sudo systemctl enable fstrim.timer` to enable periodical trim support
- Uncomment the `ll` alias for root and technat
- Clone and stow the server-vimrc file

## Configure Cloud-init
Once the system has reasonable settings we can continue to install and configure cloud-init:

```bash
sudo apt install cloud-init -y
```

The "cloud" config is located at `/etc/cloud/cloud.cfg`. We make a backup of this file and start editing the real file.

### Config source
[Reference Docs](https://pve.proxmox.com/wiki/Cloud-Init_FAQ)
First we define that there are only two possibles sources for a cloud-init config (the ones proxmox supports).

Edit `/etc/cloud/cloud.cfg.d/99_pve.cfg` and add:
```
datasource_list: [ NoCloud, ConfigDrive ]
```

### Config paramters
Then we add, change, delete the following keys in the config file:
- Remove the `default` user entry in the `users` section.
- Set `disable_root` to false (already done)
- Add the following keys at root level to ensure the system will upgrade itself when booted for the first time:
```yaml
package_update: true
package_upgrade: true
package_reboot_if_required: true
```
- `manage_etc_hosts: true`: Add this somewhere at root level so that `/etc/hosts` is updated with new hostname when template is cloned
- Remove `byobu` from the list of `cloud_config_modules`
- Add a section to generate a random password for root:
```
chpasswd:
  list:
  - root:RANDOM
```
- Remove `puppet,chef,salt-minion,mcollective` from `cloud_final_modules`
- Comment the entire `system_info.package_mirrors` section to prevent it from overriding our configured Hetzner Mirrors
- Add `#cloud-config` at the top of the file
- Add `date > /etc/birth_date` to the `bootcmd` section as a reference when this machine was first booted
- Add a section `ca-certs` at root level and add our technat.lab CA cert:
```yaml
ca-certs:
  trusted:
	- |
  	-----BEGIN CERTIFICATE-----
    MIIEPzCCAyegAwIBAgIBADANBgkqhkiG9w0BAQsFADBzMRMwEQYDVQQDEwpUZWNo
    bmF0IENBMQswCQYDVQQGEwJDSDENMAsGA1UECBMEQmVybjEYMBYGA1UEBxMPSGVy
    em9nZW5idWNoc2VlMRQwEgYDVQQKEwtUZWNobmF0IExhYjEQMA4GA1UECxMHY29j
    b251dDAeFw0yMTAxMTkxOTUyMjJaFw0zMTAxMTcxOTUyMjJaMHMxEzARBgNVBAMT
    ClRlY2huYXQgQ0ExCzAJBgNVBAYTAkNIMQ0wCwYDVQQIEwRCZXJuMRgwFgYDVQQH
    Ew9IZXJ6b2dlbmJ1Y2hzZWUxFDASBgNVBAoTC1RlY2huYXQgTGFiMRAwDgYDVQQL
    Ewdjb2NvbnV0MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvsxtZvoF
    7AMB+2zzwFb2sPy1IW2PjFLrSy6QGRS9ZMMldQhMh2bCWBTqcgFFErbk2AY8pKjf
    Ldw+OJDWkCjGdNyMM7L9HlBvQdOq9BHrWW90LCmrFhB6f/RFesMfDCsToMEH0L6E
  	Ef9MzZM+7Puydk7ETZR6twexy4UqZdm3CXNGJPMalMxUa0ewxsofx1x/0JnsCX3o
  	xoMXXrZP6/VsegjyQg7xn4ZtELt8cLhqUg5J2CNLezC2Mb5F809rk5xpGd0W2sMd
  	B80xlF+2HY3BJFnoAseyN5WKQ2Z3pe0R/PU4qNqmX1hUhXMZ+CLGb73uDkftHECG
  	gT3aTwOfMPxXOQIDAQABo4HdMIHaMB0GA1UdDgQWBBSNDGOQ/fjukvISuMe/vMgN
  	ppsrSTCBnQYDVR0jBIGVMIGSgBSNDGOQ/fjukvISuMe/vMgNppsrSaF3pHUwczET
  	MBEGA1UEAxMKVGVjaG5hdCBDQTELMAkGA1UEBhMCQ0gxDTALBgNVBAgTBEJlcm4x
  	GDAWBgNVBAcTD0hlcnpvZ2VuYnVjaHNlZTEUMBIGA1UEChMLVGVjaG5hdCBMYWIx
  	EDAOBgNVBAsTB2NvY29udXSCAQAwDAYDVR0TBAUwAwEB/zALBgNVHQ8EBAMCAQYw
  	DQYJKoZIhvcNAQELBQADggEBAAbZTKfB8zo6pe5FlOptGkIz+hVvKfLr1wJdbz4o
  	HI833MFDxKDtmB5M53Bq26eRAAs+4bWFr5JWTL3qPDbJ/vb0lZNJo6Ie/L1INY43
  	1CqbZjAq7xf8fYzEUdwMLLi7P3kh3eyJS+iOYno44ZwZxqTUmBZANat6jvdLsItm
  	5Cj1wv3D1klWBojXbuUw35AcALNvGJ3ORiuaIAOOQDn0RpHf+wQah7B10RIzRmQA
  	g3BZdxaK0orHjH29wrSDwKC+bPPPacT8bl7zY99zvNetoh4qrY/0HzCvAc9y6wt/
  	Hbc6Zcr+zGcgxCKkJMIIgtqkMzHIQh6rBrG5x84B+UTZEFU=
  	-----END CERTIFICATE-----
```

## K8S Configs
For the k8s template additionally configuration is done manually:

#### Container Runtime
The standard for technat.lab is to use containerd as runtime

[Reference Docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)


```bash
sed -i 'GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"/g' /etc/default/grub

sudo update-grub

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

sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release -y

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install containerd.io -y

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd
```

In order for containerd to use the systemd cgroup driver we set the following in the config:
```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```


#### kubeadm, kubelet, kubectl
[Reference Docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

Some prerequisites:
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

Then configure repo and install packages:
```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl -y

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
```

#### Kubectl completion
[Reference Docs](https://kubernetes.io/docs/tasks/tools/included/optional-kubectl-configs-bash-linux/)

```bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc
```

## Networking
Cloud-init will write a network file in the `/etc/network/interfaces.d/` directory. So we shouldn't let our static ip config from the initial setup in the `/etc/network/interfaces` file.

Remove it from the file so that the file looks like this:
```
source /etc/network/interfaces.d/*
```

## Last vm settings
Shutdown the machine and adjust some settings in proxmox:
- Remove CD-Drive that was used to install the inital os
- Add a cloud-init drive at ide2 port
- In options remove all boot entries except the os disk (faster boot)
- convert the vm to template
- test if it works!
