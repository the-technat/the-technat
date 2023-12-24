+++
title = "Remote Coding"
author = "Nathanael Liechti"
date = "2023-12-24"
description = "How to efficiently develop from everywhere you are"
tags = [
  "linux",
]
+++

This guide show how I code / tinker.

## Remove Coding?

If you stumbeled over this post, you probably already know solutions like [Gitpod](https://www.gitpod.io/), [Github Codespaces](https://github.com/features/codespaces) or [Coder](https://coder.com/) to quickly spin up a development environment on the go.

While they are great and I've tried some of them as well, I like to have my own server for that. Mainly because it's more cost-effective and gives you more performance for less cost.

So this post documents how my server is setup for remote coding usage.

## The server

Well I'm not using a server, but an old desktop I had lying around. Phyiscal hardware is cumbersome to manage but for a static thing like that it should work.

The specs:
Desktop: HP EliteDesk 800 G1 SFF
CPU: Intel(R) Core(TM) i5-4570 CPU @ 3.20GHz
Memory: 32GB DDR3 (overkill, but I had it lying around)
Disk: 120GB SSD for OS, 500GB SSD for Data (not formated/mounted but one day I'm glad I have it)
NIC: just some dump 1Gbit/s NIC, directly attached to the modem of our provider

In case you're more after a cloud-based solution, I suggest you checkout [tevbox](https://github.com/the-technat/tevbox), where I did the same as here but using ephemeral cloud servers (and some automation of course).

## OS Setup

I installed Ubuntu Server 22.04.3 LTS on the main disk using LMV to partition the disk automatically.

LVM gives you more flexibility in case you need to change something on the partitioning later on.

After the system was installed I tweaked some things.

### Networking

Ubuntu comes with netplan by default, but I don't like it, especially if systemd already has a preinstalled solution. So I purged netplan and replaced it with [systemd-networkd]:

```console
sudo tee /etc/systemd/network/wired.network &>/dev/null <<EOF
[Match]
Name=enp2s0
[Network]
DHCP=yes
EOF

sudo systemctl enable --now systemd-network
sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo apt purge netplan.io -y
```

This works even with active VPN / SSH connections btw ;).

### Tailscale

I enable [tailscale](https://tailscale.com) on all my devices as allows us to connect remotly to your machine from wherever you are. This will be helpful later.

For now all you need to know is: `curl -fsSL https://tailscale.com/install.sh | sh` 

And then hit `tailscale up --ssh`.

Note: the `--ssh` enables ssh access to the machine from wherever I are, even from my browser if [Tailscale ACLs](https://tailscale.com/kb/1018/acls) are setup correctly.

### UFW

My server is somewhere I'm not sure what else is running in the same network, so I always enable a host-firewall. Tailscale has [a guide](https://tailscale.com/kb/1077/secure-server-ubuntu-18-04) how to enable `ufw` while you are connected to tailscale. I've no other rules applied, but this ensures no one can connect to the server except using Tailscale or if attaching a monitor.

### SSH

Since tailscale provides it's own SSH service, I disable/mask the builtin ssh server:

```
sudo systemctl disable --now ssh
sudo systemctl mask ssh
```

### Unattended-Upgrades

To be secure all the time and since the server will later be exposed to the internet I'll activate automatic security patches:


```console
sudo tee /etc/apt/apt.conf.d/50unattended-upgrades &>/dev/null <<EOF
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
Unattended-Upgrade::SyslogEnable "true";
EOF

sudo systemctl enable --now unattended-upgrades
```

## Coding Setup

Now that the server's OS is configured, we get to the main part: the coding. There's an awesome piece of software called [code-server](https://github.com/coder/code-server) that essentially gives you VS Code in your browser.

VS Code has builtin terminal support and depending on the browser you'll use you get almost the same experience as with regular VS code.

To install code-server run the following:

```console
curl -fsSL https://code-server.dev/install.sh | sh
sudo systemctl enable code-server@technat
sudo tailscale funnel --bg 8080
```

This last command is something I want to put special attention to. It enables Tailscale [Funnel](https://tailscale.com/blog/introducing-tailscale-funnel). Funnel is a feature that allows you to automatically expose a local running service to the internet. The tailscale agent on your machine that provides you with VPN also acts as a reverse-proxy in this scenario. If your tailnet is configured correctly, you get a memorable DNS name for your code-server and a TLS certificate out of the box.

In my experience code-server runs really well behind Funnel, and all things work like they should.

But, I changed the port where my code-server is running as `8080` is a common port other apps/commands try to bind to.

To change the port and also see the password for your code-server, edit `$HOME/.config/code-server/config.yaml`. This file should be self-explanitory. Just don't forget to restart the service after a code change.

### Coding tools

Let's skip this topic for now since I'll just run: `sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply the-technat` and I'm done with it. 