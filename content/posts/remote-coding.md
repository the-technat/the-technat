+++
title = "Remote Coding"
author = "Nathanael Liechti"
date = "2024-02-20"
description = "How to code from everywhere you are"
tags = [
  "linux",
]
+++

This guide shows how I code / tinker on the go.

## Background

If you stumbeled over this post, you probably already heard of know solutions like [Gitpod](https://www.gitpod.io/), [Github Codespaces](https://github.com/features/codespaces) or [Coder](https://coder.com/) to quickly spin up a development environment on the go. While I was using them I got fascinated about their capabilites and wished that I could use them for everything I do. Unfortunately some of these solutions are tied to a specific platform and or repository or they don´t offer you the performance or control that you would like. That was when I stumbeled over [code-server](https://code-server.dev), a web-based build of [Code OSS](https://github.com/code-oss-dev/code). That was the missing piece I needed to finally build my own codespace-like solution, but that time with full control.

So this post documents how I setup a server for remote coding usage. 

## The server

Well I'm not using a server, but an old desktop I had lying around. Phyiscal hardware is cumbersome to manage but for a static thing like a code-server it works fairly well I think.

The specs:

| Part | Description |
| ---- | ---- |
| Desktop | HP EliteDesk 800 G1 SFF | 
| CPU | Intel(R) Core(TM) i5-4570 CPU @ 3.20GHz |
| Memory | 32GB DDR3 (overkill, but I had it lying around) |
| Disk | 120GB SSD for OS, 500GB SSD for Data (not formated/mounted but one day I'm glad I have it) |
| NIC | just some dump 1Gbit/s NIC |

## The Operating System

I installed Ubuntu Server 24.04.1 LTS on the main disk using LVM to partition the disk automatically. Ubuntu Server is simple, widely adopted, fairly minimal and thus perfect for that kind of server. The manual install has to be done only once and LVM gives you the flexibility to tweak the disk-partitioning later on in case you want that.

### Networking

Ubuntu comes with netplan by default, but I don't like it, especially if systemd already has a preinstalled solution. So I purged netplan and replaced it with systemd-networkd:

```console
sudo tee /etc/systemd/network/wired.network &>/dev/null <<EOF
[Match]
Name=enp2s0
[Network]
DHCP=yes 
EOF

sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo apt purge netplan.io -y
```

This works even with active VPN / SSH connections btw ;).

### Tailscale

I always install [Tailscale](https://tailscale.com) on my systems so that as long as they have egress connectivity I can connect to them somehow. Tailscale also has a feature called [Funnel](https://tailscale.com/kb/1223/funnel) that allows you to expose service in the internet without a public IP or open ports. We will use this later on.

To install Tailscale, there's a oneliner: `curl -fsSL https://tailscale.com/install.sh | sh`.

To bring it up, there's a oneliner too: `sudo tailscale up --ssh`

### UFW

It's always a good practice to use a host-firewall. I configured `ufw` for that: 

```console
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
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
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "02:00";
Unattended-Upgrade::SyslogEnable "true";
EOF

sudo systemctl enable --now unattended-upgrades
```

## Coding Setup

Now that the server's OS is configured, we get to the main part: the coding. 

To install code-server run the following:

```console
curl -fsSL https://code-server.dev/install.sh | sh
sudo systemctl enable --now code-server@technat
```

This installs the code-server as systemd-service and enables it for your current user. The default for code-server is to listen on `127.0.0.1:8080`, but I'll change that to something less common (to avoid conflicts with other local apps):

```
sed -ei 's/^bind-addr.*/bind-addr: 127.0.0.1:65000/g' ~/.config/code-server/config.yaml
```

Don´t forget to restart the service with `sudo systemctl restart code-server@technat` after that change.

### Exposing code-server

As you have noticed so far, code-server listens on localhost, but we want to use it from everywhere. That's a sane default code-server uses here because code-server itself isn't meant to be exposed directly. Instead you should use a reverse-proxy as [the docs](https://coder.com/docs/code-server/latest/guide#expose-code-server) suggest.

As mentioned earlier on, that's where Funnel comes into play. My machine is already connected to Tailscale, so I run:

```bash
sudo tailscale funnel --bg 65000
```

Funnel must be allowed for your machine, but when you run the command tailscale will tell you that. It's also a good practice to read the docs about [Funnel](https://tailscale.com/kb/1223/funnel) to learn about the prerequisites. 

Please note that exposing local development services via Tailscale is only possible [path-based](https://coder.com/docs/code-server/latest/guide#using-a-subpath). I haven't yet found a better solution to that.

### Authentication

Now that code-server is exposed in the internet, we should add some authentication. By default code-server has a builtin password authentication with a randomly generated password. But I don't like this and replaced it with an OAuth Flow to sign-in with Github. 

First thing to do for this is to disable the current authentication:

```
sed -ei 's/^auth:.*/auth: none/g' ~/.config/code-server/config.yaml
sudo systemctl restart code-server@technat
```

Then I install [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy) to my system:

```
ARCH=amd64
OS=linux
VERSION=v7.8.1
curl -fsSL -o /tmp/oauth2-proxy.tar.gz https://github.com/oauth2-proxy/oauth2-proxy/releases/download/$VERSION/oauth2-proxy-$VERSION."$OS"-"$ARCH".tar.gz
tar -C /tmp -xzf /tmp/oauth2-proxy.tar.gz
sudo install /tmp/oauth2-proxy-$VERSION."$OS"-"$ARCH"/oauth2-proxy /usr/local/bin/oauth2-proxy

cat <<EOF | sudo tee /etc/systemd/system/oauth2-proxy.service
[Unit]
Description=oauth2-proxy daemon service
After=network.target network-online.target nss-lookup.target basic.target
Wants=network-online.target nss-lookup.target
StartLimitIntervalSec=30
StartLimitBurst=3

[Service]
User=technat
Group=technat
Restart=on-failure
RestartSec=30
ExecStart=/usr/local/bin/oauth2-proxy --config=/etc/oauth2-proxy.cfg
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65535
NoNewPrivileges=true
ProtectHome=true
ProtectSystem=full
ProtectHostname=true
ProtectControlGroups=true
ProtectKernelModules=true
ProtectKernelTunables=true
LockPersonality=true
RestrictRealtime=yes
RestrictNamespaces=yes
MemoryDenyWriteExecute=yes
PrivateDevices=yes
PrivateTmp=true
CapabilityBoundingSet=

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now oauth2-proxy
```

Once it's installed we can create an OAuth app in Github according to [this doc](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/github).  I'll use `https://my-machine.blabla.ts.net/oauth2/callback` as callback URL.

And then we create the config file for oauth2-proxy:

```/etc/oauth2-proxy.cfg
footer         = "-" 
cookie_domains = ["my-machine.blabla.ts.net"]
whitelist_domains = ["my-machine.blabla.ts.net"]
cookie_secure = true
cookie_expire = "2h"
http_address = "127.0.0.1:65001"
reverse_proxy = true 
provider = "github"
client_id = "REPLACE_ME"
client_secret = "REPLACE_ME"
cookie_secret = "$(openssl rand -base64 32 | tr -- '+/' '-_')"
email_domains = ["*"] 
upstreams = ["http://127.0.0.1:65000/" ]
github_users = ["the-technat"]
```

Some notes:
- `footer` disables the version to be shown on the sign-in page
- `cookie_secret` run the command to generate your unique cookie secret and paste the exact output
- `github_users` the config allows everyone in that list to sign in, 

Restart the service after you created the config file. Finally we can recreate our funnel to point to auth2-proxy instead of code-server:

```bash
sudo tailscale funnel reset
sudo tailscale funnel --bg 65001
```

## Coding tools

Now that you have your code-server you might want to activate extensions and install progamming languages. I'll skip this topic as it highly depends on what you are using your code-serer for. I will just run: `sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply the-technat` and I'm done with it. 
