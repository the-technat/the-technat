+++
title =  "SwayWM"
author = "Nathanael Liechti"
date = "2022-12-16"
description = "My Setup of a Wayland Compositor with all the tools needed to be productive, while only having what is really necessary (minimalistic approach)."
+++

My Setup of a Wayland Compositor with all the tools needed to be productive, while only having what is really necessary (minimalistic approach).

## Overview

This guide will show from A-Z the steps done, the configs modified and tools installed to get a decent working graphical environment, just with a wayland compositor and a lot of open-source community-tools.

As for all we need some concepts and principles. Mine are:

- If possible let default configuration as is and add custom config clearly visible -> include instead of copy and modify
- Don't change keybindings! Use the default ones if possible!
- If apps run on xwayland or wayland doesn't matter. We will need both, so we don't actively force all apps that are theoretically possible to run on wayland to do so (in some years, it might be possible...)
- I'm going to style everything using [Rubik](https://fonts.google.com/specimen/Rubik) for UI text, [FiraCode NerdFont](https://github.com/ryanoasis/nerd-fonts/tree/master/patched-fonts/FiraCode) for code and [Solarized](https://ethanschoonover.com/solarized/) for colors.
- This guide shows how things are done, where the tools and configs come from, most config snippets should be shown in this guide, so that someone can follow this and configure it's sway DE accordingly
- But: My actual config files can differ from what is in this guide. They are stored in my WALL-E repo: https://github.com/the-technat/WAll-E

## Drawbacks & To Do

Linus is always broken and nothing is perfect:

- Sway doesn't have the abbility to mirror outputs (https://github.com/swaywm/sway/issues/1666)
- Ranger cannot connect to sshfs, sftp, nfs, smb using my config (rarely used, but would be nice if we could do that)
- [sway-systemd](https://github.com/alebastr/sway-systemd) or [sway-services](https://github.com/xdbob/sway-services/)? We either have to integrate sway with systemd or start sway from systemd
- This guide is not yet fully transparent, some config files are not explained but just linked (see TODOs in text)

## Prerequisites

Some of the config files don´t have to be rewritten and can just be linked in. Therefore I maintain a repository with config files for different tools which I usually clone before configuring the environment:

```bash
git clone https://github.com/the-technat/WALL-E.git ~/WALL-E
```

In order to use the config files from this repo I further use `stow` to symlink the files to the correct location (this ensures further updates over the git repository), so it's a good idea to install this as well:

```bash
sudo pacman -S stow
```
Also install the fonts now:

```bash
yay -aS ttf-rubik nerd-fonts-fira-code
```

Then we can get started with a cup of ☕.

## Sway

The first thing we need from a basic arch installation is sway (the wayland compositor). In order to further configure sway we also need a browser (firefox), foot (default terminal emulator) and dmenu (default application launcher). Because dmenu runs on X11, we need xorg-xwayland to allow X11 apps to run within our wayland session.

Install them:

```bash
sudo pacman -S sway foot xorg-xwayland dmenu firefox
```

Now after sway is installed you may ask how to start it? Well it's as simple as tipping `sway` on the command line. Well it's as simple as that, but people tend to make it more complex. There are display managers, shell files, systemd and many more that all can do this for you. I'm planning on running sway as a systemd/user service in the future, without using a display manager. So I'm fine with running `sway` from the command line after logging in. One thing I do though is making sure that sway get's all the environment variables I set in `~/.config/environment.d/*.conf`. `environment.d` is the current most simple way to set environment variables so that they are set for all your user services and with a wrapper script also for all your sway inherited processes.

So let's write the following wrapper script `~/.local/bin/swayrun.sh` that inherits systemd/user env vars:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Export all variables
set -a
# Call the systemd generator that reads all files in environment.d
source /dev/fd/0 <<EOF
$(/usr/lib/systemd/user-environment-generators/30-systemd-environment-d-generator)
EOF
set +a

exec sway
```

Make it executable `chmod +x ~/.local/bin/swayrun.sh` and then you can finally run `swayrun.sh` which will start sway.

When sway has launched, use Super+Enter to open alacritty, Super+d to launch firefox. For a good introduction in basic navigating in sway, watch the video on sway's [homepage](https://swaywm.org/).

### Config

In a terminal, I copy the default sway config over to `~/.config/sway/config`:

```bash
mkdir -p ~/.config/sway
cp /etc/sway/config ~/.config/sway/config
```

Then reload the sway configuration using Super+Shift+c.

### Basic configuration directives

This chapter show some basic configuration directives I normally add to sway's config. Most of if comes from the [sway wiki](https://github.com/swaywm/sway/wiki).

#### Close windows

Change the binding to close windows:

```bash
bindsym ctrl+q kill
```

#### Swaylock / Swayidle

To lock your screen on keypress or after some idle time there are official packages for sway. The config snippet I use is taken from [here](https://code.krister.ee/lock-screen-in-sway/).

You need the following packages:

```bash
yay -aS swaylock-effects-git
sudo pacman -S swayidle
```

Then add the following section to sway's config:

```bash
#
# Swaylock
#
# https://code.krister.ee/lock-screen-in-sway/
set $lock swaylock \
    --clock \
    --indicator \
    --screenshots \
    --effect-scale 0.4 \
    --effect-vignette 0.2:0.5 \
    --effect-blur 4x2 \
    --datestr "%a %e.%m.%Y" \
    --timestr "%k:%M"
exec swayidle -w \
    timeout 600 $lock \
    timeout 570 'swaymsg "output * dpms off"' \
    resume 'swaymsg "output * dpms on"' \
    before-sleep $lock

set $lockman exec bash ~/.config/sway/lockman.sh
bindsym Shift+Alt+q exec $lockman
```

And add the script `~/.config/sway/lockman.sh`:

```bash
#!/bin/sh
# Times the screen off and puts it to background
swayidle \
    timeout 10 'swaymsg "output * dpms off"' \
    resume 'swaymsg "output * dpms on"' &
# Locks the screen immediately
swaylock --clock --indicator --screenshots --effect-scale 0.4 --effect-vignette 0.2:0.5 --effect-blur 4x2 --datestr "%a %e.%m.%Y" --timestr "%k:%M"
# Kills last background task so idle timer doesn't keep running
kill %%
```

Note: I used Shift+Alt+q as Binding because $mod+l is already mapped in sway's default config.

#### Clamshell mode

I'm using my laptop in a dockingstation when working at home. This is called `clamshell mode` as your laptop's screen is closed but the computer is running. The [sway docs](https://github.com/swaywm/sway/wiki#clamshell-mode) tell you how you can configure sway to use this.

If requires the following config snippet:

```bash
#
# Clamshell Mode
#
# https://github.com/swaywm/sway/wiki#clamshell-mode
set $laptop eDP-1
bindswitch --reload --locked lid:on output $laptop disable
bindswitch --reload --locked lid:off output $laptop enable
exec_always ~/.config/sway/clamshell.sh
```

And a script `~/.config/sway/clamshell.sh`:

```bash
#!/usr/bin/bash
if grep -q open /proc/acpi/button/lid/LID/state; then
    swaymsg output eDP-1 enable
else
    swaymsg output eDP-1 disable
fi
```

Depending on your laptop the output name for your internal screen might be different. You can find the corret name when running `swaymsg -t get_outputs` and looking for your internal display. You need to change the output Name in `~/.config/sway/clamshell.sh`

#### Application launcher

dmenu runs xorg-wayland. There is [bemenu](https://github.com/Cloudef/bemenu), a replacement for dmenu but native for wayland.

Install the programm:

```bash
sudo pacman -S bemenu # Chose option 2
```

Find the application launcher command starting with `set $menu ...` and replace it using the following command taken from [here](https://shibumi.dev/posts/wayland-in-2021/#dynamic-menu):

```bash
# Solarized Light
#set $menu bemenu-run -i \
#    -H 21 \
#    --tb "#eee8d5" \
#    --tf "#586e75" \
#    --fb "#eee8d5" \
#    --ff "#586e75" \
#    --nb "#eee8d5" \
#    --nf "#586e75" \
#    --hb "#eee8d5" \
#    --hf "#268bd2" \
#    --fbb "#eee8d5" \
#    --fbf "#586e75" \
#    --sb "#eee8d5" \
#    --sf "#586e75" \
#    --scb "#eee8d5" \
#    --scf "#586e75" \
#    --fn "font pango:rubik 11" \
#    "$@" -m "$(swayfocused)" -p ">"

# Solarized Dark
set $menu bemenu-run -i \
    -H 21 \
    --tb "#002b36" \
    --tf "#93a1a1" \
    --fb "#002b36" \
    --ff "#93a1a1" \
    --nb "#002b36" \
    --nf "#93a1a1" \
    --hb "#002b36" \
    --hf "#859900" \
    --fbb "#002b36" \
    --fbf "#93a1a1" \
    --sb "#002b36" \
    --sf "#93a1a1" \
    --scb "#002b36" \
    --scf "#93a1a1" \
    --fn "font pango:rubik 11" \
    "$@" -m "$(swayfocused)" -p ">"
```

#### Appearance

- Disable title bars using:
  ```bash
  default_border none
  default_floating_border none
  ```
- Set font for sway:
  ```bash
  font pango:Rubik 11
  ```
- Mark xwayland apps with an `[X]`:
  ```bash
  for_window [shell="xwayland"] title_format "<span>[X] %title゜</span>"
  ```

#### App autostart

The application launcher is good for programms we start when we use them. But for programms like Dropbox or the Nextcloud Client we might want to start them at boot time. For this I use [dex](https://github.com/jceb/dex). A programm for working with desktop entry files. Usually applications that want to be autostarted place their `.desktop` file in `~/.config/autostart/`. With `dex -a` you launch all apps that have a desktop-file in there.

Install the programm:

```bash
yay -aS dex
```

The following has to be in your sway config:

```
exec dex -a
```

#### Waybar

Sway ships with a default status bar which can be customized a bit. A much more customizable bar is [waybar](https://github.com/Alexays/Waybar).

Waybar has a seperate config in `~/.config/waybar/`. The `config` file defines all the modules which are displayed and the `style.css` stylies the modules.

So install waybar:

```bash
sudo pacman -S waybar
```

Then we get the default config:

```bash
mkdir -p ~/.config/waybar
curl -o config https://raw.githubusercontent.com/Alexays/Waybar/master/resources/config
```

Now we need to tell sway to use waybar as our status bar:

```bash
bar {
    swaybar_command waybar
}
```

Remove or comment the current `bar{}` block, then reload sway.

To customize waybar I used [this example](https://git.sr.ht/~begs/dotfiles/tree/1c92a56187a56c8531f04dea17c5f96acd9e49c4/item/.config/waybar) and just swapped the colors out with solarized theme, rubik as font and the arrows were rearranged a bit. For more information to customize waybar see their [wiki](https://github.com/Alexays/Waybar/wiki/Configuration).

My new config can be seen [here](https://code.immerda.ch/technat/wall-e/-/blob/master/waybar/.config/waybar/config) in the git repo, as well as the [style.css](https://code.immerda.ch/technat/wall-e/-/blob/master/waybar/.config/waybar/style.css).

Note: Some arrows have an `on-click` action. Those actions call a programm in the background. Some of them are not yet installed, so we need to install them now:

```bash
sudo pacman -S network-manager-applet
```

#### cliphist

A Clipboard History is useful, therefore we should add one. I'm using [cliphist](https://github.com/sentriz/cliphist) for this.

So first install it and deps:

```bash
sudo pacman -S jq xdg-utils wl-clipboard
yay -aS cliphist-bin
```

Then add the following snippet to sway's config to store copied items:

```bash
exec wl-paste --watch ~/.config/sway/cliphist.sh
```

Here I'm calling a wrapper script for cliphist. That's because I want to filter out copies from KeePassXC, they should not land in the history:

```bash
#!/usr/bin/env sh
app_id=$( swaymsg -t get_tree | jq -r '.. | select(.type?) | select(.focused==true) | .app_id'  )
if [[ $app_id != "org.keepassxc.KeePassXC" ]]; then
  cliphist store
fi
```

And then we can also add a keybinding to sway's config to pick items from the history:

```bash
# Solarized Dark
bindsym Shift+Ctrl+h exec cliphist list | \
    bemenu -H 21 \
    --tb "#002b36" \
    --tf "#93a1a1" \
    --fb "#002b36" \
    --ff "#93a1a1" \
    --nb "#002b36" \
    --nf "#93a1a1" \
    --hb "#002b36" \
    --hf "#859900" \
    --fbb "#002b36" \
    --fbf "#93a1a1" \
    --sb "#002b36" \
    --sf "#93a1a1" \
    --scb "#002b36" \
    --scf "#93a1a1" \
    --fn "font pango:rubik 11" \
    "$@" -m "$(swayfocused)" -p ">" | \
    cliphist decode | \
    wl-copy

# Solarized Light
#bindsym Shift+Ctrl+h exec cliphist list | \
#    bemenu -H 21 \
#    --tb "#eee8d5" \
#    --tf "#586e75" \
#    --fb "#eee8d5" \
#    --ff "#586e75" \
#    --nb "#eee8d5" \
#    --nf "#586e75" \
#    --hb "#eee8d5" \
#    --hf "#268bd2" \
#    --fbb "#eee8d5" \
#    --fbf "#586e75" \
#    --sb "#eee8d5" \
#    --sf "#586e75" \
#    --scb "#eee8d5" \
#    --scf "#586e75" \
#    --fn "font pango:rubik 11" \
#    "$@" -m "$(swayfocused)" -p ">" | \
#    cliphist decode | \
#    wl-copy
```

#### Multiple keyboard layouts

I have multiple layouts defined in my sway config:

```bash
input "type:keyboard" {
    xkb_layout "us,us(intl)"
}
```

And a keybinding to switch between them:

```bash
# Switch between keyboard layouts
# bindsym Alt+Space input type:keyboard xkb_switch_layout next
# Not working: https://github.com/swaywm/sway/issues/6011
# Workaround:
bindsym Alt+Space exec swaymsg -t get_inputs -r \
| jq '[.[] | select(.type == "keyboard") | .xkb_active_layout_index][0] - 1 | fabs' \
| xargs swaymsg 'input type:keyboard xkb_switch_layout'
```

Currently there is no indication on which layout we currently are, and I miss a notification that tells my to which layout I have changed now.

#### mako

How doesn't want notifications on your desktop? I'm using [mako](https://github.com/emersion/mako) as notification daemon.

Install it:

```bash
sudo pacman -S mako libnotiy
```

Then add a line that mako get's launched when starting sway:

```bash
exec mako
```

You can configure mako by adding a config file in `~/.config/mako/config`. Mine has some solarized configs and looks like that:

```bash
background-color=#073642
progress-color=source #268bd2
text-color=#839496
border-color=#073642
padding=10
border-size=3
border-radius=5
max-icon-size=32
icons=1
font=Rubik 11
actions=1
max-visible=5
default-timeout=6000
margin=36,28

[urgency=critical]
text-color=#dc322f
```

#### Wob

Sometime you may want to show a progress bar for something that is going on in your system. I have [wob](https://github.com/francma/wob) configured in my system to do exactly this. The next sections are going to depend on it and I highly recommend you to install it ;).

```bash
yay -aS wob
```

Wob needs to be launched, add the following to your sway config:

```bash
set $WOBSOCK $XDG_RUNTIME_DIR/wob.sock
exec rm -f $WOBSOCK && mkfifo $WOBSOCK && tail -f $WOBSOCK | wob
```

Exit sway and come back in, your wob should now be listening in the background. You can test it by sending any number from 0-100 into the pipe at `$XDG_RUNTIME_DIR/wob.sock`:

```bash
echo 50 > $XDG_RUNTIME_DIR/wob.sock
```

As you see it's a command, so the possibilities with it and scripts are endless...

#### Brightness control

To change the brigtness on your system there are [different](https://github.com/swaywm/sway/wiki/Useful-add-ons-for-sway#brightness) tools available. I'm using [brightnessctl](https://github.com/Hummer12007/brightnessctl/) for this task as it's simple and we can get the current brightness as percentage from it to pass it into wob ;)

First install:

```bash
sudo pacman -S brightnessctl
```

And then add keybindings to sway's config:

```bash
bindsym XF86MonBrightnessDown exec brightnessctl set 5%- | sed -En 's/.*\(([0-9]+)%\).*/\1/p' > $WOBSOCK
bindsym XF86MonBrightnessUp exec brightnessctl set +5% | sed -En 's/.*\(([0-9]+)%\).*/\1/p' > $WOBSOCK
```

#### Sound control

My [arch guide](https://wiki.technat.cloud/en/Linux/arch) shows how to install pipewire, but how do you actually control the volume and inputs?

I'm using [playerctl](https://github.com/altdesktop/playerctl) and [volumectl](https://github.com/vially/volumectl) for this.

Install it:

```bash
yay -aS volumectl
sudo pacman -S playerctl
```

And then set some keybindings:

```bash
# Media player controls
bindsym XF86AudioPlay exec playerctl play-pause
bindsym XF86AudioNext exec playerctl next
bindsym XF86AudioPrev exec playerctl previous

# Volume Control
bindsym XF86AudioRaiseVolume exec volumectl up
bindsym XF86AudioLowerVolume exec volumectl down
bindsym XF86AudioMute exec volumectl toggle

# Input control
bindsym $mod+Alt+Ctrl+Space exec ~/.config/sway/mic-mute.sh
```

`~/.config/sway/mic-mute.sh` is a small script that mutes or unmutes the default microphone and displays a notification accordingly:

```bash
#!/usr/bin/bash

# Check what state the mic is currently
muted=$(pactl get-source-mute @DEFAULT_SOURCE@)

# toggle this state and send according notification
if [[ $muted == "Mute: yes" ]]
then
  pactl set-source-mute @DEFAULT_SOURCE@ 0 | notify-send -u critical -t 1000 "Mic activated"
else
  pactl set-source-mute @DEFAULT_SOURCE@ 1 | notify-send -t 1000 "Mic deactived"
fi
```

#### GTK Styling

For apps using GTK, we can download and configure a theme.

Either download [this](https://www.gnome-look.org/p/1309911/) and [this](https://www.gnome-look.org/p/1311022/) or install `gtk-theme-numix-solarized` to get GTK themes.

For your local user this themes should be placed in `~/.themes`.

Then you can change the theme by adding the following to sway's config:

```bash
set $gnome-schema org.gnome.desktop.interface

exec_always {
    gsettings set $gnome-schema gtk-theme 'Solarized-Dark'
    gsettings set $gnome-schema icon-theme 'Your icon theme'
    gsettings set $gnome-schema cursor-theme 'Your cursor Theme'
    gsettings set $gnome-schema font-name 'Your font name'
}
```

Also see [swaywm wiki](https://github.com/swaywm/sway/wiki/GTK-3-settings-on-Wayland#setting-values-in-gsettings) for more informations about this.

#### Screen Sharing

For Screen Sharing to work you need the following package and it's dependencies:

```bash
yay -S xdg-desktop-portal-wlr
```

Once it's installed, add the following in `~/.config/environment.d/wayland.conf`:

```
XDG_SESSION_TYPE=wayland
XDG_CURRENT_DESKTOP=sway
```

Restart and then you should be able to share your screen using your browser.

[Upstream Guide](https://elis.nu/blog/2021/02/detailed-setup-of-screen-sharing-in-sway/)

#### Screen recording

For Screen recording on sway, you can use different programms and approaches. I wanted something that is based on shortcuts, doesn't necessarily have a GUI but can show the keys you press on the keyboard.

So I ended up writting two simple bash scripts based on the following programms:

```bash
yay -aS wshowkeys-git
yay -S wf-recorder
```

And others that should already be installed or will be installed with this guide:
- notify-send
- jq
- pactl

When you got all those tools and know they are working, you can open your sway config and define two shortcuts, one for starting the recording and one for stopping it. It should look something like this:

```bash
bindsym Ctrl+$mod+Alt+s exec ~/.config/sway/start-record.sh
bindsym Ctrl+$mod+Shift+S exec ~/.config/sway/start-record.sh --no-keys
bindsym Ctrl+$mod+Alt+e exec ~/.config/sway/end-record.sh
```

As you can see those shortcuts call a script that resides along sway's config. Here are those scripts:

`start-record.sh`
```bash
#!/sbin/env bash

## Checks
if ! command -v wf-recorder &> /dev/null
then
  notify-send -u critical -t 2000 "wf-recorder not installed"
  exit 1
fi
if ! command -v wshowkeys &> /dev/null
then
  notify-send -u critical -t 2000 "wshowkeys not installed"
  exit 1
fi

## Start wshowkeys
# if --no-keys is passed in, skip wshowkeys
if [ $1 != "--no-keys" ]
then
  wshowkeys -F 'Rubik 30' -b '#073642' -f '#839496' -s '#859900' -a bottom -m 10 -t 2 &
  pidWshowkeys=$(echo $!)
  echo $pidWshowkeys > /tmp/wshowkeys.pid
fi

## Find default audio source
defaultSource=$(pactl get-default-source)

## Start wf-recorder
currentFocusedMonitor=$(swaymsg -t get_outputs --raw | jq '. | map(select(.focused == true)) | .[0].name' -r)
wf-recorder -a $defaultSource -o $currentFocusedMonitor -f ~/Videos/record$(date +%d%m%y)-$(date +%H%M%S).mp4 &
pidWFrecorder=$(echo $!)
echo $pidWFrecorder > /tmp/wfrecorder.pid
notify-send -t 5000 "Started recording on $currentFocusedMonitor using Audio from $defaultSource"

exit 0
```

`end-record.sh`
```bash
#!/sbin/env bash

## See if we got some pids
pidWshowkeys=$(cat /tmp/wshowkeys.pid)
pidWFrecorder=$(cat /tmp/wfrecorder.pid)

if [ $pidWFrecorder != "" ]
then
  kill -INT $pidWFrecorder
  notify-send -t 5000 "Recording has been saved to ~/Videos/record$(date +%d%m%y)-$(date +%H%M%S).mp4"
  rm /tmp/wfrecorder.pid
  exit 0
else
  notify-send -u normal -t 2000 "No recording is running"
  exit 1
fi

if [ $pidWshowkeys != "" ]
then
  kill -SIGTERM $pidWshowkeys
  rm /tmp/wshowkeys.pid
fi
```

As you can see, the start script checks you have all the tools installed, then it launches the tools individually (thus the option to disable `wshowkeys` when using the other keybinding) and remembers the main PID of the processes. It uses the current active screen and current active input sink configured in pulseaudio.

To end the recording, the other script just takes the PIDs from a file and stops them gracefully.

Simple, easy to understand, separated tools for separated tasks. What do you want more?

## System Utilities

As the intro says, sway is not anything. We need some good tooling to be productive. We already modified sway to lock like we wanted it to look, but not we need tools.

For an Engineer the shell environment is probably the most important environment as it's his workplace, his workspace and can be customized however he wants.

So to start off, let's setup the shell environment and then go over to other applications.

### Terminal Emulator

When you use a graphical environment you don't have the TTY directly, so you need some sort of a terminal emulator to work with. My prefered terminal emulator therefore is [kitty](https://sw.kovidgoyal.net/kitty/).

Get it:

```bash
sudo pacman -S kitty
```

Then swap the default emulator in sway's config:

```bash
set $term kitty
```

After a reload kitty is the new default terminal emulator.

TODO: show how you came to your config file.

Now we can customize it to our likings, or use an already polished config:

```bash
cd ~/WALL-E
stow kitty
```

### Shell

A Terminal Emulator launches a shell by default. My prefered one is ZSH using the [oh-my-zsh](https://ohmyz.sh/) framework. Zsh is already installed and my default shell, when followed the [Arch Linux](arch.md) guide.

If not done, change your shell like so:

```bash
sudo pacman -S zsh
chsh -s /sbin/zsh
# close terminal and open a new one to take affect
```

Note: You must enter your password to change the shell of your user.

Then I get the framework:

```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Now you can edit the `.zshrc` file to your likings, change themes, add plugins and more. See the [docs](https://github.com/ohmyzsh/ohmyzsh/wiki) for how to do things.

TODO: Show what I have changed on my config file.

I already have my own config file that I'll just link in:

```bash
rm ~/.zshrc # remove the default config
cd ~/WALL-E
stow zsh # adds the .zshrc, .zshenv and a plugin folder with aliases
```

### GPG

I have setup a GPG identity as described [here](https://github.com/drduh/YubiKey-Guide) for different purposes. So one part of my Sway-DE is configuring gpg to be ready for git commit signing, ssh authentication and more.

#### Import GPG Identity

First we need to import our identity. If not already installed, get the required packages:

```yaml
yay -S gnupg
```

Then add the following config to `~/.gnupg/gpg.conf`:


```
# https://github.com/drduh/config/blob/master/gpg.conf
# https://www.gnupg.org/documentation/manuals/gnupg/GPG-Configuration-Options.html
# https://www.gnupg.org/documentation/manuals/gnupg/GPG-Esoteric-Options.html
# Use AES256, 192, or 128 as cipher
personal-cipher-preferences AES256 AES192 AES
# Use SHA512, 384, or 256 as digest
personal-digest-preferences SHA512 SHA384 SHA256
# Use ZLIB, BZIP2, ZIP, or no compression
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
# Default preferences for new keys
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
# SHA512 as digest to sign keys
cert-digest-algo SHA512
# SHA512 as digest for symmetric ops
s2k-digest-algo SHA512
# AES256 as cipher for symmetric ops
s2k-cipher-algo AES256
# UTF-8 support for compatibility
charset utf-8
# Show Unix timestamps
fixed-list-mode
# No comments in signature
no-comments
# No version in output
no-emit-version
# Disable banner
no-greeting
# Long hexidecimal key format
keyid-format 0xlong
# Display UID validity
list-options show-uid-validity
verify-options show-uid-validity
# Display all keys and their fingerprints
with-fingerprint
# Display key origins and updates
#with-key-origin
# Cross-certify subkeys are present and valid
require-cross-certification
# Disable caching of passphrase for symmetrical ops
no-symkey-cache
# Enable smartcard
use-agent
# Disable recipient key ID in messages
throw-keyids
# Default/trusted key ID to use (helpful with throw-keyids)
#default-key 0xFF3E7D88647EBCDB
#trusted-key 0xFF3E7D88647EBCDB
# Group recipient keys (preferred ID last)
#group keygroup = 0xFF00000000000001 0xFF00000000000002 0xFF3E7D88647EBCDB
# Keyserver URL
#keyserver hkps://keys.openpgp.org
#keyserver hkps://keyserver.ubuntu.com:443
#keyserver hkps://hkps.pool.sks-keyservers.net
#keyserver hkps://pgp.ocf.berkeley.edu
# Proxy to use for keyservers
#keyserver-options http-proxy=http://127.0.0.1:8118
#keyserver-options http-proxy=socks5-hostname://127.0.0.1:9050
# Verbose output
#verbose
# Show expired subkeys
#list-options show-unusable-subkeys
```

Then import my public key:

```bash
export KEYID=0x22391B207DAD6969
gpg --recv 0x22391B207DAD6969
```

And assign it ultimate trust:

```bash
$ gpg --edit-key $KEYID

pub  rsa4096/0x22391B207DAD6969
     created: 2022-05-29  expires: never       usage: C
     trust: unknown       validity: unknown
sub  rsa4096/0x2E8A914D0D5D6195
     created: 2022-05-29  expires: 2023-05-29  usage: A
sub  rsa4096/0xBC3ECF83D49EB271
     created: 2022-05-29  expires: 2023-05-29  usage: E
sub  rsa4096/0x9537A18EF9A0A4EB
     created: 2022-05-29  expires: 2023-05-29  usage: S
[ unknown] (1). Nathanael Liechti <technat@technat.ch>
[ unknown] (2)  Nathanael Liechti <root@technat.ch>

gpg> trust
pub  rsa4096/0x22391B207DAD6969
     created: 2022-05-29  expires: never       usage: C
     trust: unknown       validity: unknown
sub  rsa4096/0x2E8A914D0D5D6195
     created: 2022-05-29  expires: 2023-05-29  usage: A
sub  rsa4096/0xBC3ECF83D49EB271
     created: 2022-05-29  expires: 2023-05-29  usage: E
sub  rsa4096/0x9537A18EF9A0A4EB
     created: 2022-05-29  expires: 2023-05-29  usage: S
[ unknown] (1). Nathanael Liechti <technat@technat.ch>
[ unknown] (2)  Nathanael Liechti <root@technat.ch>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

pub  rsa4096/0x22391B207DAD6969
     created: 2022-05-29  expires: never       usage: C
     trust: ultimate      validity: unknown
sub  rsa4096/0x2E8A914D0D5D6195
     created: 2022-05-29  expires: 2023-05-29  usage: A
sub  rsa4096/0xBC3ECF83D49EB271
     created: 2022-05-29  expires: 2023-05-29  usage: E
sub  rsa4096/0x9537A18EF9A0A4EB
     created: 2022-05-29  expires: 2023-05-29  usage: S
[ unknown] (1). Nathanael Liechti <technat@technat.ch>
[ unknown] (2)  Nathanael Liechti <root@technat.ch>
Please note that the shown key validity is not necessarily correct
unless you restart the program.
```

#### SSH Authentication

We are using the [gpg-agent](https://github.com/drduh/YubiKey-Guide#ssh) here for ssh authentication.

First add a config in `~/.gnupg/gpg-agent.conf`:

```
# https://github.com/drduh/config/blob/master/gpg-agent.conf
# https://www.gnupg.org/documentation/manuals/gnupg/Agent-Options.html
enable-ssh-support
ttyname $GPG_TTY
default-cache-ttl 60
max-cache-ttl 120
pinentry-program /usr/bin/pinentry-curses
#pinentry-program /usr/bin/pinentry-tty
#pinentry-program /usr/bin/pinentry-gtk-2
#pinentry-program /usr/bin/pinentry-x11
#pinentry-program /usr/bin/pinentry-qt
#pinentry-program /usr/local/bin/pinentry-curses
#pinentry-program /usr/local/bin/pinentry-mac
#pinentry-program /opt/homebrew/bin/pinentry-mac
```

Then make sure your shell is launching the gpg-agent by adding the following to your rc file:

```
export GPG_TTY="$(tty)"
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
gpgconf --launch gpg-agent
```

I'm using the `gpg-agent` plugin for oh-my-zsh, it does basically the same.

When launched, you can observe, that the gpg-agent has automatically detected our gpg-sub-key for authentication and made sure it's available for SSH authentication:

```bash
ssh-add -L
```

Of course the yubikey needs to be plugged in for that, otherwise you won´t see any identity.

You can then still adjust your ssh config as you like, if you want to give some hosts a hint which key to use (to prevent ssh from trying all keys it finds) you can save the public key from the above output to a file and point `IdentityFile` to this public key file. See [here](https://github.com/drduh/YubiKey-Guide#optional-save-public-key-for-identity-file-configuration) for details to this trick.

#### Commit signing

Grab your public key:

```bash
gpg --armor --export $KEYID |wl-copy
```

And add it into your SCM account.

Then tell git to use this key for signing:

```bash
git config --global user.signinkey $KEYID
```

#### Switch yubikey

If you have imported your gpg identity on a laptop and are using your backup key instead of the primary one, you must first tell that gpg (with the backup key plugged in):

```bash
gpg-connect-agent "scd serialno" "learn --force" /bye
```

### Languages

On a terminal you may need (programming) languages. So here I list out some languages and their tools that I need for further tooling and working and how to tweak them.

TODO: explain why changes / tools are needed

Go:

```bash
sudo pacman -S go
# add them to your .zshenv
export GOROOT="/usr/lib/go"
export GOPATH="/home/technat/go"
export GOBIN=$GOPATH/"bin"
export PATH=$PATH:$GOBIN
```

Python:

```bash
sudo pacman -S python python-pip
```

NodeJS:

```bash
sudo pacman -S nodejs npm
```

Terraform:

```bash
sudo pacman -S terraform
yay -aS terraform-docs terraform-ls # language server
```

YAML:

```bash
sudo pacman -S yamllint yq
```

JSON:

```bash
sudo pacman -S jq
```

### vim

Now comes my editor. I use it to edit many files including the programming languages I installed above.

There are a bunch of plugins set in my `.vimrc` file. All the required tools should already by installed in the languages section, so we can simply link the .vimrc file and it will install all the plugins automatically:

```bash
cd ~/WALL-E
stow vim # adds .vimrc and a config for COC plugin
```

Unfortunatelly there is currently no way to install coc languageservers using the vimrc, so do this manually:

```bash
vim +CocInstall coc-go
```

Btw for plugins to be used we need [vim-plug](https://github.com/junegunn/vim-plug). It will install itself when starting vim for the first time as well as installing all the plugins listed in `.vimrc`.

### Ranger

[Ranger](https://github.com/ranger/ranger) is a terminal file manager written in Python.

I'm working on getting this my default file manager, so let's install it:

```bash
sudo pacman -S ranger
yay -S atool unzip mediainfo highlight w3m ffmpegthumbnailer python-pillow
yay -aS dragon-drop
```

To configure it get the default config into `~/.config/ranger/rc.conf` and [edit](https://github.com/ranger/ranger/wiki) it or use your existing config:

```bash
cd ~/WALL-E
stow ranger
```

#### Files opened with wrong programm

The `~/.config/ranger/rifle.conf` file tells ranger how to open files.


Here are some modifications I did on my way:

- `onlyoffice` needs to be replaced with `onlyoffice-desktopeditors` for documents to open correctly
- Added `mime ^image, has vimiv,     X, flag f = vimiv -- "$@"` to the images section

### Code (OSS)

For coding it's sometimes easier to use an editor like vs code instead of vim.

You install it like so:

```bash
sudo pacman -S code
```

Those are the extensions I like to install:

- Go (golang)
- Docker (ms-azuretools)
- Git Graph (mhutchie)
- GitLens (eamodio)
- Terraform (hashicorp)
- Vim (vscodevim)
- YAML (Redhat)
- MarkdownLint (DavidAnson)

Keybindings not default:

- Add Cursor Below -> Ctrl+Alt+Down
- Add Cursor Above -> Ctrl+Alt+Up
- Copy Line Up -> Shift+Alt+Up
- Copy Line Down -> Shift+Alt+Down


### grim & swappy

Screenshots under Wayland can be done with different tools. I use `grim` in combination with `swappy` to edit them on the fly. To make handling easier `grimshot` as a wrapper to grim is used.

Let's installt them:

```bash
sudo pacman -S grim swappy
yay -aS grimshot
```

The tools need some keybindings to do screenshots:

```bash
# Screenshot // Screenshot active display
bindsym --locked Ctrl+Alt+s exec /sbin/grimshot --notify save output - | swappy -f - & SLIGHT=$(light -G) && light -A 30 && sleep 0.1 && light -S $SLIGHT

# # Screenshot // Screenshot current window
bindsym Shift+Alt+s exec /sbin/grimshot --notify save active - | swappy -f - & SLIGHT=$(light -G) && light -A 30 && sleep 0.1 && light -S $SLIGHT

# Screenshot // Screenshot selected region
bindsym $mod+Shift+s exec /sbin/grimshot --notify save area - | swappy -f - && SLIGHT=$(light -G) && light -A 30 && sleep 0.1 && light -S $SLIGHT
```

As you can see the keybindings pass the screenshot to swappy, a tool to quickly edit screenshots. This tool can be customized in `~/.config/swappy/config`:

```
[Default]
save_dir=$(xdg-user-dir PICTURES)/screenshots
save_filename_format=screenshot_%Y%m%d-%H%M%S.png
show_panel=true
line_size=5
text_size=20
text_font='Rubik'
```

### Dropbox

To get dropbox running we install it:

```bash
yay -aS dropbox dropbox-cli
```

The `dropbox-cli autostart y` command places a .desktop file in `~.config/autostart` which will execute dropbox when sway is started.

Docs and further informations: [Arch Wiki Guide](https://wiki.archlinux.org/title/Dropbox)

### Nextcloud sync client

The nextcloud-client want's to save nextcloud credentials in gnome keyring. So this package has to be installed too. Nextcloud-Client will create a default keyring and ask for a master password when setting up the nextcloud-client.
Then on every login you will be prompted for this master password

Install it:

```bash
sudo pacman -S gnome-keyring nextcloud-client
```

### KeePassXC

```bash
sudo pacman -S keepassxc
```

Pro Tipp: Install the KeePassXC-Browser extension to directly paste passworts into login masks when linked.

### mutt

A terminal based mail client. Install it:

```bash
yay -S mutt
```

And then you need a config file in `~/.config/mutt/muttrc` that looks something like this:

```
#################
# General
#################
# Source secret config values from encryted file
# source "gpg -dq $HOME/.my-pwds.gpg |"
source .my-pwds # or unencrypted file
# content: set my_pass="password_here"
set my_user=technat@technat.ch
set realname="Nathanael Liechti"
set from="technat@technat.ch"
set use_from=yes
set ssl_force_tls=yes
set ssl_starttls=yes

#################
# IMAP
#################
set folder=imaps://mail.infomaniak.com:993/
set imap_user=$my_user
set imap_pass=$my_pass
set spoolfile=+INBOX
set imap_check_subscribed
set header_cache=~/.cache/mutt
set message_cachedir=~/.cache/mutt
unset imap_passive
set imap_keepalive=300
set mail_check=30
set postponed = +INBOX/Drafts

#################
# SMTP
#################
set smtp_pass=$my_pass
set smtp_url=smtps://$my_user@mail.infomaniak.com
```

### KVM

The tools needed:

```bash
sudo pacman -S virt-manager vde2 qemu qemu-arch-extra edk2-ovmf bridge-utils dnsmasq
```

Give yourself permissions for virtualization:

```
sudo usermod -G libvirt -a technat
```

Of course virtualization support has to be activated in the BIOS...

### USB Drives automount

The following articles are helpful:

- [Github Wiki](https://github.com/coldfix/udiskie/wiki/Usage)
- [Arch Wiki Article](https://wiki.archlinux.org/title/Udisks)

Install the following packages:

```bash
yay -S udisks2 udiskie
```

There are two ways of running `udiskie`:

1. Systemd-service as user
2. Exec in sway config

If you want option 1, install and enable the service:

```bash
yay -a udiskie-systemd-git
systemctl --user enable --now udiskie.service
```

Note: This version does not support a tray icon unless you edit the service file.

For option 2 add the following to your sway config:

```bash
exec udiskie -Nt &
```

If you notice that udiskie does not mount your thumb-driver you may want to check [here](https://github.com/coldfix/udiskie/wiki/Permissions) for permission errors.

### PDF Viewer

For PDFs you need a reader. There are many [options](https://wiki.archlinux.org/title/PDF,_PS_and_DjVu). I tried some of them and setteled on `apvlv`:

```bash
sudo pacman -S apvlv
xdg-mime default apvlv.desktop application/pdf
```

### Image Viewer

For Images yo uneed a viewer. There are many [options](https://wiki.archlinux.org/title/List_of_applications/Multimedia#Image_viewers). I tried some of them and setteled on `vimiv`:

```bash
yay -S vimiv
xdg-mime default vimiv.desktop image/jpeg
xdg-mime default vimiv.desktop image/png
```

### kubectl

```bash
yay -S kubectl
yay -S kubectx
```

### Docker

```bash
yay -S docker docker-compose
```

### blueman

A graphical environment for bluetooth. Install it:

```bash
yay -S blueman
```

### kmeet

Get the AppImage from [here](https://www.infomaniak.com/en/apps/download-kmeet), make it executable and put it somewhere in your PATH (eq /home/technat/.local/bin).

Then add the following as `kmeet.desktop` into `~/.local/share/applications` to make it launchable by your application launcher:

```
[Desktop Entry]
Encoding=UTF-8
Version=1.0
Type=Application
Exec=/home/technat/.local/bin/kmeet
Name=Kmeet
Comment=Infomaniak video-conferencing
```

### kDrive

Get the AppImage from [here](https://www.infomaniak.com/en/apps/download-kdrive), make it executable and put it somewhere in your PATH (eq /home/technat/.local/bin).

Then add the following as `kdrive.desktop` into `~/.local/share/applications` to make it launchable by your application launcher:

```
[Desktop Entry]
Encoding=UTF-8
Version=1.0
Type=Application
Exec=/home/technat/.local/bin/kdrive
Name=kDrive
Comment=Infomaniak file sync
```

### Teams

Sometimes used for meetings:

```bash
yay -S teams
```

### Thunderbolt Docks

Linux supports Thunderbolt Docks natively. See [here](https://wiki.archlinux.org/title/Thunderbolt) for some docs about it.

If you are curious or want to manually authorized devices, you can install `bolt` to inspect and deal with TB devices.

```bash
yay -S bolt
```

And then use it as `boltctl`.

### Set default apps

This section collects things that are opened with the wrong programm.

- Default browser: `xdg-settings set default-web-browser firefox.desktop`

## Further reading

- [https://wiki.archlinux.org/title/Wayland#GUI_libraries](https://wiki.archlinux.org/title/Wayland#GUI_libraries)
- [https://wiki.archlinux.org/title/List_of_applications/Documents](https://wiki.archlinux.org/title/List_of_applications/Documents)
