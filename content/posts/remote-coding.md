+++
title = "Remote Coding"
author = "Nathanael Liechti"
date = "2024-02-20"
description = "How to code from everywhere you are"
tags = [
  "linux",
]
+++

This guide shows how I code / tinker.

## Background

If you stumbeled over this post, you probably already of know solutions like [Gitpod](https://www.gitpod.io/), [Github Codespaces](https://github.com/features/codespaces) or [Coder](https://coder.com/) to quickly spin up a development environment on the go. While I was using them I got fascinated about their capabilites and wished that I could use them for everything I do. Unfortunately some of these solutions are tied to a specific platform and or repository or they don´t offer you the performance or control that you would like. That was when I stumbeled over [code-server](https://code-server.dev), a web-based build of [Code OSS](https://github.com/code-oss-dev/code). That was the missing piece I needed to finally build my own codespace-like solution, but that time with full control.

So this post documents how I setup a server for remote coding usage. In case you're more after a cloud-based solution with ephemeral reproducable machines (that are otherwise setup the same as here), I suggest you checkout [tevbox](https://github.com/the-technat/tevbox), where I did exactly this with a lot of automation. Otherwise read on.

## The server

Well I'm not using a server, but an old desktop I had lying around. Phyiscal hardware is cumbersome to manage but for a static thing like that a code-server it works fairly well I found. 

The specs:

| Part | Description |
| ---- | ---- |
| Desktop | HP EliteDesk 800 G1 SFF | 
| CPU | Intel(R) Core(TM) i5-4570 CPU @ 3.20GHz |
| Memory | 32GB DDR3 (overkill, but I had it lying around) |
| Disk | 120GB SSD for OS, 500GB SSD for Data (not formated/mounted but one day I'm glad I have it) |
| NIC | just some dump 1Gbit/s NIC, directly attached to the modem of our provider |

## The Operating System

I installed Ubuntu Server 22.04.3 LTS on the main disk using LMV to partition the disk automatically. Ubuntu Server is simple, widely adopted, fairly minimal and thus perfect for that kind of server. The manual install has to be done only once and LVM gives you the flexibility to tweak the disk-partitioning later on in case you want that.

### Networking

Ubuntu comes with netplan by default, but I don't like it, especially if systemd already has a preinstalled solution. So I purged netplan and replaced it with systemd-networkd:

```console
sudo tee /etc/systemd/network/wired.network &>/dev/null <<EOF
[Match]
Name=enp2s0
[Network]
DHCP=yes 
EOF

sudo tee /etc/systemd/resolved.conf.d/dns.conf &>/dev/null <<EOF
[Resolve]
DNS=9.9.9.9,149.112.112.112,2620:fe::fe,2620:fe::9
EOF

sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo apt purge netplan.io -y
```

This works even with active VPN / SSH connections btw ;).

### Tailscale

I always install [Tailscale](https://tailscale.com) on my systems so that as long as they have egress connectivity I can still connect to them somehow (e.g in case I mixed something up).

### UFW

It's always a good practice to use a host-firewall, not only for the purpose I explain later. I configured `ufw` for that: 

```console
sudo sed -ei 's/^IPV6.*/IPV6=yes/g' /etc/default/ufw # should already be done
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow 443
sudo ufw allow 80
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

This installs the code-server as systemd-service and enables it for your current user. The default for code-server is to listen on `127.0.0.1:8080`, but I'll change that to something less common (to avoid conflicts):

```
sed -ei 's/^bind-addr.*/bind-addr: 127.0.0.1:65000/g' ~/.config/code-server/config.yaml
```

Don´t forget to restart the service with `sudo systemctl restart code-server@technat` after that change.

### Configuring code on code-server

The Code OSS part of code-server has it's config in `.local/share/code-server/User/settings.json`. That's what you will automatically configured if you change settings on the web in code-server. I usually put some defaults in there:

```json
{
    "workbench.colorTheme": "Solarized Light",
    "redhat.telemetry.enabled": false,
    "workbench.sideBar.location": "right",
    "workbench.startupEditor": "none",
    "terminal.integrated.defaultProfile.linux": "zsh",
    "explorer.confirmDragAndDrop": false
}
```

### Exposing code-server

As you have noticed so far, code-server listens on localhost, but we want to use it from everywhere. That's a sane default code-server uses here because code-server itself isn't meant to be exposed directly. Instead you should use a reverse-proxy as [the docs](https://coder.com/docs/code-server/latest/guide#expose-code-server) suggest.

I got two options explained here: tailscale or caddy

### Tailscale

Just hit:

```
sudo tailscale funnel --bg 65000
```

And read the docs about [Funnel](https://tailscale.com/kb/1223/funnel) to learn about the prerequisites.

Please note that exposing local development services via Tailscale is only possible path-based this way.

#### Caddy

First install it:
```console
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```
Then define the caddy config in `/etc/caddy/Caddyfile` (update the domain to something you own):

```
technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
8080.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
8081.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
9090.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
9091.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
3000.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
12000.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
```

And restart the service: `sudo systemctl restart caddy`. 

##### What will happen?

Caddy will now automatically request a TLS certificate for the domain you defined and configure it. To do so caddy listens on the specified IPv6 address which is configured on my host and allowed on the firewall (ufw as well as my router in front). If you have a public-ip and do port-forwarding, just remove the bind-address to listen on all interfaces and IPs.

Of course for this to work the main and all the subdomains must resolve to the public IP.

##### Why subdomains?
Because of [accessing web-services exposed via code-server](https://coder.com/docs/code-server/latest/guide#accessing-web-services). It's an easy way to reach a local development services from the internet. With the tailscale variant above that will work too, but on sub-paths which is sometimes not desired.

You could also do this with one wildcard DNS entry, but then you have to configure a DNS-01 challenge for caddy which requires more effort and credentials.

If you have configured the subdomains, also add the domain in `.config/code-server/config.yaml` (this helps code-server automatically show you the full URL with one click):

```
sed -ei 's/^proxy-domain:.*/proxy-domain: technat.dev/g' ~/.config/code-server/config.yaml
```

And restart the service again. But be careful: A typo and the service won't come up again.

### Coding tools

Let's skip this topic for now since I'll just run: `sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply the-technat` and I'm done with it. 
