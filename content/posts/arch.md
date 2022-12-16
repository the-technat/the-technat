+++
title = "Arch Linux"
author = "Nathanael Liechti"
date = "2022-12-16"
description = "Install Arch Linux on my daily driver notebook"
tags = [
  "linux",
  "arch",
  "wayland",
]
+++

Install Arch Linux on my daily driver notebook.

## Components in this installation

Arch Linux is flexible and there are many guides out there that show you how to install it. During my research I found it unpleasant having to read the entire article before knowing what software the article shows to configure for diferent components. Therefore here's the list of what Software I use in my Arch Linux Installation:

* System Type: EFI
* Partitioning: LVM (ext4) on LUKS -> all data encrypted (except boot partition)
* Encryption: LUKS2, either using password, keyfile or FIDO2 device
* Bootloader: systemd-boot
* initramfs: systemd-init hooks
* SWAP: swapfile

## Disclaimer

I documented in this guide how I installed Arch Linux on my notebook to have a reference I can use if something goes wrong and I need to know how I configured it. It's also a preparation for the installation as I can test this guide on a virtual machine or another computer and if I know that these commands work, I can configure my system in one run. So don't expect to have a full guide showing any possibilities and tricks to follow allonw. Note that I'm also just one of many computer scients and not all have the same opinion concering how things are done.

So this guide is mostly for myself, but maybe someone finds it helpful too. Let me know if it was a help for you.

## Boot Medium

On a Linux Machine with a usb stick plugged in and internet access these commands will turn your usb stick `/dev/sdx` into a bootable Arch Linux medium:

```bash
wget https://theswissbay.ch/archlinux/iso/2021.02.01/archlinux-2021.05.01-x86_64.iso
cd Download
dd if=archlinux-2021.05.01-x86_64.iso of=/dev/sdx bs=4M
```

Or if you want to learn something new check out [this mutliboot software](https://www.ventoy.net/en/index.html).

## Preparations on live installation

Before installing Arch on the actual disk, there are some checks and configs to do on the live installation.

1. Output of the directory below should not be empty, then we are sure we booted in UEFI mode:

```bash
ls /sys/firmware/efi/efivars
```

2. The default terminal font is quite small and on most laptops it's hard to read it. We can change it:

```bash
ls /usr/share/kbd/consolefonts # list of all available fonts on live system
setfont ter-g20b
```

3. Plug in a cable and check with ping that you have an internet connection:

```bash
ping -c 5 technat.ch
```

4. If you want to use WiFi you can configure it for this live session. Replace `ssid1` with the name of your WiFi and `wlan0` with the name of your wifi card (see `ip link`). You will be asked for the password:

```bash
iwctl station wlan0 connect ssid1
```

5. enable NTP to have an up to date time during installation:

```bash
timedatectl set-ntp true
```

6. generate new pacman mirror list with http mirrors close to you (packages are signed with gpg keys, no needs for slower https):

```bash
pacman -Syy
reflector —protocol http -c Switzerland —sort rate —save /etc/pacman.d/mirrorlist
```

Now you should be good to go with the installation. Note that you'll download quite a lot of packages during the installation, so the faster your internet connection is, the less ☕ you'll need :)

## Partitioning

We are in our live system. But how can we install Arch now? Let's start by partitioning the disk. For this you need the correct drive. You can find it with `lsblk`:

```bash
NAME                          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                             8:0    0 476.9G  0 disk
├─sda1                          8:1    0   500M  0 part
└─sda2                          8:2    0 476.5G  0 part
sdb                             8:16   0 223.6G  0 disk
├─sdb1                          8:17   0   500M  0 part  /boot
└─sdb2                          8:18   0 223.1G  0 part
  └─cryptlvm                  254:0    0 223.1G  0 crypt
    ├─vgwall--e-vgcrypt--root 254:1    0    60G  0 lvm   /
    └─vgwall--e-vgcrypt--home 254:2    0  98.7G  0 lvm   /home
```

In my case it's going to be `/dev/sda`. For the rest of this installation all commands will use `/dev/sda`. If your drive is another one change that accordingly.

But how does the partitioning layout look like for an Arch installation?

Like that:

| Drive Letter    | Part Nr  | Size                  | Type          |
|+++++++++++++++--|+++++++++-|+++++++++++++++++++++--|+++++++++++++++|
| /dev/sda1       | 1        | 300M                  | 1 EFI System  |
| /dev/sda2       | 2        | Reminder of the space | 43 Linux LVM  |

As we have a UEFI system we just need one small boot partition for the bootloader. This is the unencrypted part of the installation. The second partition will be encrypted using LUKS before we use LVM to create logical volumes on top of the LUKS device.

Using fdisk, enter the partiton dialog and create a new fresh GPT signature, aka **wipe all partitions and data!**

```bash
fdisk /dev/sda
g
```

Still in the dialog type those commands to create the first partition:

```bash
n
enter
enter
+300M
t
1
```

And for the second partiton we need those commands:

```bash
n
enter
enter
enter
t
enter
43
```

Now we write the changes to the disk and exit the fdisk dialog:

```bash
p
w
```

### Encrypt LVM partition with LUKS

Before we create logical volumes we encrypt the second partition with LUKS:

```bash
cryptsetup luksFormat -v /dev/sda2
```
Type YES to continue and enter a strong password twice.

Now it's encrypted. But we need to install the system on it so let's open it again:

```bash
cryptsetup luksOpen /dev/sda2 cryptlvm
```

Note: when opening the device you have to give it a name I called mine `cryptlvm` but you can name it howevery you want. We will use this name later so remeber it.

This will make the LUKS device available as `/dev/mapper/cryptlvm`.

#### FIDO2 Key unlocking

I'm using Yubikeys, so for me a password is not what I want to unlock the device. So I close / lock the device again and run the following commands to configure my Yubikey as another valid auth mechanism:

```bash
systemd-cryptenroll --fido2-device=list
systemd-cryptenroll --fido2-device=auto /dev/sda2
```

Note: Only do this if you really know what you are doing. You need to enter your FIDO2 pin which you should have setup already. If not, read some articles about that before you do that and for now continue using a password.

Later on remove the password as valid auth option:

```bash
systemd-cryptenroll /dev/sda2 --wipe-slot=0
```
**Only do this if you have tested that your FIDO2 Key works and you have at least two fido keys configured.**

### LVM

On our encryped partition we can now setup LVM. I'm going to create multiple logical volumes like that:

| LV Name  | Size     | VG Name   |
|+++++++++-|+++++++++-|+++++++++--|
| root     | 40GB     | vgcrypt   |
| var      | 20GB     | vgcrypt   |
| tmp      | 5GB      | vgcrypt   |
| vm       | 300GB    | vgcrypt   |
| home     | 100GB    | vgcrypt   |

So I start by creating a physical volume (note the `cryptlvm` name specified earlier):

```bash
pvcreate /dev/mapper/crpytlvm
```

On top of this PV we create a volume group:

```bash
vgcreate vgcrypt /dev/mapper/cryptlvm
```

And then we need to define our logical volumes:

```bash
lvcreate -L 40GB vgcrypt -n root
lvcreate -L 20GB vgcrypt -n var
lvcreate -L 5GB  vgcrypt -n tmp
lvcreate -L 300GB vgcrypt -n vm
lvcreate -L 100GB vgcrypt -n home
modprobe dm_mod
vgscan
vgchange -ay
```

Note: It's not necessary to create so many logical volumes. You could also just do the `root` volume.
But for better separation and overview about where the space is used I created multiple ones.
Also I didn't fill up the entire space of my volume group.
This is usefull if I have to extend a logical volume in the future as extending is easier than shrinking.
But if you want you can also let one volume use the reminder of the space:

```bash
lvcreate -l 100%FREE vgcrypt -n home
```

If you ever have to shrink an LV and extend another one, see [this guide](https://wiki.technat.cloud/en/Kubernetes/lv_extend) in my Wiki for some tipps how to do this with the Arch ISO.

## Formatting

Now that partitioning is done we need to format our volumes. The `boot` partition should be FAT32, for the others you have multiple options, I use ext4.

```bash
mkfs.vfat /dev/sda1
mkfs.ext4 /dev/vgcrypt/root
mkfs.ext4 /dev/vgcrypt/var
mkfs.ext4 /dev/vgcrypt/tmp
mkfs.ext4 /dev/vgcrypt/home
mkfs.ext4 /dev/vgcrypt/vm
```

## Mount partitions

We're coming close to the inital bootstrap of Arch. For this we now need to mount our volumes in the live system. I'm doing this below `/mnt`, so that from the perspective of the installed system `/mnt` is `/`:

```bash
mount /dev/vgcrypt/root /mnt
mkdir -p /mnt/{home,boot,vm,tmp,var,etc}
mount /dev/vgcrypt/home /mnt/home
mount /dev/vgcrypt/vm /mnt/vm
mount /dev/vgcrypt/tmp /mnt/tmp
mount /dev/vgcrypt/var /mnt/var
mount /dev/sda1 /mnt/boot
```

## Generate fstab file

Before bootstraping let's create a `fstab` file for our new system. The Arch ISO has a tool called `genfstab` that generates the fstab file for you:

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
```

My `/mnt/etc/fstab` file then looks like this:

```bash
# /dev/mapper/vgcrypt-root
UUID=083b5377-f768-445b-81eb-0154ee77bb75       /               ext4            rw,relatime     0 1

# /dev/mapper/vgcrypt-home
UUID=21c4372a-88d5-406d-b7e2-a88aea95dedd       /home           ext4            rw,relatime     0 2

# /dev/mapper/vgcrypt-vm
UUID=39635044-d2b3-42fa-b24c-ebf1602d558d       /vm             ext4            rw,relatime     0 2

# /dev/mapper/vgcrypt-tmp
UUID=91858b84-ef95-45cc-bc83-eb60b22a8646       /tmp            ext4            rw,relatime     0 2

# /dev/mapper/vgcrypt-var
UUID=de61f80f-529b-4884-99e1-f034a2542f8f       /var            ext4            rw,relatime     0 2

# /dev/sda1
UUID=73FB-6274          /boot           vfat            rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro   0 2
```

Note: UUIDs will be different on every system.

## Bootstrap system

Now we are ready to get a system on our disk. In Arch there is a tool called `pacstrap` that does the heavy lifting for us.

To start we install the `base` package alongside the `linux` package (which is the actual kernel). Then we also install `lvm2` because the system needs lvm support. The `linux-firmware` package contains drivers for many hardware and is in my case used to get the correct WiFi drivers. The last `base-devel` package contains multiple tools for development and compiling. It's usually required for packages from the AUR (they are compiled locally) alongside the `linux-headers` which is also required for compiling some tools.

So we call pacstrap to install those tools on our mounted disk:

```bash
pacstrap -i /mnt base base-devel linux linux-firmware lvm2 linux-headers
```

Alternatively you could install the LTS-kernel. The LTS or Long-Time-Support kernel is known for it's focus on stability and it's security updates over many years. It can be a good idea to actually install both so you always have a fallback if your main kernel breaks something on an upgrade. However if something is wrong with your installation (if you mixed up your initramfs or you forgot to remove a mount from fstab for example) the LTS kernel won't help you as the root disk is still the same.

Anyway the new pacstrap command would be:

```bash
pacstrap -i /mnt base base-devel linux-lts linux-firmware lvm2 linux-lts-headers
```

### arch-chroot

After we've installed the base system we can change our root to `/mnt` so that we can do further configurations to the os-disk:

```bash
arch-chroot /mnt
```

## Install system packages

On top of the base system we are going to install several system packages:

- `vim`: To edit system files and because it's my favourite editor
- `intel-ucode` or `amd-ucode` Microcode update files for Intel | AMD CPUs
- `terminus-font` the font we are currently using is part of this package
- `networkmanager` as networking solution of choice (alternatives [here](https://wiki.archlinux.org/index.php/Network_configuration))
- `dnsutils` for DNS debugging capability
- `openssh` for the SSH client
- `iptables-nft` newer iptables, for details see [here](https://www.redhat.com/en/blog/using-iptables-nft-hybrid-linux-firewall)
- `bluez` and `bluez-utils`for bluetooth functionality
- `xdg-utils` and `xdg-user-dirs` for some default folders and utilities to work with desktops
- `ntfs-3g` for NTFS support
- `mtools` and `dosfstools` for support of FAT file systems
- `nfs-utils` for NFS support
- `cups` for the printing system
- `sudo` to run commands with super-user privileges
- `hplip` for HP printing support
- `gvfs` and `gvfs-smb` for an API to virtual file systems
- `pipewire` as linux sound server
- `pipewire-alsa`, `pipewire-pulseaudio` and `pipewire-jack` to route apps to pipewire

```bash
pacman -Syy
pacman -S vim intel-ucode terminus-font networkmanager dnsutils openssh iptables-nft bluez bluez-tuils xdg-utils xdg-user-dirs ntfs-3g mtools dosfstools nfs-utils cups sudo hplip gvfs gvfs-smb pipewire pipewire-alsa pipewire-pulse pipewire-jack
```

Choose wireplumber when asked what to use as dependency fir pipewire and confirm that we replace iptables with iptables-nft.

After the tools are installed, we enable some of the important services:

```bash
sudo systemctl enable NetworkManager
sudo systemctl enable bluetooth
sudo systemctl enable cups
sudo systemctl enable fstrim.timer
```

## Regional Settings

Once the packages are there we go on and configure our regional stuff. We start by generating our locales in `/etc/locale.gen`. Uncomment your appropriate language and then generate the language pack:

```bash
locale-gen
```

The configured language needs to be set in the `/etc/locale.conf` file as follows:

```bash
LANG=en_US.UTF-8
```

We installed the terminus-font package so let's make use of it by setting our font in the `/etc/vconsole.conf` file.
If you want to try a font use `setfont` to change the font, this will not persist over reboots. A list of all available fonts in the system can be seen in `/usr/share/kdb/consolefonts/`

```bash
FONT=ter-g20b
```

To set the timezone we just use a hardlink to `/etc/localetime` like so:

```bash
ln -sf /usr/share/zoneinfo/Europe/Zurich /etc/localtime
timedatectl set-ntp true
hwclock --systohc
```

I'm using the english (US) Layout, so theroretically I don't need to change anything on the keyboard layout. However because I'm from Switzerland and you sometimes need to write something in swiss german I need to type öüä quite a lot. So I switched from standard english (US) to english (Intl., with AltGr dead keys).
Anyway here's how you could change the default keyboard layout:

```bash
localectl list-keymaps # list all available keymaps
```

and then add the keymap to `/etc/vconsole.conf` as follows:

```bash
KEYMAP=us-acentos
```

## Sudo

Add the following to `/etc/sudoers` using `EDITOR=vim visudo` to allow the wheel group to run sudo commands:

```bash
%wheel ALL=(ALL) ALL #--> will prompt for password when using sudo
%wheel ALL=(ALL) NOPASSWD: ALL #--> won't prompt for password when using sudo
```

## initramfs

As the documentation for Arch Linux says the initramfs should automatically be generated when installing the base system with pacstrap. But the default HOOKS in the initramfs don't contain support for LVM and LUKS, so we add them.

I edit `/etc/mkinitcpio.conf` and change the HOOKS line to match that:

```bash
HOOKS=(base systemd keyboard sd-vconsole autodetect modconf block sd-encrypt lvm2 fsck filesystems)
```

If you want to know more about the order and the hooks for the initramfs I highly recommend reading [this page](https://wiki.archlinux.org/index.php/Mkinitcpio).

If we change something in this file, we need to regenerate the initramfs:

```bash
mkinitcpio -p linux
mkinitcpio -p linux-lts # if you have an LTS kernel
```

## root password

To login after the reboot we need to a root password:

```bash
passwd
```

## Hostname

Set the hostname of the machine like so:

```bash
echo WALL-E > /etc/hostname
```

And adjust the /etc/hosts file accordingly:

```bash
cat <<EOF >/etc/hosts
127.0.0.1 localhost
::1   localhost
127.0.1.1 WALL-E.localdomain WALL-E
EOF
```

## bootloader

At the beginning of the tutorial I said that this installation is a UEFI installation and that I'm going to use systemd-boot as my bootloader. There are mutliple options depending on your preference and your setup. See [here](https://wiki.archlinux.org/index.php/Arch_boot_process) for more information on other options.

With `systemd-boot` we initialize the bootloader first:

```bash
systemd-machine-id-setup
bootctl --path=/boot install
```

Now we need to create a bootloader entry for Arch. For this we need to know the UUID of the root device (the partition mounted on `/`). We get it with the following command:

```bash
uuid=$(blkid --match-tag UUID -o value /dev/sda2)
```

Note that we don't get the UUID of the actual `root` logical volume but instead of the second parttion. This is fine as we specify the LV to be used in the bootloader entry afterwards. If you don´t use LVM, you would get the UUID of the root partition.

So let's add an entry to our bootloader:

```bash
cat <<EOF >/boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux
initrd /intel-ucode.img
initrd  /initramfs-linux.img
# initrd /initramfs-linux-lts.img
options rd.luks.name=${uuid}=cryptlvm root=/dev/vgcrypt/root
EOF
```

And we also change the bootloader config to use the above entry as default:

```bash
cat <<EOF >/boot/loader/loader.conf
default arch
#default arch-lts
timeout 0
editor 0
EOF
```

The `editor 0` directive disables the option to add kernel parameters during boot. The `timeout 0`  waits 0 seconds before continuing to boot. So you won't see the bootloader with the different options at all, if you don't press space during boot.

## FIDO2 Key

If you have added a FIDO2 key for your LUKS partition, you have to configure the following in the `options` to make sure your root partition is unlocked using the fido2 key during boot:

```bash
options rd.luks.name=UUID=system rd.luks.options=fido2-device=auto root=/dev/system/root
```

In addition to that, you need to tell the systemd to automatically unlock the device. Add a line to `/etc/crypttab`:

```bash
system  /dev/sda2 fido2-device=auto
```

Note: only remove your passphrase key slot when you have multiple Yubikeys configured and made sure they all work to unlock the computer!

## Keyfile

On my machine I had a problem which causes the keyboard to not work during the early boot process (BIOS problem) even with the keyboard hook in initramfs.
So I used a workaround which uses a keyfile stored in the initramfs to unlock the LUKS partition. This is not very secure as the initramfs is stored on the `/dev/sda1` partition which is not encrypted and anyone with access to the disk could extract the keyfile and then decrypt your data. A better option would be to store the keyfile on a thumb drive and insert this everytime you boot. But I'm not at a point where I think this is necessary for me. If you want to do it you can read everything about it [here](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption).

So that's what I did to fix my problem:

```bash
dd bs=512 count=4 if=/dev/random of=/keyfile iflag=fullblock
chmod 400 /etc/mykeyfile
cryptsetup luksAddKey /dev/sda2 /keyfile
```

then in the `/etc/mkinitcpio.conf` file I added the following to the FILES and MODULES array:

```bash
FILES=(/keyfile)
MODULES=(ext4)
```

Regenerated the initramfs:

```bash
mkinitcpio -p linux
```

and then modified the options line in the boot entry by adding the keyfile:

```bash
options rd.luks.key=${uuid}=/keyfile
```

## Success?

So this is it for the installation. Now you can `exit` the arch-chroot environment and `reboot` the system to proofe that you successfully installed a working Arch Linux!

If the installation fails to boot, I recommend you boot back into the live system, decrypt the LUKS partition, mount the volumes again and take another look at your boot entry.

From my experience typos in the bootloader config are the most common cause of problems :=)

## Recommended first steps

Before using your system I recommend you do some basic steps, some of them are optional, some of them are quite important.

## Personal user

There are reasons why we shouldn't work with `root` day by day:

```bash
sudo pacman -S zsh
useradd -m -g users -G wheel,storage,power -s /usr/bin/zsh technat
passwd technat
```

## Networking

Now that NetworkManager is running, we canu se it to connect to WiFis:

```bash
nmcli device wifi list
nmcli device wifi connect SSID password password
```

More information about `nmcli` can be found [here](https://wiki.archlinux.org/index.php/NetworkManager#nmcli_examples). Note that there is also a more user-firendly `nmtui` that can also be used to interact with `NetworkManager`.

Well not always Networking (and especially WiFi) works as expected. So let's take a step back and go through all the things that could go wrong.

### Check drivers

The most important thing is to see wether the system has detected our network interface and can use it by loading the correct driver for it. Normally the linux kernel does a good job by picking the correct default driver for your network interface (that's why you have `linux-firmware` package). Let's check if he did:

```bash
lspci -k | grep -i net -A 3
```

You should see at least one network interface listed. My output is as follows:

```bash
00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection I219-LM (rev 21)
        Subsystem: Fujitsu Limited. Device 192c
        Kernel driver in use: e1000e
        Kernel modules: e1000e
02:00.0 Network controller: Intel Corporation Wireless 8260 (rev 3a)
        Subsystem: Intel Corporation Dual Band Wireless-AC 8260
        Kernel driver in use: iwlwifi
        Kernel modules: iwlwifi
```

If the NIC's are not listed you need to find the correct drivers manually. See [here](https://wiki.archlinux.org/index.php/Network_configuration/Ethernet#Device_driver) or [here](https://wiki.archlinux.org/index.php/Network_configuration/Wireless#Device_driver) for more information on how to get network drivers and load them.

### DNS

NetworkManager has multiple ways to resolve Names. One is to delegate DNS to the `systemd-resolved` service. For this the following has to be set:

```bash
sudo cat <<EOF >/etc/NetworkManager/conf.d/dns.conf
[main]
dns=systemd-resolved
EOF
sudo systemctl enable --now systemd-resolved
```

From now on DNS is completly managed by `systemd-resolved`.
Note: `/etc/resolv.conf` is now also managed by `systemd-resolved`. See [this page](https://wiki.archlinux.org/title/NetworkManager#/etc/resolv.conf) for how NetworkManager uses or doesn't use the `/etc/resolv.conf` file.

### Captive portals

See [NetworkManager](https://wiki.archlinux.org/title/NetworkManager#Configuration) for a list of solutions to make networmanager open a browser window when you connect to a network that has a captive portal.

### WPA2_Enterprise Networks

Universities and schools often use wpa2-enterprise networks where you have to authenticate yourself with credentials instead of a pre-shared-key.
NetworkManager can connect to such WiFi's but seems like you have to do it via a manual added connection.

So create a file in `/etc/NetworkManager/system-connections/[CONNECTION_NAME].nmconnection` and place the following content in it:

```bash
[connection]
id=[CONNECTION_NAME]
uuid=[UUID]
type=wifi
interface-name=[WIFI-INTERFACE NAME]
permissions=

[wifi]
mac-address-blacklist=
mode=infrastructure
ssid=[SSID]

[wifi-security]
auth-alg=open
key-mgmt=wpa-eap

[802-1x]
eap=peap;
identity=[USERNAME]
password=[PASSWORD]
phase2-auth=mschapv2

[ipv4]
dns-search=
method=auto

[ipv6]
addr-gen-mode=stable-privacy
dns-search=
method=auto

[proxy]
```

Replace the [] place holder with the correct values. An UUID can be generated with `uuidgen`.

Or see the [Wiki Page](https://github.com/wylermr/NetworkManager-WPA2-Enterprise-Setup) with a similar description.

## Bluetooth

The packages are already installed and the service is enabled. Use `bluetoothctl` to connect your devices.

For automatic power on of the bluetooth adapter at boot, edit the file `/etc/bluetooth/main.cf` and uncomment the following line:

```bash
[Policy]
AutoEnable=true
```

### Troubleshooting

If you have errors with connecting devices or powering on the adapter, you can try the following.

#### Drivers

To check if btusb is loaded and used run a `lsmod | grep btusb`

You should see that bluetooth is using it:

```bash
bluetooth             720896  43 btrtl,btintel,btbcm,bnep,btusb,rfcomm
```

#### Blocks

It's also possible that rfkill blocks the bluetooth module which can cause special behaviours. Check that with `rfkill list`:

My output was as following:

```bash
0: hci0: Bluetooth
        Soft blocked: yes
        Hard blocked: no
1: phy0: Wireless LAN
        Soft blocked: no
        Hard blocked: no
```

There is a "yes" by soft blocked bluetooth. I fixed this by running `rkfill unblock bluetooth`.

### Bluetooth Headsets

For bluetooth headsets see this [wiki page](https://wiki.archlinux.org/title/Bluetooth_headset#Disable_auto_switching_headset_to_HSP/HFP).

## Swapfile

My Arch Installation has no SWAP partition. That's because swapfiles are also good and they are way more flexible. Configure one like that:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
cp /etc/fstab /etc/fstab.bak
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab
cat /etc/fstab
```

Tipp: Read about SWAP in the [arch linux wiki](https://wiki.archlinux.org/title/Swap) or take a look at [zram](https://wiki.archlinux.org/title/Improving_performance#zram_or_zswap)

## AUR Helper

To install packages from the Arch User Repository you'll either need to do it manually or install a helper which get's your new package manager. It will be able to compile packages from the AUR as well as us pacman in the back to install regular packages. I use yay for that but there are other options as well:

```bash
sudo pacman -S git go
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

Note: Once yay is installed it can update itself in the future ;)

## What's next?

Okay now you have a decent system which you can work with. But there is still no graphical interface. Depending on your use case for the machine this might be fine. For me I need some sort of a graphical environment to work. If you are interested on how I have done that see my [Sway-DE Guide](https://wiki.technat.cloud/en/Linux/sway-de). There I noted how to do these things and much more.

## Fruther Reading

* [https://paedubucher.ch/articles/2020-09-26-arch-linux-setup-with-disk-encryption.html](https://paedubucher.ch/articles/2020-09-26-arch-linux-setup-with-disk-encryption.html)
