+++
title = "cloud-init"
date = "2022-11-15"
+++

You **want** to use cloud-init, for sure! Here are my configs I usually use for bootstraping new servers quickly.

## Ubuntu 22.04 - plain

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
    ssh-authorized_keys:
     - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJov21J2pGxwKIhTNPHjEkDy90U8VJBMiAodc2svmnFC cardno:18 055 612
write_files:
- path: /etc/ssh/sshd_config
  content: |
    Port 22
    PermitRootLogin no
    PermitEmptyPasswords no
    PasswordAuthentication no
    PubkeyAuthentication yes
    ChallengeResponseAuthentication no
    KbdInteractiveAuthentication no
    UsePAM yes
    X11Forwarding no
    PrintMotd no
    AcceptEnv LANG LC_*
    Subsystem    sftp    /usr/lib/openssh/sftp-server
    Include /etc/ssh/sshd_config.d/*.conf
runcmd:
  - systemctl restart sshd
```

## Ubuntu 22.04 - tailscale exit-node

```yaml
#cloud-config <hostname>

locale: en_US.UTF-8
timezone: UTC
users:
  - name: technat
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL # Allow any operations using sudo
    gecos: "Admin user created by cloud-init"
    shell: /usr/bin/bash
    ssh-authorized_keys:
     - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJov21J2pGxwKIhTNPHjEkDy90U8VJBMiAodc2svmnFC cardno:18 055 612
apt:
  sources:
    tailscale:
      source: "deb [signed-by=$KEY_FILE] https://pkgs.tailscale.com/stable/ubuntu jammy main"
      keyid: 458CA832957F5868
package_update: true
package_upgrade: true
packages:
- vim
- git
- wget
- curl
- dnsutils
- net-tools
- tailscale

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
- path: /etc/ssh/sshd_config
  content: |
    Port 22
    PermitRootLogin no
    PermitEmptyPasswords no
    PasswordAuthentication no
    PubkeyAuthentication yes
    ChallengeResponseAuthentication no
    KbdInteractiveAuthentication no
    UsePAM yes
    X11Forwarding no
    PrintMotd no
    AcceptEnv LANG LC_*
    Subsystem    sftp    /usr/lib/openssh/sftp-server
    Include /etc/ssh/sshd_config.d/*.conf
runcmd:
  - systemctl restart sshd
  - sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
  - sudo systemctl enable --now unattended-upgrades.service
  - sudo ufw default deny incoming
  - sudo ufw default allow outgoing
  - sudo ufw allow in on tailscale0
  - sudo ufw --force enable
  - sudo tailscale up --advertise-exit-node --ssh --auth-key "<auth key>"
  # Make sure the generated key is pre-approved and the exit-node will be autoapproved (e.g using a tag)
```
