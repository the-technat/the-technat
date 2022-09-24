---
title: "proxmox"
---

This page shows one of many approaches how Proxmox can be installed and configured on a Dedicated Root Server from Hetzner.

## Server

This doc was created using an [AX41-NVME](https://www.hetzner.com/dedicated-rootserver/ax41-nvme) server with the following specs:

CPU: AMD Ryzen 5 3600 6-Core Processor
Memory: 64GB DDR4 Ram
Disks: 2x 512GB NVME
Data Disks: 2x 1TB SATA SSD
NIC: 1 GBit/s Port

### Hetzner Firewall

In the Hetzner Robot you have the option to add firewall rules for your server on Hetzner's Infrastructure. As proxmox has a built-in Firewall this is disabled.

### Hetzner Monitoring

In the Hetzner Robot you can also add simple Up-state monitoring Rules for your server. The following ones could be helpful:

| Check | Target | Protocol | Options | Contact |
| :----- | :------ | :-------- | :-------- | :--------- |
| ping | 135.181.162.151 | icmp | 5,180 | technat |
| https | 135.181.162.151 | https | 5,180 | technat |

## Proxmox VE (OS)

### Initial-Installation

To install proxmox the easiest way is to request a KVM console using the support form and add a commment that you would like a bootable usb stick with the latest Proxmox VE installer plugged into your server.
During the inital installation over the KVM console the following values have been applied:

| Option | Value |
| :------ | :------ |
| OS-Disk | ZFS (Raid1) with the NVME 512GB disks |
| ZFS ashift | 12 |
| ZFS compress | on |
| ZFS checksum | on |
| ZFS copies | 1 |
| ZFS hdsize | 476g (depends on the age of the disks...) |
| Location | Switzerland |
| Time Zone| Europe Zurich |
| Keyboard Layout | U.S English |
| Admin Mail | technat@technat.ch |
| IP | 135.181.162.151 |
| Gateway | 135.181.162.129 |
| DNS | 185.12.64.1 |

### Network Configuration

If proxmox is installed you can reach the Proxmox UI using the public ip addresse and the port `8006`. Login using `root`.

One of the first challenges we need to solve is how to do networking within proxmox for your VM's. There are different approaches using routing, bridging and more. This guide used a bridging apporach where we let the default `vmbr0` that was created during the installation as is.

In debian terms that would mean the following:

```
auto lo
iface lo inet loopback

iface enp35s0 inet manual

auto vmbr0
iface vmbr0 inet static
        address 135.181.162.151/26
        gateway 135.181.162.129
				pointopoint 135.181.162.129
        bridge_ports enp35s0
        bridge_stp off
        bridge_fd 0
```

That means that if we add a VM to `vmbr0` it would be in Hetzners network. This requires that you order an additional IP address and request a separate MAC address for your VM. In my case this will be a Firewall VM which will thus get a public IP and we can then use port-forwarding on that firewall VM to route to other VM's.

More details on Networking: https://community.hetzner.com/tutorials/install-and-configure-proxmox_ve

### DNS Configuration

Hetzner has [recursive DNS servers](https://docs.hetzner.com/dns-console/dns/general/recursive-name-servers/) that are rechable within their infrastructure:

```
nameserver 185.12.64.1
nameserver 185.12.64.2
nameserver 2a01:4ff:ff00::add:1
nameserver 2a01:4ff:ff00::add:2
```

### TLS Certificate

By default the Proxmox UI uses a self-signed certificate. To be a bit more secure I added a Job to obtain a Let's Encrypt certificate on host-level in the Proxmox UI. This only works as long as you haven't added any firewall rules.

Your only problem: If you later close Port 80, renewal won't work automatically.

Another solution would be to use Certbot directly and a cronjob to renew the cert. The benefit you have there is that you can dynamically open port 80 in your proxmox firewall to renew the cert.

If you have initally requested your certificate using `certonly` and `standalone` you can use the following example script in a cronjob to renew the cert:

```bash
#!/usr/bin/bash
<<Header
Script:   ssl-renew.sh
Date:     12.01.2021
Author:   technat
Version:  1.0
History   User    Date        Change
          technat 12.01.2021  initial version
Description: update ssl cert from Let's Encrypt using certbot and copy certs for use with pveproxy
Cronjob: 0 5 1 * * /root/ssl-renew.sh
Dependency: certbot, pveproxy

Header

#############################################################################
################################# Variables #################################
#############################################################################

# general
domain='yourdomain.com'
logfile='/var/log/ssl-renew.log'
rule='IN ACCEPT -i vmbr0 -dest 1.2.3.4 -p tcp -dport 80 -log nolog'

#############################################################################
############################### Preparations ################################
#############################################################################

# certbot is installed
which certbot
if [[ ! $? -eq 0 ]]; then
    sudo apt install certbot -y
fi

# open port 80 on proxmox firewall
sed -i "/\[RULES\]/a $rule" /etc/pve/firewall/cluster.fw
sudo pve-firewall stop
sudo pve-firewall start

#############################################################################
################################ Main Script ################################
#############################################################################

# renew cert
/usr/bin/certbot renew >> $logfile

# delete the old cert
rm -rf /etc/pve/local/pve-ssl.pem
rm -rf /etc/pve/local/pve-ssl.key
rm -rf /etc/pve/pve-root-ca.pem

# copy the renewed one to the correct location
cp /etc/letsencrypt/live/$domain/fullchain.pem /etc/pve/local/pve-ssl.pem
cp /etc/letsencrypt/live/$domain/privkey.pem /etc/pve/local/pve-ssl.key
cp /etc/letsencrypt/live/$domain/chain.pem /etc/pve/pve-root-ca.pem

#############################################################################
################################## Cleanup ##################################
#############################################################################

# close port 80 on proxmox firewall
sed -i "s/$rule//g" /etc/pve/firewall/cluster.fw
sudo pve-firewall stop
sudo pve-firewall start

# postprocessing - restart pveproxy
systemctl restart pveproxy

```

### SSH Configuration

For security the following settings were applied in OpenSSH config:

| Option | Value |
| :------ | :------ |
| Port | 59245 |
| PubkeyAuthentication | yes |
| PasswordAuthentication | no |
| PermitEmptyPasswords | no |
| PermitRootLogin | no |

This requires that you have an admin user other than `root` which has an ssh-key configured.

### User Management

I recommend you disable Root Login to the Proxmox UI by disabling the Root user.

For admin users it's recommended to add some sort of 2FA (e.q using TOTP or Yubikeys)

More documentation from Proxmox: https://pve.proxmox.com/wiki/User_Management

### Hetzner StorageBox

If you but a dedicated root server you get an additional 100GB Storagebox for free. I use this to store my ISOs and Container images. You can add the storage to your server at datacenter level in the UI.

Use the following options

| Option | Value |
| :------ | :------ |
| ID | remote-cifs |
| Server | u253012.your-storagebox.de |
| User | u253012 |
| PW | concealed |
| Content | All except containers and disk images |

### local-data

While ordering the server I added 2 additional 1TB SATA drives for data disks of VMs. In the ZFS tab on the host I created a data store out of them:

| Option | Value |
| :-------| :------|
| Name | local-data |
| ZFS ashift | 12 |
| Type | Mirror
| ZFS compress | on |

### APT Repositories Configuration

In most cases we don't have a license for the Proxmox Enterprise Repositories. To avoid annoying Error messages when doing `sudo apt update` we can disable the enterprise repo. Delete the single line within `/etc/apt/sources.list.d/pve-enterprise.list`.

If we do that we need to add the non-enterprise proxmox repositories as well as the [Hetzner APT mirrors](https://docs.hetzner.com/robot/dedicated-server/operating-systems/hetzner-aptitude-mirror/).

My `/etc/apt/sources.list` looks like that at the end:

```
# Packages and Security Updates from the Hetzner Debian Mirror
deb https://mirror.hetzner.com/debian/packages  bullseye           main contrib non-free
deb https://mirror.hetzner.com/debian/packages  bullseye-updates   main contrib non-free
deb https://mirror.hetzner.com/debian/security  bullseye-security  main contrib non-free
deb https://mirror.hetzner.com/debian/packages  bullseye-backports main contrib non-free

# Offizial Debian Bullseye Repositories (fallback)
deb http://deb.debian.org/debian               bullseye          main non-free contrib
deb http://deb.debian.org/debian               bullseye-updates  main non-free contrib
deb http://security.debian.org/debian-security bullseye-security main contrib non-free

# proxmox updates
deb http://download.proxmox.com/debian buster pve-no-subscription
```

### Proxmox Firewall

Now we can enable the Proxmox firewall. Default in-Policy is Drop and then I added the following rules at datacenter level:

| Direction | Action | Interface | Source-IP | Dest-IP | Macro | Protocol | Source-Port | Dest-Port | LogLevel |
| :------ | :--------- | :-------------- |  :------------ | :----------------- | :--------- | :---------- | :------ | :----- | :----- |
| in | ACCEPT | vmbr0 | any | 135.191.162.151 | - | icmp | any | any | nolog |
| in | ACCEPT | vmbr0 | any | 135.181.162.151 | - | tcp | any | 8006 | critical |
| in | ACCEPT | vmbr0 | any | 135.181.162.151 | - | tcp | any | 59245 | critical |

No rules at host level.

### Fail2ban

I recommend you configure fail2ban for your server. The following article may help: https://www.ukhost4u.com/securing-proxmox-and-ssh-using-fail2ban/

## Further Reading

Note: After a while your memory may be filled up without using it. This is because of disk caching. See https://www.linuxatemyram.com/

- [Network Configuration Hetzner](https://docs.hetzner.com/robot/dedicated-server/ip/additional-ip-adresses/)
- [Other Community Guide installing proxmox on hetzner dedicated](https://community.hetzner.com/tutorials/install-and-configure-proxmox_ve)
