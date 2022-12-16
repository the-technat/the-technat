+++
title =  "gitea"
+++
Setup a private Git on a Debian 11 machine.

## Preparation: Data Disk
Gitea needs a place to store it´s repositories and files. This will be under `/var/lib/gitea`.  In order to have the data separated from the operating system a second disk `/dev/sdb` will be used for that. Also the database will have it´s data stored a `/var/lib/mysql` so this should also be on the second disk.

Here is how I did it with LVM:
```
sudo fdisk /dev/sdb
: g
: n (all defaults)
: T
: 31
: w
pvcreate /dev/sdb1
vgcreate git-data /dev/sdb1
lvcreate -L 10G git-data -n db
lvcreate -l 100%FREE git-data -n gitea
mkfs.ext4 /dev/git-data/db
mkfs.ext4 /dev/git-data/gitea
mkdir /var/lib/gitea
mkdir /var/lib/mysql
echo /dev/git-data/db /var/lib/mysql ext4 defaults 0 0 | tee -a /etc/fstab
echo /dev/git-data/gitea /var/lib/gitea ext4 defaults 0 0 | tee -a /etc/fstab
reboot
```

## Preparation: MySQL
Gitea needs a database. Gitea supports [different databases](https://docs.gitea.io/en-us/database-prep/). It´s initial setup is done like described here.

### Installation
Done as described here: https://www.digitalocean.com/community/tutorials/how-to-install-the-latest-mysql-on-debian-10

Note: As the data dir is already installed the mysql systemd service will fail to start as the permissions to the data dir may not be correct. I fixed this with the following permissions:

```
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod 750 /var/lib/mysql
sudo rm -rf /var/lib/mysql/*
sudo systemctl stop mysql
sudo systemctl start mysql
```

### Gitea Preparations
Now its time to setup a user and database for gitea as descibed in [their documentation](https://docs.gitea.io/en-us/database-prep/).


## Installation
The installation of gitea is done [from binary](https://docs.gitea.io/en-us/install-from-binary/) as there is no official package for debian.


## Initial Configuration
| Option | Value |
| ++++++- | +++-- |
| DB-Backend | MySQL |
| Host | 127.0.0.1:3306 |
| Username | gitea|
| Password | see vault |
| Database Name | giteadb |
| Charset | utf8mb4 |
| Site Title | Technat's Gitea |
| Repository Root Path | /home/git/gitea-repositoris |
| Git LFS Root Path | /var/lib/gitea/data/lfs |
| Run as Username | git |
| SSH Server Domain | git.technat.ch |
| SSH Server Port | 9999 |
| Gitea HTTP Listen Port | 3000 |
| Gitea Base URL | http://git.technat.ch/ |
| Log Path | /var/log/gitea/ |
| SMTP Host | mail.cyon.ch |
| Send Mail as  | git@technat.ch |
| SMTP Username | regenwurm@technat.ch |
| SMTP Password | see vault |
| Options | Disable Self-Registration |
| Hidden Email Domain | git.technat.ch |
| Administrator Username | wurm |
| Password | see vault |
| Email Address | technat@technat.ch |

The log path needs to be created before installing:
```
sudo mkdir /var/log/gitea/
chown git:git /var/log/gitea
chmod -R 750 /var/log/gitea
```

## Fail2ban
According to the docs fail2ban is setup on the tea server: https://docs.gitea.io/en-us/fail2ban-setup/
