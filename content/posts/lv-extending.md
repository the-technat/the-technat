+++
title =  "LVM - extend LV"
date = "2022-12-06"
+++

Containers have their temporare file system on the node under `/var/...`. This means that K8s worker nodes need to have a big `/var` if you mount this on a separate partition.

If using LVM and VMs we can manually extend the logical volume behind `/var`. But for this the volume must be unmounted, so we need to use a live installation.

First drain the node:

```bash
k drain node-0x --ignore-daemonsets --delete-empty-dir-data
```

Then we shutdown the node and mount a live-iso. I'm using the [archlinux](https://archlinux.org/download) iso here.

Don't forget to resize the VM disk to be bigger.

Now let's extend the PV:

```bash
fdisk /dev/sda
d
2
n
enter
enter
enter
No # we don't want to override the existing file signature
t
2
30
pvresize /dev/sda2
```

Then we can extend the var:

```bash
e2fsck -ff /dev/mapper/os-var
lvextend -L+10G /dev/mapper/os-var
resize2fs /dev/mapper/os-var
```

If you don't want to resize the VM disk but instead steal some space from another LV you can:

```bash
e2fsck -ff /dev/mapper/os-home
lvreduce --resizefs -L 10G /dev/mapper/os-home
e2fsck -f /dev/mapper/os-home
