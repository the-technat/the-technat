+++
title = "cloud-init"
date = "2022-11-15"
+++

You **want** to use cloud-init, for sure! Here are my configs I usually use for bootstraping new servers quickly.

## Ubuntu 24.04

Delete the content you don't need. Otherwise it joins the server to my tailnet, installs some packages, does a system upgrade and enables automatic updates.

```yaml
#cloud-config <hostname>

locale: en_US.UTF-8
timezone: UTC
package_update: true
package_upgrade: true
packages:
- vim
- git
- wget
- curl
- dnsutils
- net-tools

users:
  - name: technat
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL # Allow any operations using sudo
    gecos: "Admin user created by cloud-init"
    shell: /usr/bin/bash

write_files:
- path: /etc/sysctl.d/99-tailscale.conf
  content: |
    net.ipv4.ip_forward          = 1
    net.ipv6.conf.all.forwarding = 1
- path: /etc/apt/apt.conf.d/50unattended-upgrades
  content: |
    Unattended-Upgrade::Allowed-Origins {
      "${distro_id}:${distro_codename}";
      "${distro_id}:${distro_codename}-security";
      "${distro_id}ESMApps:${distro_codename}-apps-security";
      "${distro_id}ESM:${distro_codename}-infra-security";
      "${distro_id}:${distro_codename}-updates";
      "${distro_id}:${distro_codename}-proposed";
      "${distro_id}:${distro_codename}-backports";
    };

    Unattended-Upgrade::DevRelease "auto";
    Unattended-Upgrade::Remove-Unused-Dependencies "true";
    Unattended-Upgrade::Automatic-Reboot "true";
    Unattended-Upgrade::Automatic-Reboot-Time "23:00";
    Unattended-Upgrade::SyslogEnable "true";

runcmd:
  - sysctl -p /etc/sysctl.d/99-tailscale.conf
  - systemctl enable --now unattended-upgrades.service
  - systemctl mask ssh
  - curl -fsSL https://tailscale.com/install.sh | sh
  - tailscale up --ssh --auth-key "<single-use-pre-approved-key>"
  - ufw default deny incoming
  - ufw default allow outgoing
  - ufw allow in on tailscale0
  - ufw --force enable
```
