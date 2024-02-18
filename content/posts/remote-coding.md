+++
title = "Remote Coding"
author = "Nathanael Liechti"
date = "2023-12-24"
description = "How to efficiently develop from everywhere you are"
tags = [
  "linux",
]
+++

This guide shows how I code / tinker.

## Remove Coding?

If you stumbeled over this post, you probably already know solutions like [Gitpod](https://www.gitpod.io/), [Github Codespaces](https://github.com/features/codespaces) or [Coder](https://coder.com/) to quickly spin up a development environment on the go.

While they are great and I've tried some of them as well, I like to have my own server for that. Mainly because it's more controlable and gives you more performance for less cost.

So this post documents how I setup a server for remote coding usage.

## The server

Well I'm not using a server, but an old desktop I had lying around. Phyiscal hardware is cumbersome to manage but for a static thing like that it should work.

The specs:

| Kind | Detail |
| ---- | ---- |
| Desktop | HP EliteDesk 800 G1 SFF | 
| CPU | Intel(R) Core(TM) i5-4570 CPU @ 3.20GHz |
| Memory | 32GB DDR3 (overkill, but I had it lying around) |
| Disk | 120GB SSD for OS, 500GB SSD for Data (not formated/mounted but one day I'm glad I have it) |
| NIC | just some dump 1Gbit/s NIC, directly attached to the modem of our provider |

In case you're more after a cloud-based solution, I suggest you checkout [tevbox](https://github.com/the-technat/tevbox), where I did the same as here but using ephemeral cloud servers (and some automation of course).

## OS Setup

I installed Ubuntu Server 22.04.3 LTS on the main disk using LMV to partition the disk automatically.

LVM gives you more flexibility in case you need to change something on the partitioning later on.

### Networking

Ubuntu comes with netplan by default, but I don't like it, especially if systemd already has a preinstalled solution. So I purged netplan and replaced it with systemd-networkd:

```console
sudo tee /etc/systemd/network/wired.network &>/dev/null <<EOF
[Match]
Name=enp2s0
[Network]
DHCP=ipv6 
EOF

sudo tee /etc/systemd/resolved.conf.d/dns.conf &>/dev/null <<EOF
[Resolve]
DNS=2620:fe::fe,2620:fe::9
EOF

sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo apt purge netplan.io -y
```

Note: the first command reconfigured systemd-networkd to only use IPv6 for DHCP. That's my intended behaviour!

This works even with active VPN / SSH connections btw ;).

### UFW

My Server is directly attached to my IPSs Router and thus got an IPv6 address we can later use to expose stuff. But since this IP is public and the Router itself doesn´t have a firewall, it's a good practice to enable a host-firewall. 

For this purpose I use `ufw` like so:

```console
sudo ufw enable
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2a00:d4e0:100:4500:da47:32ff:fee8:bf51:443
sudo ufw allow 2a00:d4e0:100:4500:da47:32ff:fee8:bf51:80
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

Now that the server's OS is configured, we get to the main part: the coding. There's an awesome piece of software called [code-server](https://github.com/coder/code-server) that essentially gives you VS Code in your browser.

VS Code has builtin terminal support and depending on the browser you'll use you get almost the same experience as with regular VS code.

To install code-server run the following:

```console
curl -fsSL https://code-server.dev/install.sh | sh
sudo systemctl enable --now code-server@technat
```

Once this is done, you can tweak the config of code-server in `.config/code-server/config.yaml`. I usually change the port to something none-conflicting + change the randomly generated password in that file. Whatever you change, don´t for get to restart the service with `sudo systemctl restart code-server@technat`.

### Configuring code on code-server

The VS Code part of code-server has it's config in `.local/share/code-server/User/settings.json` which looks for me like that:

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

Code-server by default listens on `http:127.0.0.1:8080` (or `65000`) for me. To expose this now to the internet using SSL I'm using [Caddy](https://caddyserver.com/).

First install it:
```console
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Then let's define a DNS record we want to use and create corresponding DNS records. Here I'll now use the IPv6 adress of my box which I can assume it's probably static. I also set a wildcard DNS entry for my code-server box. This will help me to expose services on my box via code-server to the world. E.g if I run a development version of my code via code-server it will automatically forward the port via code-server and if caddy does the same, my local dev instance is publicly available.

See [here](https://coder.com/docs/code-server/latest/guide#accessing-web-services) for more information about that feature. If you don't want it, omit the DNS record.

We can then define the caddy config in `/etc/caddy/Caddyfile`:

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
http://*.technat.dev {
  bind 2a00:d4e0:100:4500:da47:32ff:fee8:bf51
  reverse_proxy 127.0.0.1:65000
}
```

And restart the service: `sudo systemctl restart caddy`. This will automatically request a TLS certificate for the domain you defined and configure it. Check the logs of caddy with `sudo journalctl -xu caddy` and then browser to your code-server in browser.

Please note: the first block is for code-server, the others for accessing web-services using code-server. I could also have defined a wildcard there, but since wildcard certificates require you to use DNS-01 I deciced to stick with the most common ports I would use.

If you now still want to use exposing of web-services you should define the domain code-server is exposed on in `.config/code-server/config.yaml`:

```
proxy-domain: technat.dev
```

And restart the service again. But be careful: A typo and the service won't come up again.

### Coding tools

Let's skip this topic for now since I'll just run: `sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply the-technat` and I'm done with it. 
