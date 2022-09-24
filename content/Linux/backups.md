---
title: "backups"
---

This page describes how to backup files using restic.

## Concept

We need backups, that's abvious. But how? My concept is to only backup data on a file based layer, no server backups or configuration backup as this should all be in a wiki or in a git repository.

As backup target I use Infomaniak Swiss Backup. Primarly using [restic](https://restic.readthedocs.io/en/latest/) and an s3 repository.

## Restic

### Init Repository

The restic s3 repository needs to be initialized. For this we need an env file contaning the following informations:

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export RESTIC_REPOSITORY="s3:server-url/bucket-name"
export RESTIC_PASSWORD="a-strong-password"
```

Add them to a file `~/.restic-env` and then source using `source ~/.restic-env`.

If this file is sourced you can init the repository using:

```bash
restic init
```

Note: You only do this once, then you will never ever thave to do that.

### File backup

You can always backup to the same s3 repository as long as you use restic. So please use this method on as many servers as you can so that deduplication start to shine ;).

For a file/directoy to the following to back it up:

```bash
restic backup ~
```

### Automated backups

You can install the following crontab to a user that has access to the files you want to backup:

```bash
42 * * * * . /home/technat/restic-backup.sh
```

Where the script is:

```bash
#!/usr/bin/bash
<<Header
Script:   restic-backup.sh
Date:     24.01.2022
Author:   Technat (Nathanael Liechti)
Version:  0.0
History   User    Date        Change
          technat 24.01.2022  Initial Version 1.0
Description: a backup script backing up the necessary files on a Nextcloud server using restic
Cronjob: @daily /var/www/restic-backup.sh
Dependency: restic, pg_dump
Docs: https://restic.readthedocs.io/en/latest/

© Technat

Header

###############
# Variables
###############

# Source vars
DATA_DIR="/data/nc"
WEBROOT_DIR="/var/www/technat.cloud"
DB_NAME="ncdb"

# Restic
ENV_FILE="/var/www/.restic-env"

###############
# Checks
###############
command -v restic >/dev/null 2>&1 || { echo >&2 "restic is not installed.  Aborting."; exit 1; }
if [ ! -f "$ENV_FILE" ]; then
    echo "$ENV_FILE does not exist. Aborting"
    exit 1
else
  source $ENV_FILE
fi
if [[ $RESTIC_PASSWORD == "" ]]
then
  echo >&2 "\$RESTIC_PASSWORD is not set. Aborting"; exit 1
fi
if [[ $RESTIC_REPOSITORY == "" ]]
then
  echo >&2 "\$RESTIC_REPOSITORY is not set. Aborting"; exit 1
fi
if [[ $AWS_ACCESS_KEY_ID == "" ]]
then
  echo >&2 "\$AWS_ACCESS_KEY_ID is not set. Aborting"; exit 1
fi
if [[ $AWS_SECRET_ACCESS_KEY == "" ]]
then
  echo >&2 "\$AWS_SECRET_ACCESS_KEY is not set. Aborting"; exit 1
fi

###############
# Main Script
###############

# Enable maintenance mode
php $WEBROOT_DIR/occ maintenance:mode --on

# backup data dir
restic backup -v $DATA_DIR

# backup webroot
restic backup -v $WEBROOT_DIR

# db dump
pg_dump -U www-data -w $DB_NAME -F p | restic backup -v --stdin --stdin-filename $DB_NAME"_dump.sql"

# disable maintenance mode
php $WEBROOT_DIR/occ maintenance:mode --off

# forget and prune old snapshots
restic forget -q --prune --keep-hourly 24 --keep-daily 7
```

This will do the following:

- Enable maintenance mode on nextcloud (remove that if you don't use this for a nextcloud)
- Backup data dir and webroot
- Do postgres db dump and save the file as well
- Delete old snapshots except 24 hour and 7 days old ones

### Restore Backup

You can list all backups using:

```bash
restic snapshots
```

And restore one:

```bash
restic restore 427696a3 --target /tmp/restore
```

### Cleanup

I recommend you start cleaning old snapshots regurarly using the following cronjob somewhere:

```bash
@weekly source /home/technat/.restic-env && usr/local/bin/restic forget -q --prune --keep-hourly 24 --keep-daily 7
```

## Further reading

- [Digitalocean Guide](https://www.digitalocean.com/community/tutorials/how-to-back-up-data-to-an-object-storage-service-with-the-restic-backup-client)
- [Docs](https://restic.readthedocs.io/en/latest/)
