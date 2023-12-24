+++
title = "Pop OS Setup"
author = "Nathanael Liechti"
date = "2023-12-24"
description = "How I like to setup Pop OS for work"
tags = [
  "linux",
]
+++

## Intro

Just a quick guide how I setup [Pop OS](https://pop.system76.com/) on my laptop to work with

## dconf 

Most stuff in Pop OS can be configured using dconf. 

You can view your current dconf config using `dconf dump /`.

To load some settings into dconf, use `dconf load / < dconf.conf`

For reference, this is the diff I would load:

```
[org/gnome/desktop/calendar]
show-weekdate=true

[org/gnome/desktop/datetime]
automatic-timezone=false

[org/gnome/desktop/input-sources]
current=uint32 0
sources=[('xkb', 'us')]
xkb-options=@as []

[org/gnome/desktop/interface]
color-scheme='prefer-dark'
show-battery-percentage=true

[org/gnome/desktop/privacy]
recent-files-max-age=30
remove-old-temp-files=true
remove-old-trash-files=true
report-technical-problems=false

[org/gnome/desktop/wm/keybindings]
minimize=['<Super>minus']

[org/gnome/mutter]
edge-tiling=false
workspaces-only-on-primary=true

[org/gnome/settings-daemon/plugins/media-keys]
custom-keybindings=['/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/', '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/']

[org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0]
binding='<Super>z'
command='/usr/bin/diodon'
name='Diodon'

[org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1]
binding='<Shift><Super>s'
command='flameshot gui'
name='Flameshot'

[org/gnome/settings-daemon/plugins/power]
power-button-action='suspend'
sleep-inactive-ac-timeout=1800
sleep-inactive-ac-type='suspend'
sleep-inactive-battery-timeout=1800
sleep-inactive-battery-type='suspend'

[org/gnome/shell]
welcome-dialog-last-shown-version='42.5'

[org/gnome/shell/extensions/dash-to-dock]
extend-height=false
manualhide=true

[org/gnome/shell/extensions/pop-cosmic]
clock-alignment='LEFT'
overlay-key-action='WORKSPACES'
show-applications-button=false
show-workspaces-button=false
workspace-picker-left=false

[org/gnome/shell/extensions/pop-shell]
activate-launcher=['<Super>space']
gap-inner=uint32 2
gap-outer=uint32 2
show-title=false
tile-by-default=true

[org/gnome/system/location]
enabled=true
```

## Apps

I try to install most apps using the Pop Shop, so that I don't have to care about them.

Here's a list of software I install:
- ungoogled-chromium
- spotify
- yubico authenticator
- diodon
- flameshot
- handbrake
- caffeine
- vpn client
- file sync client -> I keep my files on a cloud storage and only sync what's really necessary
- (chat apps) -> sometimes also used in the browser

All other tools shall be used in the browser if possible. Coding is done [remotly](https://technat.ch/posts/remote-coding).

## kDrive

Get the AppImage from [here](https://www.infomaniak.com/en/apps/download-kdrive), make it executable and put it somewhere in your PATH (eq /home/technat/.local/bin). It's important that you don't rename the executable.

Then add the following as `kdrive.desktop` into `~/.local/share/applications` to make it launchable by your application launcher:

```
[Desktop Entry]
Encoding=UTF-8
Version=1.0
Type=Application
Exec=/home/technat/.local/bin/Drive-3.5.5.20231213-amd64.AppImage
Name=kDrive
Comment=Infomaniak file sync
```

Finally you need the following dependencies:

```bash
sudo apt install fuse2fs
```
