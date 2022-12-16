+++
title =  "Nextcloud Maintenance"
date = "2022-07-31"
+++

## Backups

To backup a nextcloud instance you have to save the following directories:

- Webroot (usually under `/var/www`)
- Data dir (defined in nextcloud's config)
- DB Dump -> either using mysqldump or pgdump

To do a mysql dump you can use the following command:

```bash
sudo mysqldump -u root -p ncdb  -R -e --triggers --single-transaction > ncdb.sql
```

- ncdb is the name of you database
- root is the user that is allowed to connect to mysql using unix socket authentication

And for pgdump:

```bash
pg_dump -U www-data -w $DB_NAME -F p
```

- www-data is the linux system user that is allowed to connect to this DB

## Upgrades

Do upgrades in the following order:

1. create a backup (or Server snapshot)
2. Update the nextcloud using the Web Updater
3. Finish the update using the CLI
4. Do a system update
5. Reboot the server
5. Disable maintenace mode

Some cli commands that are usefull:

```bash
sudo -u www-data php /var/www/technat.cloud/occ maintenace:mode --on

sudo -u www-data php /var/www/technat.cloud/occ upgrade
sudo -u www-data php occ db:add-missing-indices

sudo -u www-data php /var/www/technat.cloud/occ maintenace:mode --off
```
