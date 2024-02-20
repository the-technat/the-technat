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

Please note that exposing local development services via Tailscale is only possible [path-based](https://coder.com/docs/code-server/latest/guide#using-a-subpath) this way.

#### Caddy

First install it:
```console
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Then define the caddy config in `/etc/caddy/Caddyfile` (update the domain to something you own and create the related DNS records):

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

Caddy will now automatically request a TLS certificate for the domain you defined and configure it. This will only work if you set DNS records, if not do so now. In my config Caddy listens on the specified IPv6 address which is configured on my host and allowed on the firewall of my router (and in ufw of course). This is because I'm lazy and didn't wanted to setup DynDNS nor port-forwarding. If you want your code-server to be IPv4 capable you probably want to port-forward from your router to the server and setup a DynDNS record for the public IP of your router. Or you might also have a public static IP assigned to your server already to use. In any way remove the `bind` directive for Caddy to listen on all interfaces. 

Please note: My box still has a private IPv4 address to reach IPv4 only endpoints (like Github which currently doesn't support IPv6). 

##### Why subdomains?

Because of [accessing web-services exposed via code-server](https://coder.com/docs/code-server/latest/guide#accessing-web-services). It's an easy way to reach a local development services from the internet. With the tailscale variant above that will work too, but on sub-paths which is sometimes not desired.

You could also do this with one wildcard DNS entry instead of defining all the domains seperately, but then you have to configure a DNS-01 challenge for Caddy which requires more effort and credentials (because wildcards can only be obtained using DNS-01 from Let's Encrypt).

One last thing to do in this matter is to also add the domain in `.config/code-server/config.yaml`. This is required for code-server to forward the request to your local process properly:

```
sed -ei 's/^proxy-domain:.*/proxy-domain: technat.dev/g' ~/.config/code-server/config.yaml
```

And restart the service again. But be careful: A typo and the service won't come up again.

### Authentication

Let's quickly mention that: code-server generates a password that you can find in it's config file. If you change it restart the service. But I don't like this and replaced it with an OAuth Flow to sign-in with Github. 

First thing to do for this is to disable the current authentication:

```
sed -ei 's/^auth:.*/auth: none/g' ~/.config/code-server/config.yaml
```

Then I install [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy) to my system:

```
ARCH=amd64
OS=linux
VERSION=v7.6.0
curl -fsSL -o /tmp/oauth2-proxy.tar.gz https://github.com/oauth2-proxy/oauth2-proxy/releases/download/$VERSION/oauth2-proxy-$VERSION."$OS"-"$ARCH".tar.gz
tar -C /tmp xzf /tmp/oauth2-proxy.tar.gz
sudo install /tmp/oauth2-proxy-$VERSION."$OS"-"$ARCH"/oauth2-proxy /usr/local/bin/oauth2-proxy

cat <<EOF | sudo tee /etc/systemd/system/oauth2-proxy.service
[Unit]
Description=oauth2-proxy daemon service
After=syslog.target network.target

[Service]
User=caddy
Group=caddy

ExecStart=/usr/local/bin/oauth2-proxy --config=/etc/oauth2-proxy.cfg --github-user=the-technat
ExecReload=/bin/kill -HUP $MAINPID

KillMode=process
Restart=always

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now oauth2-proxy
```

Be sure to replace your username in `--github-user`. It's the only directive that's currently somehow not supported in the config file.

Once it's running we can create an OAuth app in Github according to [this doc](https://oauth2-proxy.github.io/oauth2-proxy/configuration/providers/github).  I'll use `https://technat.dev/oauth2/callback` as callback URL.

And then create the config file for oauth2-proxy:

```/etc/oauth2-proxy.cfg
cookie_domains = ".technat.dev"
cookie_secure = true
cookie_expire = "2h"
http_address = "127.0.0.1:65001"
reverse_proxy = true # Are we running behind a reverse proxy? Will not accept headers like X-Real-Ip unless this is set.
provider = "github"
client_id = "REPLACE_ME"
client_secret = "REPLACE_ME"
cookie_secret = "$(openssl rand -base64 32 | tr -- '+/' '-_')" # generate new cookie secret with this command
email_domains = ["*"] 
upstreams = ["http://127.0.0.1:65000/" ]
```

Note: this config allows everyone with a Github account to sign in, somehow the `--github-user` flag can't be translated into a config directive, but as shown above this flag is set on the systemd service.

Don't forget to restart the service afterwards. Once this is done, simply replace all `reverse_proxy` directives in caddy with `127.0.0.1:65001` (or omit it if you don't want authentication for an endpoint) and you're done!

## Coding tools

Now that you have your code-server you might want to active extensions and install progamming languages. I'll skip this topic as it highly depends on what you are using your code-serer for. I will just run: `sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply the-technat` and I'm done with it. 
