+++
title = "Pop OS Setup"
author = "Nathanael Liechti"
date = "2024-03-10"
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
[net/launchpad/diodon/clipboard]
add-images=true
synchronize-clipboards=false

[org/gnome/desktop/calendar]
show-weekdate=true

[org/gnome/desktop/datetime]
automatic-timezone=false

[org/gnome/desktop/input-sources]
current=uint32 0
sources=[('xkb', 'us')]
xkb-options=@as []

[org/gnome/desktop/peripherals/mouse]
accel-profile='flat'
natural-scroll=false
speed=0.13970588235294112

[org/gnome/desktop/peripherals/touchpad]
two-finger-scrolling-enabled=true

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
idle-dim=false
power-button-action='suspend'
sleep-inactive-ac-timeout=1800
sleep-inactive-ac-type='nothing'
sleep-inactive-battery-timeout=1800
sleep-inactive-battery-type='suspend'

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
activate-launcher=['<Super>slash', '<Super>space']
active-hint-border-radius=uint32 5
gap-inner=uint32 0
gap-outer=uint32 0
tile-by-default=true

[org/gnome/system/location]
enabled=true
```

## Apps

I try to install most apps using the Pop Shop, so that I don't have to care about them.

Here's a list of software I install:
- ungoogled-chromium
- onlyoffice
- spotify
- yubico authenticator
- diodon
- flameshot
- handbrake
- caffeine
- vpn client
- file sync client -> I keep my files on a cloud storage and only sync what's really necessary

All other tools shall be used in the browser if possible. 

## kDrive

Get the AppImage from [here](https://www.infomaniak.com/en/apps/download-kdrive), make it executable and run it in a terminal.

Sign in and make sure it starts automatically. The app itself will move the binary somewhere on the PATH and add a desktop file to autostart automatically. You can then close the app in the terminal and remove the AppImage, after ensuring there's desktop file in `~/.config/autostart` 

You may need the following dependencies:

```bash
sudo apt install fuse2fs
```
