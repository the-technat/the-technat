+++
title =  "LVM - Tricks"
date = "2022-12-06"
+++

## Extend Logical Volumes

If using LVM and VMs we can manually extend the logical volumes. But for this the volume must be unmounted, so we need to use a live installation.

Shutdown the server you want to extend an LV and mount a live-iso. I'm using the [archlinux](https://archlinux.org/download) iso here.

Don't forget to resize the VM disk to be bigger before you boot the live ISO.

Now let's extend the PV:

```console
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

```console
e2fsck -ff /dev/mapper/os-var
lvextend -L+10G /dev/mapper/os-var
resize2fs /dev/mapper/os-var
```

If you don't want to resize the VM disk but instead steal some space from another LV you can:

```console
e2fsck -ff /dev/mapper/os-home
lvreduce --resizefs -L 10G /dev/mapper/os-home
e2fsck -f /dev/mapper/os-home
```
