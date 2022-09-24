---
title: "homelab_concept_old"
---

## Introduction

This page serves as a concept for my homelab. It defines some basic principles that are important for me when homelabing. Note though that it can evolve over time and that it's just a hobby.

## To Do

- [ ]: Explain technat.ch and technat.home at the beginning and restructure accordingly
- [ ]: Think about services hosted twice, once for the cloud and once at home
- [ ]: Backup Concept: how to get the data from Swiss Backup to the NAS? Or should we switch the order?
- [ ]: Backups: Is this DR friendly? Will that work?
- [ ]: Test out DR
- [ ]: See if we can make this page public

## Services

Here's an overview of all the services I have worked on in my homelab.

Web Apps:

- cloud.technat.ch: My person cloud (& doc.technat.ch)
- wiki.technat.ch: My personal wiki
- s3.technat.ch: My personal S3 storage
- vpn.technat.ch: My WireGuard VPN
- vault.technat.ch: My personal Vault

Websites:

- technat.ch: Portfolio website (ConcreteCMS)
- 360.technat.ch: different 360 panos (marzipano.net)
- js-buchsi.ch: Website of JS-Buchsi (Mocca CMS)
- foto.js-buchsi.ch: Photogallery of JS-Buchsi (Lychee)
- fvphub.ch: Website for FPV Hub in switzerland (ConcreteCMS)
- fpv-enthusiasts.ch: Website for our FPV club (ConcreteCMS)

Infrastructure Services:

- [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy): Reverse Proxy on faultier to expose services
- Backups


### External Services

The following services of my homelab are used externaly:

- github.com: SCM of choice
- app.terraform.io: IaaC Tool of choice
- Domain Registration: Infomaniak
- Mail Hosting: Infomaniak
- Public Nameservers: Hetzner Online
- Servers: Hetzner Online

## Hosting

It wasn't easy for me to find a place for my homelab. Although the name says it should be putten in your home, it's much easier to do this in the cloud and independent too. So the current state of the art is to put things in the cloud.

Which cloud?

The services list above already shows it, but a summary here:

Hetzner Online: Hoster from germany, used for Compute and Nameservers due to it's unbeatable prices

Infomanaik: Hoster in Switzerland, used for everthing I don't want to host myself (e.g Mail)

## Architecture

The following picture shows the big architecture I'm using:

![]()

We got two Kubernetes Clusters:

admin.technat.ch: My management cluster, powered by K3s, setup completely manually and using a single VM. It contains Cluster API and tools that are not a runtime dependency for important workload. The idea is to solve Chicken/Egg situations with this server but still have Kubernetes for everything. This includes:

- Cluster API -> if it doesn't work, the cluster can't be updated, but that's all
- (Argo CD) -> maybe to deploy the management apps itself to this cluster, but for sure not to deploy the apps on the other clusters
- Backup Tool for cluster itself and workload cluster (should be combined with Cluster API) -> no problem if it's not working for some days
- Hashicorp Vault as secrets solution -> potentially a runtime dependency as we can't work on new Apps / Update certificates when Vault becomes unaccessable
- Personal Wiki -> no problem if it's unavailable for some days
- Personal Vault -> no problem if it's unavaulable for some days, local cache is still working

The goal is to have a single node setup, that can be quickly restored to another location if necessary, to regain control over workload. But if it fails, no one will blame you or run because it's just the management layer that burnt up. Maybe in the future this cluster will be running at home or somewhere else.

technat.ch: The actual workload cluster where all my Apps are gona live I ever deploy.

## Backups

The thing with backups is, that they need to be stored in a location that can be accessed in a Desaster Recovery situation. So at the end, someone should put the Data into Infomaniak Swiss Backup.

But who is someone? Well it's the management cluster! The management cluster does the following:
- Backup itself to Infomaniak
- Backup the workload cluster to Infomaniak

So what happens when the Management Cluster is gone? You have to manually restore it from Backup. What if the workload cluster is gone? Cluster API can restore your cluster with your backup tool... At least that's the idea.

## Desaster Recovery

Desaster Recovery is composed of two factors:

- Something you know -> a list of important passwords
	- Master Password for vault
  - Pins of Yubikey
  - PGP Master Password
  - Backup Credentials
  - Cloud Provider Credentials (e.g to recreate backup creds)
- Something you have -> a Yubikey for 2FA

=> Those factors are placed somewhere in your home and somewhere in another home.

Hypothesis: With those two factors in a save place, it should be possible for someone to gain access to everything I have in my homelab and other places in the Internet

The following list thinks of different szenarious and how to react:
- Key Chain stolen: can´t login to differnet accounts anymore -> Backup Key
- Phone stolen: can't access my Database anymore -> vault.technat.ch, Master password and Yubikey
- Laptop stolen: can't ssh into servers, personal data is stolen -> personal data on nextcloud, login possible with Yubikey, ssh is still possible as the key is on your Yubikey
- Vault compromised: can´t access my passwords anymore -> Use passwordlist to restore backup of vault
- Nextcloud compromised: can´t access my data anymore -> restore data from backup either using vault or password list
- home network is compromised: restore everything from remote backup using offline password list
- home is physically compromised: restore everything from remote backup using remote password list

=> It's a pretty save DR. Many szenarious are covered and it's hard to lose complete track of them all.

### Dependency Chain

This section show up a dependency graph. This is important for rebuilding everything, as you always have dependencies.

1. My Personal Computer
2. Services on Infomaniak (Domains, Backup, Mail)
3. Management Cluster
4. Management Tools (Wiki, Vault)
5. Workload Cluster
6. Infrastructure Tools
...


## Design principles

### Naming

- technat.ch is my Domain for the public
  - Usernames
  - Mail
  - Services
  - root@technat.ch is the mail address to use for any system accounts and applications sending mails

### Security
- Use public repos where possible, hiding things with private repos is not the way to go.
- Use a vault as central store for credentials, let apps sync them from there.
- Infrastructur Accounts like TFE, AWS, Hetzner should use the Yubikey as second factor and not OTP

### No Automation

Automation is cool, but it is also more effort. For a homelab it makes no sense to automate everything. Build your stuff manually and learn something.

Of course that doesn't mean that we can’t try out things, but don't use it for productive services.

### No HA

technat.home: No HA! It's simply not possible to have everything HA, therefore forget about it from the beginning on.

technat.ch: Yes please HA if possible, but not a hard requirement, rather a soft requirement. But for serivces that you host for others, HA is a requirement.

### Container over VM

There are so many applications providing container images these days that it is so easy to spin up a new tool using containers rather than run it in a vm. So use containers where possible.

BUT: don't use docker! It's ugly and Kubernetes is better ;).

Althoug, there are some use-cases which are simply not designed to be deployed in a container. Databases for example. So there is always the option to use a VM for something if it's better suited.

### Maintenance Windows

We don't have regular maintenance windows, we patch frequently but not automatically.

### Documentation

All documentation is done in this wiki here. Do only the necessary things in git.

### Never ending story

This homelab is big and it's totaly unnecessary. At least most of it. So whenever you think you should go on to finish creating, forget about it. The funniest part of it is creating it, so don't try to be finished as soon as possible. It's a good place to learn and because it's fun it is even better. But keep in mind that maybe in some years this is no longer something you want to do, so don't write things into stone.
