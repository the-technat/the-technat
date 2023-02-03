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
    Port 59245
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
