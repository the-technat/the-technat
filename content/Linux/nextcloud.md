---
title: "nextcloud"
---

My personal nextcloud on hcloud.

## Infrastructure

Managed by terraform, see [this file](https://code.immerda.ch/technat/technat_cloud/-/blob/master/terraform/nextcloud.tf).

## DNS

Managed by terraform, see [here](https://code.immerda.ch/technat/technat_cloud/-/blob/master/terraform/dns.tf#L20).

## OS Preparations

### Shell tipps

- Edit the `ENV_PATH` variable in `/etc/login.defs` to contain `/sbin:/usr/sbin`
- Add the file `~/.bash_aliases` with the following content:

```bash
alias off="sudo -u www-data php /var/www/cloud.technat.ch/occ maintenance:mode --off"
alias on="sudo -u www-data php /var/www/cloud.technat.ch/occ maintenance:mode --on"
alias occ="sudo -u www-data php /var/www/cloud.technat.ch/occ"
alias l="ls -lahF"
```

### External OS Disk

`/dev/sdb` is configured as our storage disk using LVM.

Start with partitioning:

```bash
technat@cloud:~$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.36.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x5795d6a5.

Command (m for help): g
Created a new GPT disklabel (GUID: EC377253-3EF3-3A4E-B3BA-3D1C12093F4F).

Command (m for help): p
Disk /dev/sdb: 150 GiB, 161061273600 bytes, 314572800 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: EC377253-3EF3-3A4E-B3BA-3D1C12093F4F

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-314572766, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-314572766, default 314572766):

Created a new partition 1 of type 'Linux filesystem' and of size 150 GiB.

Command (m for help): t
Selected partition 1
Partition type or alias (type L to list all): 30
Changed type of partition 'Linux filesystem' to 'Linux LVM'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Next we encrypt the disk using `luks` to make sure our data is encrypted at rest:

```bash
technat@nc:~$ sudo cryptsetup luksFormat -v /dev/sdb1

WARNING!
========
This will overwrite data on /dev/sdb1 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdb1:
Verify passphrase:
Passphrases do not match.
Command failed with code -2 (no permission or bad passphrase).
root@nc:~# cryptsetup luksFormat -v /dev/sdb1

WARNING!
========
This will overwrite data on /dev/sdb1 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /dev/sdb1:
Verify passphrase:
Key slot 0 created.
Command successful.
```

And open it again to work with it:

```bash
technat@nc:~$ sudo cryptsetup luksOpen /dec/sdb1 cryptlvm

Enter passphrase for /dev/sdb1:
technat@nc:~$ lsblk
NAME     MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda        8:0    0 38.1G  0 disk
├─sda1     8:1    0   38G  0 part  /
├─sda14    8:14   0    1M  0 part
└─sda15    8:15   0  122M  0 part  /boot/efi
sdb        8:16   0   10G  0 disk
└─sdb1     8:17   0   10G  0 part
  └─data 253:0    0   10G  0 crypt
sr0       11:0    1 1024M  0 rom
```

A password is okay but we don't want to enter our password on every boot, so let's generate a keyfile and add this as authentication method:

```bash
technat@nc:~$ sudo dd bs=512 count=4 if=/dev/random of=/keyfile iflag=fullblock
4+0 records in
4+0 records out
2048 bytes (2.0 kB, 2.0 KiB) copied, 0.000119724 s, 17.1 MB/s

technat@nc:~$ sudo chmod 400 /keyfile

technat@nc:~$ sudo cryptsetup luksAddKey /dev/sdb1 /keyfile
Enter any existing passphrase:

technat@nc:~$ sudo cryptsetup status cryptlvm
/dev/mapper/data is active.
  type:    LUKS2
  cipher:  aes-xts-plain64
  keysize: 512 bits
  key location: keyring
  device:  /dev/sdb1
  sector size:  512
  offset:  32768 sectors
  size:    20936671 sectors
  mode:    read/write
```

Now we can configure LVM:

```bash
sudo pvcreate /dev/mapper/cryptlvm
sudo vgcreate data /dev/mapper/cryptlvm
sudo lvcreate data -n nc -l 100%FREE
echo '/dev/data/nc /data ext4 defaults 0 1' | sudo tee -a /etc/fstab
sudo mkfs.ext4 /dev/data/nc
sudo mkdir /data
```

With a file-system in place we can mount the disk. Note that we also need to specify which keyfile to use when decrypting it. Otherwise the server hangs at boot prompting for our password:

```bash
technat@nc:~$ echo "cryptlvm /dev/sdb1 /keyfile" |sudo tee -a /etc/crypttab
```

Note: Now is a good time to reboot the server and check if the disk can be mounted properly.

### Fail2Ban

One last thing before we start, what do you do when someones tries to DDOS you? You use fail2ban to prevent it!

Note: Nextcloud also has a built-in brute-froce protection, you can also use this one.

[Reference Guide](https://marsown.com/wordpress/fail2ban-protection-nextcloud/)

You install fail2ban like so:

```bash
sudo apt install fail2ban -y
```

The following config files are needed to create a simple default config:

- `/etc/fail2ban/filter.d/nextcloud.conf`:
  ```bash
  [Definition]
  failregex = ^{.*"message":"Login failed: .* \(Remote IP: <HOST>\)".*}$
              ^{.*"message":"Login failed: '.*' \(Remote IP: '<HOST>'\)".*}$
  ignoreregex =
  ```

- `/etc/fail2ban/jail.d/nextcloud.local`:
  ```bash
  [DEFAULT]
  # Destination email address used solely for the interpolations in
  # jail.{conf,local,d/*} configuration files.
  destemail = root@technat.ch
  # Sender email address used solely for some actions
  sender = root@technat.ch
  # E-mail action. Since 0.8.1 Fail2Ban uses sendmail MTA for the
  # mailing. Change mta configuration parameter to mail if you want to
  # revert to conventional 'mail'.
  mta = mail
  action = %(action_mwl)s

  [nextcloud]
  backend = auto
  enabled = true
  port = 80, 443
  protocol = tcp
  filter = nextcloud
  maxretry = 10
  bantime = 36000
  findtime = 36000
  logpath = /data/nextcloud.log
  ```

Then just restart the service:

```
sudo systemctl restart fail2ban
```

Note: For remote mails to be sent when something happens, you need to configure a mail client as described [here](https://technat.ch/linux/cronjobs).

## Postgresql

The first thing on our main nextcloud installation is the database. I'm using postgresql here, but mariadb or mysql are also [supported](https://doc.nextcloud.com/server/20/admin_manual/configuration_database/linux_database_configuration.html).


Install the package:

```bash
sudo apt-get install postgresql -y
```

The systemd-service can be managed using those commands:

```bash
sudo systemctl start postgresql
sudo systemctl stop postgresql
sudo systemctl restart postgresql
```

### New Nextcloud Database

Now we will create a database and user for nextcloud. Postgresql uses peer authentication by default, that means we can use a socket and don't have to communicate over localhost. In addition, the user that is allowed to connect must by a local system user. So in our setup only `www-data` will be allowed to connect to our database.

For more information see the [Reference Docs](https://doc.nextcloud.com/server/stable/admin_manual/configuration_database/linux_database_configuration.html#postgresql-database).

So we create our database and user in a new sql-prompt: `sudo -u postgres psql -d template1`

```sql
CREATE USER "www-data" CREATEDB;
CREATE DATABASE ncdb OWNER "www-data";
\q
```

## PHP-FPM

Next after the database is PHP. Most guides start with the webserver first, but as PHP in my view is a depenency for the webserver we make sure PHP is up and running before the webserver tries to connect to PHP.

First configure the PPA Repository where the current PHP versions are located:

```bash
sudo apt install apt-transport-https lsb-release ca-certificates wget -y
sudo wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" |sudo tee /etc/apt/sources.list.d/php.list
sudo apt update
```

Then install the PHP-FPM package and the required PHP-modules for nextcloud:

```bash
sudo apt install php8.0-fpm -y
sudo apt install php8.0-curl php8.0-gd php8.0-mbstring php8.0-xml php8.0-zip php8.0-opcache php8.0-pdo php8.0-intl php8.0-gmp php8.0-imagick php8.0-bcmath php8.0-bz2 php8.0-pgsql -y
sudo apt install libmagickcore-6.q16-6-extra -y # if warning Module php-imagick in this instance has no SVG support. For better compatibility it is recommended to install it.
```

See the [Required Modules](https://doc.nextcloud.com/server/stable/adminmanual/installation/sourceinstallation.html#prerequisites-for-manual-installation) page in the nextcloud documentation for a list of all modules used.

When we have the packages installed, we edit the configuration for the postgresql extension in `/etc/php/8.0/mods-available/pgsql.ini`, to match with this sample config:

```bash
# configuration for PHP PostgreSQL module
#extension=pdo_pgsql.so
extension=pgsql.so

[PostgresSQL]
pgsql.allow_persistent = On
pgsql.auto_reset_persistent = Off
pgsql.max_persistent = -1
pgsql.max_links = -1
pgsql.ignore_notice = 0
pgsql.log_notice = 0
```

We also need to allow the `www-data` user to access the postgresql socket:

```bash
sudo usermod -aG postgres www-data
```

### FPM-Pool

PHP-FPM knows pools which is a logic abstraction for PHP. For nextcloud we create our own pool so that we can configure PHP settings only for nextcloud and use a custom socket.

The config file for our pool is `/etc/php/8.0/fpm/pool.d/nextcloud.conf` with the following content:

```bash
[nextcloud]
listen = /var/run/php8.0-fpm-nextcloud.sock
user = www-data
group = www-data
listen.owner = www-data
listen.group = www-data
; php.ini overrides
php_admin_value[disable_functions] = passthru,system
php_admin_value[memory_limit] = 512M
php_admin_value[upload_max_filesize] = 32G
php_admin_value[post_max_size] = 32G
php_admin_value[date.timezone] = Europe/Zurich
php_admin_flag[allow_url_fopen] = off
php_admin_value[redis.session.locking_enabled] = 1
php_admin_value[redis.session.lock_retries] = -1
php_admin_value[redis.session.lock_wait_time] = 10000
php_admin_value[session.save_handler] = redis
php_admin_value[session.save_path] = "unix:///var/run/redis/redis-server.sock"
php_admin_value[opcache.interned_strings_buffer] = 128
; Choose how the process manager will control the number of child processes.
pm = dynamic
pm.max_children = 200
pm.start_servers = 16
pm.min_spare_servers = 15
pm.max_spare_servers = 30
pm.process_idle_timeout = 10s
; some environment variables
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```

Note: the `php_admin_value` lines what basically override the php.ini configuration. As the php.ini file is so big and it's very hard to keep track of what has changed in there, I decided to leave it as default and only override specific keys through the fpm pool.

[Reference Docs](https://doc.nextcloud.com/server/stable/adminmanual/installation/sourceinstallation.html#php-fpm-configuration-notes)

Then restart the service:
```
sudo rm /etc/php/8.0/fpm/pool.d/www.conf # Delete the default pool
sudo systemctl restart php8.0-fpm.service
```

## NGINX

Now that PHP is running we can configure the webserver. Although Nextcloud doesn't officially support nginx I still want to use nginx as it is a very performant webserver and integrates well with PHP-FPM.

Start by installing the package with systemd service:

```bash
sudo apt install nginx -y
```

### HTTPS

Before continuing to configure a virtualhost let's take a second and get a TLS certificate (which is used in the next step). Obtaining a TLS certificate was a nightmare until Let's Encrypt came to be. Now it's fairly simple with their `certbot`:

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot certonly --nginx -d cloud.technat.ch --agree-tos -m root@technat.ch
```

This obtains a certificate for our domain using the http-01 challenge and saves the certificate in `/etc/letsencrypt/live/cloud.technat.ch/fullchain.pem`

Before continuing something to note: Let's Encrypt certificates are only valid for 90 days. To avoid an expired certificate we can automate the renewal similar to the way we obtained the cert:

Add the following cronjob to root using `sudo crontab -e `:

```
0 */12 * * * /usr/bin/certbot renew > /var/log/letsencrypt/certbot-renew.log
```

With this we can now continue to setup nginx.


### Virtualhost

Virtually any webserver uses virtualhosts for configuring multiple websites on one webserver. This isn't different for nextcloud. We place a virtualhost in `/etc/nginx/sites-available/cloud.technat.ch`.

The following one is based on a config found in the docs:

```bash
# Ref: https://doc.nextcloud.com/server/latest/admin_manual/installation/nginx.html

upstream php-handler {
    server unix:/var/run/php8.0-fpm-nextcloud.sock;
}

server {
    listen 80;
    listen [::]:80;
    server_name cloud.technat.ch;
    server_tokens off;

    # Enforce HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443      ssl http2;
    listen [::]:443 ssl http2;
    server_name technat.ch;
    server_tokens off;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_certificate     /etc/letsencrypt/live/cloud.technat.ch/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cloud.technat.ch/privkey.pem;

    # TLS settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers ECDH+AESGCM:ECDH+CHACHA20:ECDH+AES256:ECDH+AES128:!aNULL:!SHA1:!AESCCM;

    # HSTS settings
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;

    # set max upload size and increase upload timeout:
		client_max_body_size 32G;
    client_body_timeout 300s;
		fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;


		# Pagespeed is not supported by Nextcloud, so if your server is built
		# with the `ngx_pagespeed` module, uncomment this line to disable it.
		#pagespeed off;

		# HTTP response headers borrowed from Nextcloud `.htaccess`
		add_header Referrer-Policy                      "no-referrer"   always;
		add_header X-Content-Type-Options               "nosniff"       always;
		add_header X-Download-Options                   "noopen"        always;
		add_header X-Frame-Options                      "SAMEORIGIN"    always;
		add_header X-Permitted-Cross-Domain-Policies    "none"          always;
		add_header X-Robots-Tag                         "none"          always;
		add_header X-XSS-Protection                     "1; mode=block" always;

		# Remove X-Powered-By, which is an information leak
		fastcgi_hide_header X-Powered-By;

		# Path to the root of your installation
		root /var/www/cloud.technat.ch;

		# Specify how to handle directories -- specifying `/index.php$request_uri`
		# here as the fallback means that Nginx always exhibits the desired behaviour
		# when a client requests a path that corresponds to a directory that exists
		# on the server. In particular, if that directory contains an index.php file,
		# that file is correctly served; if it doesn't, then the request is passed to
		# the front-end controller. This consistent behaviour means that we don't need
		# to specify custom rules for certain paths (e.g. images and other assets,
		# `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
		# `try_files $uri $uri/ /index.php$request_uri`
		# always provides the desired behaviour.
		index index.php index.html /index.php$request_uri;

		# Rule borrowed from `.htaccess` to handle Microsoft DAV clients
		location = / {
				if ( $http_user_agent ~ ^DavClnt ) {
						return 302 /remote.php/webdav/$is_args$args;
				}
		}

		location = /robots.txt {
				allow all;
				log_not_found off;
				access_log off;
		}

		# Make a regex exception for `/.well-known` so that clients can still
		# access it despite the existence of the regex rule
		# `location ~ /(\.|autotest|...)` which would otherwise handle requests
		# for `/.well-known`.
		location ^~ /.well-known {
				# The rules in this block are an adaptation of the rules
				# in `.htaccess` that concern `/.well-known`.

				location = /.well-known/carddav { return 301 /remote.php/dav/; }
				location = /.well-known/caldav  { return 301 /remote.php/dav/; }

				location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
				location /.well-known/pki-validation    { try_files $uri $uri/ =404; }

				# Let Nextcloud's API for `/.well-known` URIs handle all other
				# requests by passing them to the front-end controller.
				return 301 /index.php$request_uri;
		}

		# Rules borrowed from `.htaccess` to hide certain paths from clients
		location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
		location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

		# Ensure this block, which passes PHP files to the PHP process, is above the blocks
		# which handle static assets (as seen below). If this block is not declared first,
		# then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
		# to the URI, resulting in a HTTP 500 error response.
		location ~ \.php(?:$|/) {
        # Required for legacy support
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php$request_uri;

				fastcgi_split_path_info ^(.+?\.php)(/.*)$;
				set $path_info $fastcgi_path_info;

				try_files $fastcgi_script_name =404;

				include fastcgi_params;
				fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
				fastcgi_param PATH_INFO $path_info;
				fastcgi_param HTTPS on;

				fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
				fastcgi_param front_controller_active true;     # Enable pretty urls
				fastcgi_pass php-handler;

				fastcgi_intercept_errors on;
				fastcgi_request_buffering off;
		}

		location ~ \.(?:css|js|svg|gif|png|jpg|ico)$ {
				try_files $uri /index.php$request_uri;
				expires 6M;         # Cache-Control policy borrowed from `.htaccess`
				access_log off;     # Optional: Don't log access to assets

        location ~ \.wasm$ {
            default_type application/wasm;
        }
		}

		location ~ \.woff2?$ {
				try_files $uri /index.php$request_uri;
				expires 7d;         # Cache-Control policy borrowed from `.htaccess`
				access_log off;     # Optional: Don't log access to assets
		}

		# Rule borrowed from `.htaccess`
		location /remote {
				return 301 /remote.php$request_uri;
		}

		location / {
				try_files $uri $uri/ /index.php$request_uri;
		}
}
```

Activate the virtualhost lile so:

```bash
sudo ln -s /etc/nginx/sites-available/cloud.technat.ch /etc/nginx/sites-enabled/cloud.technat.ch
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
```

## Redis

To improve Nextcloud Performance it's recommended to add a [memcache](https://doc.nextcloud.com/server/22/adminmanual/configurationserver/caching_configuration.html).

So here is how to install and configure redis as memcache for nextcloud:

```bash
sudo apt install redis-server php8.0-redis -y
```

We want redis to listen on a unix socket instead of a port. Update (comment, uncomment) the following lines in `/etc/redis/redis.conf`:

```
unixsocket /var/run/redis/redis-server.sock
unixsocketperm 770
port 0
# bind 127.0.0.1 ::1
```

NGINX needs access to that socket:

```bash
sudo usermod -a -G redis www-data
sudo systemctl restart redis
```

## Initial Nextcloud setup

Now we are ready to setup nextcloud. Get the latest application into the specifed webroot, fix permissions and grant the nextcloud permissions to our data folder:

```
cd /tmp
wget https://download.nextcloud.com/server/releases/nextcloud-23.0.0.zip
sudo apt install unzip -y
unzip nextcloud-23.0.0.zip
sudo mv nextcloud /var/www/cloud.technat.ch
sudo chown -R www-data:www-data /var/www/cloud.technat.ch
sudo chmod -R 770 /var/www/cloud.technat.ch
sudo mkdir /data/nc
sudo chown -R www-data:www-data /data/nc
sudo chmod -R 740 /data/nc
```

Now you can open your domain in a browser and configure the last settings:

Admin Username: everything but not admin

Admin Password: Long but not necessarily complex

Data Folder: /data/nc
DB User: www-data
DB: ncdb
DB Host: /var/run/postgresql

Congratulations! Your nextcloud is now ready to use!

Or when migrating the webroot from another server start by ziping the webroot there and copying it to the new host:

```bash
sudo zip -r cloud.technat.ch.zip /var/www/cloud.technat.ch/
```

Anyway the permissions must be correct:

```bash
sudo chown -R www-data:www-data /var/www/cloud.technat.ch
sudo chmod -R 770 /var/www/cloud.technat.ch
```

And then adjust the `config.php` of nextcloud:

* only `cloud.technat.ch` should be added to the trusted domains
* adjust the data path to `/data`
* adjust overwrite.cli.url to `https://cloud.technat.ch`
* ensure db connections are correct

When the webroot is correct also zip the data dir on the old server, copy it to the new server and unpack it.

Make sure the permissions are correct:

```bash
sudo chown -R www-data:www-data /data/nc
sudo chmod -R 770 /data/nc
```

## Post Setup

Although you are done now, it's highly recommended to configure some additional settings inside the Nextcloud UI and the config file.

### Config file

Add the following config values to your `config.php`:
```
'memcache.locking' => '\OC\Memcache\Redis',
'memcache.local' => '\OC\Memcache\Redis',
'memcache.distributed' => '\OC\Memcache\Redis',
'redis' => [
     'host'     => '/var/run/redis/redis-server.sock',
     'port'     => 0,
     'dbindex'  => 0,
     'password' => '',
     'timeout'  => 1.5,
],
```

This is because redis.

### Security & setup warnings

In the admin settings overview page Nextcloud has a list of warnings and errors it checks for you. Fix what the suggest.

Some cases are listed in the sub chapters here.

Default Phone Region missing

Head over to the Nextcloud config file at /var/www/cloud.technat.ch/config.php and add the following directive:

```json
'default_phone_region' => 'CH',
```

If you want to remove the "Get your own free account" link on the signin page add this to your config as well:

```json
'simpleSignUpLink.shown' => false,
```

### Nextcloud jobs using cron

[Reference Docs](https://doc.nextcloud.com/server/stable/adminmanual/configurationserver/backgroundjobsconfiguration.html)

Nextclouds runs some peridoc tasks. The most reliable way to run them is via cron.

To set this up add the following cron job with the command `sudo crontab -u www-data -e`:

```bash
*/5  *  *  *  * php -f /var/www/cloud.technat.ch/cron.php
```

Note: The php-cli has it's own php.ini in /etc/php/8.0/cli/php.ini so you might want to add keys like date.timezone there as well according to the php-fpm config.


## Nextcloud Admin UI Settings

### 2FA

Install from app store: Two-Factor Mail Provider, Two-Factor TOTP Provider

Enfore 2FA: true Enforced for groups: admin Not enforced for groups: none

### Password Policies

* Min Password lenght: 14
* User password history: 24
* Number of days until user password expires: 90
* Number of login attemps before the user account is blocked: 10
* forbid common passwords: true
* enfore upper and lower case characters: false
* enfore numeric characters: false
* enfore special characters: false
* check passwords against... : true

## OnlyOffice Docs

If you want to edit documents it's recommended to use something like onlyoffice doc. The community server is garbage, you will need the real server. For now we install the onlyoffice docs server on the same machine as the nextcloud. This requires some tweaks and special config.

The following docs are all helpful:

- [Docs](https://helpcenter.onlyoffice.com/installation/docs-community-install-ubuntu.aspx?_ga=2.121380878.782359554.1594636128-1157782750.1587541027)
- [Github Gist](https://gist.github.com/tavinus/4cd108fa6a76c2a11da81a0e5c552bd0)
- [custom nginx tutorial](https://tweenpath.net/install-onlyoffice-document-server-nginx-debian-10/)

### Postgresql

Onlyoffice needs a database, we intentionally installed nextcloud using postgresql so that we can now just create another DB:

```bash
sudo -i -u postgres psql -c "CREATE DATABASE onlyoffice;"
sudo -i -u postgres psql -c "CREATE USER onlyoffice WITH password 'password_here';"
sudo -i -u postgres psql -c "GRANT ALL privileges ON DATABASE onlyoffice TO onlyoffice;"
```

### Rabittmq

Rabittmq is a dependency of onlyoffice docs too:

```bash
sudo apt-get install rabbitmq-server -y
```

### Onlyoffice

Now we can install onlyoffice doc. But before we do this, we prepare a second domain, ssl cert and dhparam so that we can run onlyoffice docs on a separate nginx virtualhost. If we would run it in default it would listen on 80,443 without any virtualhost which doesn't work. There would be an https example in the docs but I don't trust it that it still doesn't use virtualhosts.

So let's get a second cert for `doc.technat.ch`:

```bash
sudo certbot certonly --nginx -d doc.technat.ch --agree-tos -m root@technat.ch
```

Once this is saved create a dhparam:

```bash
cd /etc/ssl/certs/
openssl dhparam -out dhparam.pem 4096
```

Then we can install onlyoffice from their repository:

```bash
sudo apt install gnupg2
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys CB2DE8E5
echo "deb https://download.onlyoffice.com/repo/debian squeeze main" | sudo tee /etc/apt/sources.list.d/onlyoffice.list
sudo apt-get update
sudo apt-get install onlyoffice-documentserver -y
```

Note: You have to enter the database password while installing the service.

And now for the custom domain, we need to adjust the virtualhost.

But first let's use their template:

```bash
cd /etc/onlyoffice/documentserver/nginx/
sudo cp ds-ssl.conf.tmpl ds.conf
```

And now edit the `ds.conf` file. There are three servers in there, one is only localhost, we don't touch this one. But the others that listen on all interfaces need adjustments:
- `server_name doc.technat.ch;` for both
- `listen 0.0.0.0:80;` needs to be replaced with `listen 80;`
- `listen [::]:80 default_server;` needs to be replaced with `listen [::]:80;`
- `listen 0.0.0.0:443 ssl;` needs to be replaced with `listen 443      ssl http2;`
- `listen [::]:443;` needs to be replaced with `listen [::]:443 ssl http2;`
- `ssl_certificate /etc/letsencrypt/live/doc.technat.ch/fullchain.pem;`
- `ssl_certificate_key /etc/letsencrypt/live/doc.technat.ch/privkey.pem`
- Enable the `ssl_dhparam` option by removing the comment

Then restart the nginx.

### Docs config

And finally configure a secret to use the server. Use your password generator to create one and then change the following (without removing what is already there) in `/etc/onlyoffice/documentserver/local.json`:

```json
{
  "services": {
    "CoAuthoring": {
      "token": {
        "enable": {
          "request": {
            "inbox": true,
            "outbox": true
          },
          "browser": true
        }
      },
      "secret": {
        "inbox": {
          "string": "secret_key"
        },
        "outbox": {
          "string": "secret_key"
        },
        "session": {
          "string": "secret_key"
        }
      }
    }
  }
}
```

Save the secret somewhere as your nextcloud instance needs to know it when you configure onlyoffice in the settings.

And then restart the service using `sudo supervisorctl restart all`.


### Fonts

As a last thing get some fonts onto the server.

```bash
sudo apt-get install ttf-mscorefonts-installer
sudo apt install wget cabextract fontforge
wget https://gist.githubusercontent.com/tavinus/1a92c79d790657d5b66546996dd006b9/raw/ttf-vista-fonts-installer.sh -q -O - | sudo bash
sudo /usr/bin/documentserver-generate-allfonts.sh
```

#### Custom fonts

1. Download them and put them in `/usr/share/fonts`
2. Run `sudo /usr/bin/documentserver-generate-allfonts.sh`

## Troubleshooting

### Troubleshoot onlyoffice issues

- `sudo supervisorctl restart all` also starts the example service, you want to this to be disabled: `sudo supervisorctl stop ds:example`
- Logs can be found in /var/log/onlyoffice/documentserver/
- `sudo supervisorctl status` shows you if the documentservices are running

### General troubleshooting

- /var/www should be owned by root:root (using 771)
