---
title: "faultier"
---

The faultier is the answer to the Chicken and the Egg. If you have desinged anything in IT, you know that there is always at least one Chicken and Egg situation where you have to choose what depends on what. This is exactly what the faultier solves. It's a VM created by Terraform, but configured manually. Why? Because there are some services that just need to exist if you want to setup an automated environment.

## Infrastructure

Just a plain simple VPS docker host.

## DNS

For now, we have set `faultier.technat.ch` to point to the public IP of faultier.

## OS Configuration

The hornochse is setup with Debian Bullseye. The cloud_init file already created our Admin user and setup the SSH config.

### Updates

First of all doing updates:

```bash
apt update
apt dist-upgrade -y
```

### Base packages

Install some important packages:

```bash
apt install vim sudo bash-completion git wget curl stow restic git -y
```

### Firewall

Cloud Provider's Firewall service is used. Closed all ports, except SSH, ICMP and 80/443 for websites.

### Shell customizing

Not necessary but makes your life easier.

#### Bash aliases

Create the file `~/.bash_aliases` and add the following lines:

```bash
alias ll='ls -lahF --color=auto'
alias dir='dir --color=auto'
alias vdir='vdir --color=auto'
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'
alias d='docker'
alias db='docker build'
alias drrmti='docker run --rm -ti'
```

#### Vim

To edit files on the server directly we have installed vim. To have some Syntax highlithing and the ability to save the file using Sudo (when forgot to open it as sudo). I add the [server-vimrc](https://code.immerda.ch/technat/wall-e/-/blob/master/server-vim/.vimrc) file in `~/.vimrc` and `/root/.vimrc`.


### Git

You need git everywhere. To interact with private repositories we add an ssh-key and some configs.

First create a new ssh-key:

```bash
ssh-keygen -t ed25519 -C "service_faultier"
```

Then adjust the permissions:

```bash
chmod 700 ~/.ssh
chmod 400 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519.pub
```

Add the public key to your git account.

Lastly we need to add the following ssh config in `~/.ssh/config`:

```bash
Host code-ssh.immerda.ch
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

And lastly configure git by adding the following to `~/.gitconfig`:

```bash
[user]
	email = technat@technat.ch
	name = technat
[pull]
	rebase = true
```

### Environment Variables

A system always needs some environment variables to be configured.

Add them to `/etc/environment`:

```bash
EDITOR=vim
PATH=/usr/local/bin:/usr/bin:/bin:/sbin
```

### Mail relay

Postfix is setup and configured to send mails using the guide on the [cronjob page](https://wiki.technat.cloud/en/Linux/Cronjobs).

## Docker

All the services that need to run on faultier are run with docker. This because it's super easy to spin up the services, minimizes the time to deploy them and brings an abstraction.

### Installation

See [Install Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-debian-10) and [Install docker-compose](https://www.digitalocean.com/community/tutorials/how-to-install-docker-compose-on-debian-10).

### Docker Basic Config

The admin user should be able to interact with docker without using sudo:

```bash
sudo usermod -aG docker technat
```

### Docker Networking


We have a very simple networking concept. All services should be connected to the default `faultier` bridge network. If you must expose something to the internet, add port-mappings, otherwise only add the `EXPOSE` directive to the Dockerfile and the ports are accessable within the `faultier` network.

Create the `faultier` network like so:

```bash
docker network create faultier --ipv6 --subnet="2a01:4f9:c011:8f4::/64" --gateway="2a01:4f9:c011:8f4::1"
```

This will give all containers within the `faultier` network a public IPv6 address.

Note: docker-compose by default creates custom bridge networks for services. Add the following directive to a service definition to add them to the `faultier` network:

```yaml
networks:
 - faultier
```

And tell docker-compose at the bottom how the file is named:

```bash
networks:
  faultier:
    external: true
```

### Docker maintenance

To make sure we don't have containers and images lying around and not beeing use we run the following cronjob as `root`:

```
@weekly /usr/bin/docker system prune -f
```

### Docker volumes

We don´t use docker volumes but map everything that requires data to an external folder below `/volumes`.

So we need to create `/volumes`:

```bash
sudo mkdir /volumes
sudo chown -R technat:technat /volumes
sudo chmod -R 744 /volumes
```

### Docker configs

All docker configs for the different services are saved within [this repo](https://code.immerda.ch/technat/faultier).

Note: secrets must be excluded by using `.env` files or docker secrets.

If containers have a config file, put this into the git repository and map it into the container using relative path. If a container has data (like an sqlite file), map it using absolute path to `/volumes/yourcontainer/...`.

### Docker backups

Configs should all be in the git repository, so no need to backup them. The `/volumes` folder is backed up using [restic](https://wiki.technat.cloud/en/Linux/Backups).

The script we are using:

```bash
#!/usr/bin/bash
source /home/technat/.restic-env
restic backup /volumes
```

and the cronjob (run as root):

```bash
MAILTO=root@technat.ch
@daily /home/technat/restic-backup.sh
```

Make sure you have created the `/home/technat/.restic-env` file as explained in the backup guide for the credentials.

## Docker Services

This chapter lists all the docker services we have. Every service has at least a section about the initial deployment and how to restore the data, recreate the service later.

### Nginx-Proxy

We need a proxy to expose services to the internet. I'm using [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) for this.

#### Initial configuration

Once the config files are written you can just do `docker-compose up -d` in the directory. The missing config volumes will be created.

#### Restore

The certificates are mounted from `/volumes/nginx-proxy/certs` and the acme account settings in `/volumes/nginx-proxy/acme`. You could recreate them, but they are not that important as they will otherwise be recrated. Just spin the service up using all the configs in the git repo.

### wiki.technat.cloud

#### Initial configuration

Create the volume: `mkdir /volumes/jswiki/`.

Then spin it up, wait some seconds for the certificate to be there and then finish the installation in the browser.

#### Restore

Restore the `/volumes/jswiki/` volume with the `wiki.sqlite` file. Then just spin up the service.

### MinIO

#### Initial configuration

Download [docker-compose.yml](https://raw.githubusercontent.com/minio/minio/master/docs/orchestration/docker-compose/docker-compose.yaml) and [nginx.conf](https://raw.githubusercontent.com/minio/minio/master/docs/orchestration/docker-compose/nginx.conf).

The docker-compose file was edited a bit:

- the user and password are sourced from a `.env` file
- all the volumes were mounted unter `/volumes`.
- Of couse all containers are running in the `faultier` network.
- the minio servers have the environment variable MINIO_BROWSER_REDIRECT_URL: https://object-console.technat.cloud set to properly redirect to the console when accessed in a browser
- The `client_max_body_size 2M;` directive was added to the `vhosts/object.technat.cloud` file.

#### How it works

The nginx-proxy sends requests for object.technat.cloud and object-console.technat.cloud to the minio-nginx proxy. He listens on port 9000 for both domains. The console is redirect to the minio servers on port 9001 and the api to port 9000 on the minio servers. If you intentionally access object.technat.cloud using a browser instead of an s3 client, the minio servers have an environment variable telling them which url the console would be and redirect you there.

#### Restore

The data is all stored in `/volumes` folder, so you need to put them back before you pull up the service again.

### File Browser

#### Initial configuration

Create the volumes and touch the db file:

```bash
mkdir -p /volumes/filebrowser/data
touch /volumes/filebrowser/filebrowser.db
```

Then spin it up, wait some seconds for the certificate to be there and then finish the installation in the browser.

#### Restore

Restore the `/volumes/filebrowser/` volume with the `filebrowser.db` file and the data folder, then just spin up the service.

### Gitlab Runners

One of the main reasons for the faultier are gitlab runners. Gitlab runners are used to build the rest of the infrastructure.

[Good article](https://testdriven.io/blog/gitlab-ci-docker/)

#### technat.cloud

##### Initial configuration

Create the volume: `mkdir /volumes/runners/technat_cloud/`.

Spin up the container.

Then register the runner on gitlab:

```bash
docker-compose exec technat-cloud-runner \
    gitlab-runner register \
    --non-interactive \
    --url https://code.immerda.ch \
    --registration-token <YOUR-GITLAB-REGISTRATION-TOKEN> \
    --executor docker \
    --description "technat-cloud-runner" \
    --docker-image "docker:stable" \
    --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

You may want to update the config after the registration and change the following lines:

```yaml
concurrent = 4
check_interval = 3

[[runners]]
  limit = 2
  request_concurrency = 2
  [runners.cache]
    Type = "s3"
    Path = "k8s_at_hetzner"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "object.technat.cloud"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "us-east-1"
      Insecure = false
      AuthenticationType = "access-key"
      AccessKey = "gitlab-runner-cache"
      SecretKey = "saflajhlsdhfoue42io3lmefjlkwhri1"
```

Then restart the container using `docker restart technat-cloud-runner`.

All other runners work exactly the same, you can also register multiple runners

##### Restore

The runner has no data saved in `/volumes`. The config file contains a registration token which is stored within the git repo, you can just spin up the runner again. If you lose the token or the repo no longer exists, just delete the config file, reregister the runner somewhere and add the above configs back into the file.

## Maintenance

To update service do the following:

1. Replace the image tag with a newer one when specified
2. `docker-compose pull`
3. `docker-compose up -d --remove-orphans`
4. `docker image prune`
