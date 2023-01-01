+++
title = "Pop OS"
author = "Nathanael Liechti"
date = "2022-12-27"
description = "Install Pop OS on my daily driver notebook"
tags = [
  "linux",
  "popos",
  "gnome",
  "system76",
]
+++

# Intro

I love Arch Linux and will continue to use Arch as described in my [arch guide](./arch.md). But sometimes you are kept back by Wayland. Something doesn't work or something is currently broken due to rolling releases. That's the moment when another Distro is used. So this guide shows how I installed and configured Pop OS as my secondary linux distro, for cases where Wayland limits me.

# Concepts

Since it's my second distro and Pop OS comes with a lot of good things already preconfigured, I will apply some other principals here when configuring stuff:

- When ever possible let PopOS defaults, only change what is really necessary
  - Never ever manipulate the gnome shell! No custom themes or anything like that!
  - Don't modify shortcuts!
- Use the Pop!_Shop for applications, `apt` only for command line programms (e.g `docker` or `sl`)
- Snap or APT doesn't matter, as long as it's a click in the store and works.

# Bootstrap

The bootstrap of PopOS is really simple, as the GUI installer shows you everything you need to know. The only things we need to make sure, is that the disk is encrypted. Everything else is let as default.

# Settings

After the installation process is finished and we are in the OS itself, I page throuth the settings and change some things.

- Dock is completely disabled
- Hostname is changed to `axiom`

## Shell

As on my arch setup, the shell is configured with the same config files. I installed `kitty` and just stow my config files for the different tools. [oh-my-zsh](https://ohmyz.sh/) is installed too.

## Environment variables

I got my custom env vars that should be present everywhere they could be used. So I link my `systemd` folder to the correct location, so that my environment variables are picked up by systemd/User.

## Fonts

For different things I use some cool fonts which must be installed into the system:

- [Rubik](https://www.1001freefonts.com/rubik.font)
- [FiraCode Nerd Font](https://www.nerdfonts.com/font-downloads)

The easiest way is to just double-click everyone of them to let gnome install them.

## Sudo

The user setup with pop-os is already member of the sudo group, but we want to type password-less sudo commands.

So use `EDITOR=vim sudo visudo` to edit the suoders file and change the following line:

```
%sudo ALL=(ALL) NOPASSWD: ALL
```

## Yuibikey usage

The following tools need to be installed:

```bash
sudo apt install pcscd gnupg-agent gnupg2 scdaemon -y
```

And of course the agent's config needs to be stowed for it to work and then the agent must start.

# Applications

Most of them can be installed from the Shop. But some need to be installed manually or need special configuration. They are listed below here

## kDrive

Has to be installed as a binary using a local desktop file. First get the app [here](https://www.infomaniak.com/en/apps/download-kdrive) and place it into `~/.local/bin/kdrive`.

Then add the following desktop file into `~/.local/share/applications/kdrive.desktop`:

```
[Desktop Entry]
Encoding=UTF-8
Version=1.0
Type=Application
Name=kDrive
Exec=/home/technat/.local/bin/kdrive
Icon=/home/technat/.local/share/icons/kdrive.png
Comment=Infomaniak file sync
```

## kMeet

Similar process for kMeet, get the binary from [here](https://www.infomaniak.com/en/apps/download-kmeet) and place it into `~/.local/bin/kmeet`.

Then use the following desktop file in `~/.local/share/applications/kmeet.desktop`.

```
[Desktop Entry]
Encoding=UTF-8
Version=1.0
Type=Application
Name=kMeet
Exec=/home/technat/.local/bin/kmeet
Icon=/home/technat/.local/share/icons/kmeet.jpeg
Comment=Infomaniak Meetings
```

The icon was downloaded from the internet somewhere.

## GPaste

Multi-value clipboard. Can be installed from the shop, but needs some settings to be enabled for it to show up in the status bar.
