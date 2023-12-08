# alleaffengaffen

![wip](https://img.shields.io/badge/Status-Work%20in%20Progress-important)

Kubernetes & Cloud lab of the-technat

We separate this logically and technically where possible to avoid damage to important stuff. Here are some timeless statements to make this happen and general guidelines to protect resources.

## General
- `alleaffengaffen.ch` and `technat.dev` are **reserved** domains for labing purposes
- banane@alleaffengaffen.ch is the mail address used for accounts and contacts within the lab
- dedicated accounts have to be used in all cases, exceptions have to be listed here:
  - exception 1: Github (there we have an organization, but the same account)
  - exception 2: Infomaniak (there we have an organization that holds mail + domain)
  - exception 3: everything that uses Github as IDP has two organizations from which one is named `alleaffengaffen`
- dedicated credit card that has a cost-limit
- relies on my private desaster recovery plan to keep access to the accounts 
- use [automatic nuking](https://github.com/alleaffengaffen/account-nuker) to keep costs low 

## Data
- a lab usually has no backup
- configs and docs that could serve as knowledge for the future shall be stored in Git repos in this organization
  - this means that this docs repo is just a collection point for text that has no other home
- secrets and sensitive informations are stored on Akeyless SaaS

## Networking
- tailnet `alleaffengaffen.org.github` should be use for all networking 
  - the [policy.hujson](./policy.hujson) in this repo controls the ACL of this tailnet
  - the domain `little-cloud.ts.net` is asossicated with this tailnet
- if not using the tailnet, you shall use the internet with appropriate zero-trust models, but don't hide labs behind firewalls
- use IPv6 wherever possible (which is currently impossible since github.com is not IPv6 capable)

## Bucketlist

Some ideas I had that I want to try out one day:
- A fully automated Cluster setup using Terraform and Kubespray that can be used to tinker around 
- Cluster-API management cluster using kind moved to a cloud-provider
- Postgresql Operator (Crunchydata or Zalando)
- Production-ready K3s Cluster on tailscale (for services that are better self-hosted) -> started in [axiom](https://github.com/the-technat/axiom)
- Try Crossplane to provision EKS clusters
- Install cluster using kubespray (and maybe Terraform + ansible provider inventory)
- Figure out if kOps should be used and with which platform (e.g add CI)
- Add more Account Nukers
- Find a generic notification service Workflows can use 
- Try Paralus for kubectl access
- Try Flatcar Linux (and get to know it)
- Linux from Scratch
- Kubernetes IPv6 (e.g use the /64 net of each host to create a truly public IPv6 cluster)
