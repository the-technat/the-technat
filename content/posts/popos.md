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
- Snap or APT doesn't matter, as long as it's a click in the store

# Bootstrap

The bootstrap of PopOS is really simple, as the GUI installer shows you everything you need to know. The only things we need to make sure, is that the disk is encrypted. Everything else is let as default.

# Settings

After the installation process is finished and we are in the OS itself, I page throuth the settings and change some things.

- Dock is completely disabled
- Hostname is changed to `axiom`

# Applications

Most of them can be installed from the Shop. But some need to be installed manually.

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
