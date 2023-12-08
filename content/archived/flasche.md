
Minecraft Server for friends

## Hardware

Simple "furz" on Hcloud:
- Helsinki
- Ubuntu 22.04
- CPX11
- Backups
- Default cloud-init
- DNS record for both IPv4 and IPv6 -> flasche.technat.ch

## Prerequisites

### Updates

`/etc/apt/apt.conf.d/50unattended-upgrades`:
```console
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
```

```
sudo systemctl enable --now unattended-upgrades.service
```

### Tailscale

Remote management:

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
```

### Firewall

```
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw --force enable
```

## Installation

### Java

```
sudo apt install openjdk-17-jdk-headless -y
```


### mcrcon

Small unmaintained utlity to send remote commands from the cli to the server:

```
sudo apt install make gcc -y
cd /tmp
git clone https://github.com/Tiiffi/mcrcon.git
cd mcrcon
make
sudo make install
```

### MC User

```
sudo groupadd -r minecraft
sudo useradd -r -g minecraft -d "/var/minecraft" -s "/bin/bash" minecraft
sudo mkdir -p /var/minecraft/{backup,server}
sudo chown minecraft.minecraft -R /var/minecraft/
sudo chmod -R 770 /var/minecraft
sudo su minecraft
```

### Server Binary

Get the latest binary from https://www.minecraft.net/en-us/download/server to `/var/minecraft/server/server.jar`

Two options:
- either own it 770 by minecraft -> allows for auto-upgrade by cronjob
- or 755 by root -> prevents replacing by itself

### Initial configuration

Run the server once to generate all files:

```
sudo su minecraft
cd /var/minecraft/server
java -Xmx1024M -Xms1024M -jar server.jar --nogui
```

And then change the following params in `server.properties:
```
enable-rcon=true
rcon-password=gugus
```

And accept the EULA in `eula.txt`:
```
eula=true
```

### Systemd service

`/etc/systemd/system/minecraft.server`:
```
[Unit]
Description=Minecraft Server
After=network.target

[Service]
Type=simple
User=minecraft
Group=minecraft
WorkingDirectory=/var/minecraft/server
ExecStart=/usr/local/bin/mc-start
ExecStop=/usr/local/bin/mc-stop
Restart=always

ProtectHome=true
ProtectSystem=full
PrivateDevices=true


NoNewPrivileges=true
PrivateTmp=true
InaccessibleDirectories=/root /sys /srv -/opt /media -/lost+found
ReadWriteDirectories=/var/minecraft/server

[Install]
WantedBy=default.target
```

`/usr/local/bin/mc-start`:

```
#!/bin/sh

cd /var/minecraft/server
java -Xmx1G -Xms1G \
  -jar server.jar nogui
```

`/usr/local/bin/mc-stop`:

```
#!/bin/sh

/usr/local/bin/rcon stop

while kill -0 $MAINPID 2>/dev/null
do
  sleep 0.5
done
```

`/usr/local/bin/rcon`:

```
#!/bin/sh

mcrcon -H localhost -P 25575 -p "gugus" $@
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now minecraft
```


### Open Port

Finally open the port in the firewall:

```
sudo ufw allow 25565
```

## Configuration Tweaking

Now we are ready to accept configuration tweaking from our friends.

```
motd=Flasche MC Server
max-players=5
white-list=true
enforce-whitelist=true
```

`ops.json`:

```
[
  {
    "uuid": "7f67870f-57d6-406c-9ad1-e848715a9453",
    "name": "technat",
    "level": 4,
    "bypassesPlayerLimit": false
  }
]
```

`whitelist.json`:

```
[
  {
    "uuid": "7f67870f-57d6-406c-9ad1-e848715a9453",
    "name": "technat"
  }
]
```

## Resources
- https://minecraft.fandom.com/wiki/Tutorials/Setting_up_a_server
- https://teilgedanken.de/Blog/post/setting-up-a-minecraft-server-using-systemd/
- https://nolte.github.io/minecraft-infrastructure/index.html
- https://unix.stackexchange.com/questions/302733/minecraft-server-startup-shutdown-with-systemd#303164
