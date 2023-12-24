+++
title = "Remote Coding"
author = "Nathanael Liechti"
date = "2023-12-14"
description = "How to efficiently develop from everywhere you are"
tags = [
  "linux",
]
draft = true
+++

This guide show how I code / tinker.

## Remove Coding?

Remote coding solutions like [Gitpod](https://www.gitpod.io/), [Github Codespaces](https://github.com/features/codespaces) or [Coder](https://coder.com/) have been
increasingly popular in the last few years. I can understand that since they offer great 

## My setup

[tevbox](https://github.com/the-technat/tevbox)

banana: 

## OS Setup
LVM for greater flexibility (make sure names are not bound to hostname)

## OS Config

- install tailscale (managed by tags)
- disable netplan.io
- use stub-resolver of systemd-resovled
- use systemd-networkd using:
  ```wired.network
  [Match]
  Name=enp2s0
  
  [Network]
  DHCP=yes
  ```
- ufw is enabled according to [tailscale's guide](https://tailscale.com/kb/1077/secure-server-ubuntu-18-04)
- code-server
- chezmoi bootstraps the rest of my shell
- unattended-upgrades enabled for security patches:
  ```
  Unattended-Upgrade::MinimalSteps "true";
  Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
  Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
  Unattended-Upgrade::Remove-Unused-Dependencies "true";
  Unattended-Upgrade::Automatic-Reboot "true";
  Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
  Unattended-Upgrade::Automatic-Reboot-Time "02:00";
  Unattended-Upgrade::SyslogEnable "true";
  ```

Try to use ephemeral credentails and not store any permanent data on the machine. Sometimes it makes sense to keep stuff around for some days/weeks, but if it's a project you are working on, put it in a repository.
