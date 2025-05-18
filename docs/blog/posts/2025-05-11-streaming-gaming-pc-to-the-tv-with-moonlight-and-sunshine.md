---
date: 2025-05-11
categories:
 - linux
 - ubuntu
 - moonlight
 - sunshine
 - gaming
 - streaming
 - setup
title: Streaming gaming PC to the TV with Moonlight and Sunshine
---

[Streaming a Gaming PC straight to an Android TV](https://community.sony.ch/t5/android-tv/stream-your-gaming-pc-straight-to-your-android-tv-wirelessly/td-p/2158499)
turned out to be easier than expected; it essentially was much like with
[Steam Link](https://play.google.com/store/apps/details?id=com.valvesoftware.steamlink&hl=en)
in the past, and now the WiFi network is a lot faster!

<!-- more -->

## Sunshine

*[Sunshine](https://github.com/LizardByte/Sunshine?tab=readme-ov-file#sunshine) 
is a self-hosted game stream host for Moonlight*; the "*server*" to run on
the gaming PC.

### Installation

Installing Sunshing is as simple as downloading the **latest** (not *pre-release*)
[Debian package for Ubuntu](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2getting__started.html#debianubuntu):
from the [releases](https://github.com/LizardByte/Sunshine/releases) repository, along with its only dependency (`miniupnpc`):

``` console
$ sudo apt install miniupnpc
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following package was automatically installed and is no longer required:
  nvidia-firmware-550-550.120
Use 'sudo apt autoremove' to remove it.
The following NEW packages will be installed:
  miniupnpc
0 upgraded, 1 newly installed, 0 to remove and 5 not upgraded.
Need to get 17.0 kB of archives.
After this operation, 67.6 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu noble/universe amd64 miniupnpc amd64 2.2.6-1build2 [17.0 kB]
Fetched 17.0 kB in 0s (101 kB/s)     
Selecting previously unselected package miniupnpc.
(Reading database ... 492360 files and directories currently installed.)
Preparing to unpack .../miniupnpc_2.2.6-1build2_amd64.deb ...
Unpacking miniupnpc (2.2.6-1build2) ...
Setting up miniupnpc (2.2.6-1build2) ...
Processing triggers for man-db (2.12.0-4build2) ...

$ wget \
  https://github.com/LizardByte/Sunshine/releases/download/v2025.122.141614/sunshine-ubuntu-24.04-amd64.deb

$ sudo dpkg -i sunshine-ubuntu-24.04-amd64.deb 
Selecting previously unselected package sunshine.
(Reading database ... 492370 files and directories currently installed.)
Preparing to unpack sunshine-ubuntu-24.04-amd64.deb ...
Unpacking sunshine (2025.122.141614) ...
Setting up sunshine (2025.122.141614) ...
Not in an rpm-ostree environment, proceeding with post install steps.
Setting CAP_SYS_ADMIN capability on Sunshine binary.
/usr/sbin/setcap cap_sys_admin+p /usr/bin/sunshine-v2025.122.141614
CAP_SYS_ADMIN capability set on Sunshine binary.
Reloading udev rules.
Udev rules reloaded successfully.
Processing triggers for desktop-file-utils (0.27-2build1) ...
Processing triggers for hicolor-icon-theme (0.17-2) ...
```

There is also the option to install `sunshine.AppImage`.

Either way, a bit of
[initial setup](https://docs.lizardbyte.dev/projects/sunshine/latest/md_docs_2getting__started.html#initial-setup)
is required after installation; for X11 capture to work, you may need to
disable the capabilities that were set for KMS capture:

``` console
$ sudo getcap  $(readlink -f $(which sunshine))
/usr/bin/sunshine-v2025.122.141614 cap_sys_admin=p
$ sudo setcap -r $(readlink -f $(which sunshine))
$ sudo getcap  $(readlink -f $(which sunshine))

$ systemctl --user start sunshine
$ systemctl --user status sunshine
● sunshine.service - Self-hosted game stream host for Moonlight
     Loaded: loaded (/usr/lib/systemd/user/sunshine.service; disabled; preset: enabled)
     Active: active (running) since Sun 2025-05-11 00:04:25 CEST; 8s ago
    Process: 3266886 ExecStartPre=/bin/sleep 5 (code=exited, status=0/SUCCESS)
   Main PID: 3270182 (sunshine)
      Tasks: 18 (limit: 38298)
     Memory: 118.1M (peak: 120.3M)
        CPU: 715ms
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/sunshine.service
             └─3270182 /usr/bin/sunshine

May 11 00:04:29 rapture sunshine[3270182]: [2025-05-11 00:04:29.975]: Info:
May 11 00:04:29 rapture sunshine[3270182]: [2025-05-11 00:04:29.975]: Info: // Ignore any errors mentioned above, they are not relevant. //
May 11 00:04:29 rapture sunshine[3270182]: [2025-05-11 00:04:29.975]: Info:
May 11 00:04:29 rapture sunshine[3270182]: [2025-05-11 00:04:29.975]: Info: Found H.264 encoder: h264_nvenc [nvenc]
May 11 00:04:29 rapture sunshine[3270182]: [2025-05-11 00:04:29.975]: Info: Found HEVC encoder: hevc_nvenc [nvenc]
May 11 00:04:30 rapture sunshine[3270182]: [2025-05-11 00:04:30.107]: Info: Open the Web UI to set your new username and password and getting started
May 11 00:04:30 rapture sunshine[3270182]: [2025-05-11 00:04:30.108]: Info: File /home/ponder/.config/sunshine/sunshine_state.json doesn't exist
May 11 00:04:30 rapture sunshine[3270182]: [2025-05-11 00:04:30.109]: Info: Adding avahi service rapture
May 11 00:04:30 rapture sunshine[3270182]: [2025-05-11 00:04:30.110]: Info: Configuration UI available at [https://localhost:47990]
May 11 00:04:31 rapture sunshine[3270182]: [2025-05-11 00:04:31.050]: Info: Avahi service rapture successfully established.

$ systemctl --user enable sunshine
Created symlink /home/ponder/.config/systemd/user/xdg-desktop-autostart.target.wants/sunshine.service → /usr/lib/systemd/user/sunshine.service
```

### Configuration

Sunshine is configured via the web ui, which is available on
<https://localhost:47990> by default.

This is where clients are authorized when the try to connect (under the
**PIN** tab) and applications are specified so they can be launched from
the clients. A few applications are provided by default, including one to
launch *Steam Big Picture*.

## Moonlight

[Moonlight](https://moonlight-stream.org/) is the streaming **client** to
connect to the Sunshine server. The client should definitely be **not** on
the same X11/Wayland session, that creates an infinite lopp situation.

It can be used on a different Ubuntu PC, e.g. to play games on a light NUC PC
one could connect from it (client) to the gaming PC (server):

``` console
$ snap search moonlight
Name         Version  Publisher   Notes  Summary
moonlight    6.1.0    maxiberta✪  -      Stream games and other applications from another PC running Sunshine or GeForce Experience
heads-tails  1.0.13   technolog   -      Heads or Tails Multiplayer Crypto DexGames
 
$ sudo snap install moonlight
moonlight 6.1.0 from Maximiliano Bertacchini (maxiberta✪) installed
```

For an Android TV, one can simply install the Android app:
[Moonlight Game Streaming](https://play.google.com/store/apps/details?id=com.limelight&hl=en).
