---
date: 2024-11-15
categories:
 - linux
 - ubuntu
 - installation
 - setup
 - intel
 - nuc
title: Ubuntu Studio 24.04 on Super Tuna (NUC PC)
---

This *would* be a very minimal journey of installing
[Ubuntu Studio 24.04](https://ubuntustudio.org/2024/04/ubuntu-studio-24-04-lts-released/)
on a relatively new mid-range Intel® NUC 13 Pro mini PC, had it not gone wahoonie-shaped...

<!-- more -->

## Preparation

The following components are chosen for a most compact, possibly *portable*
desktop setup, to be running 24x7 without taking much space:

- [ASUS Arena Canyon NUC13ANKI5](https://www.simplynuc.media/wp-content/uploads/2023/07/Arena-Canyon-i5-v5-Slim-NUC13ANKi5-v5-v1.1.pdf).
  with a 13th Generation Intel® Core™ i5-1340P and integrated Intel® UHD Graphics.
- [Kingston FURY Impact DDR4 Memory](https://www.kingston.com/en/memory/gaming/kingston-fury-impact-ddr4-memory)
  (32 GB, 3200 MHz, DDR4).
- [Kingston FURY Renegade PCIe 4.0 NVMe M.2 SSD](https://www.kingston.com/latam/ssd/gaming/kingston-fury-renegade-nvme-m2-ssd).
- 14" Full HD 1080p Portable Monitor 
  [Verbatim PM-14](https://www.verbatim-europe.com/en/portable-monitors/products/portable-monitor-14-full-hd-1080p-pm14-49590).
- Logitech
  [K400 Plus Wireless Touch Keyboard](https://www.logitech.com/en-eu/shop/p/k400-plus-touchpad-keyboard).

## Install Ubuntu Studio 24.04

Prepared the USB stick with `usb-creator-kde` and booted into
it, then used the “Install Ubuntu” launcher on the desktop.

1.  Plug the USB stick and turn the PC on.
1.  Press **`F8`** to select the boot device and choose the
    **UEFI: ...** option.
1.  In the Grub menu, choose to **Try or Install Ubuntu**.
1.  Select language (English) and then **Install Ubuntu**.
1.  Select keyboard layout (can be from a different language).
1.  Select the appropriate wireless or **wireless network**.
1.  Select **Try Ubuntu Studio**.
1.  **Disable screen locking** to prevent
    [KDE Plasma live lock screen rendering the session useless](https://launchpad.net/bugs/2062402):
    *   Press Alt-Space to invoke Krunner and type “System Settings”.
    *   From there, search for “Screen Locking” and
    *   deactivate “Lock automatically after…”.
1.  Select Type of install: **Interactive Installation**.
1.  Enable the options to
    *   **Install third-party software for graphics and Wifi hardware** and
    *   **Download and install support for additional media formats**.
1.  Select **Manual Installation**
    *   Use the arrow keys to navigate down to the **nvme0n1** disk.
    *   Set **nvme0n1p1** (300 MB) as **EFI System Partition** mounted
        on `/boot/efi`
    *   Set **nvme0n1p2** (60 GB) as **ext4** mounted on `/`
    *   Leave **nvme0n1p3** (60 GB) alone (to be used for Ubuntu 26.04)
    *   Set **nvme0n1p4** (3.88 TB) as **Leave formatted as Btrfs**
        mounted on `/home`
    *   Set **Device for boot loader installation** to **nvme0n1**
1.  Click on **Next** to confirm the partition selection.
1.  Confirm first non-root user name (`coder`) and computer
    name (`super-tuna`).
1.  Select time zone (seems to be detected correctly).
1.  Review the choices and click on **Install** to start copying files.
1.  Once it's done, select **Restart**
    (remove install media and hit `Enter`).

!!! note
    60 GB should be a enough for the root partition for the amount
    of software that tends to be installed in this PC.

### First boot into Ubuntu Studio 24.04

#### Disable Power Saving / Suspend

This desktop will need to continue running as normal even when left alone for
extended periods of time. To avoid interruptions, open the **Energy Saving**
section in **System Settings** and disable **Screen Energy Saving** and
**Suspend session**.

#### SSH Server

Ubuntu Studio doesn't enable the SSH server by default, but we want
this to adjust the system remotely:

``` console
# apt install ssh -y
# sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
# systemctl enable --now ssh
# systemctl daemon-reload
# systemctl restart sshd.service
```

Enable SSH from trusted hosts by adding them to `.ssh/authorized_keys` and copy content of `/etc/hosts` from a relevant host (e.g. `pi-z2`).

#### Check IP addresses on LAN & WLAN

This system will normally not be connected to the local wired network,
so it is important to take note of both IP addresses it has obtained
from the DHCP server:

``` console
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp86s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 48:21:0b:57:51:92 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.68/24 brd 192.168.0.255 scope global dynamic noprefixroute enp86s0
       valid_lft 85819sec preferred_lft 85819sec
    inet6 fe80::4a21:bff:fe57:5192/64 scope link 
       valid_lft forever preferred_lft forever
3: enx5c857e3e1129: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN group default qlen 1000
    link/ether 5c:85:7e:3e:11:29 brd ff:ff:ff:ff:ff:ff
4: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether e0:c2:64:39:17:73 brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
    inet 192.168.0.17/24 brd 192.168.0.255 scope global dynamic noprefixroute wlo1
       valid_lft 85820sec preferred_lft 85820sec
    inet6 fe80::56d1:3cea:139:daaf/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

## Install Essential Packages

Start by installing a **subset** of the
[essential packages](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#install-essential-packages), plus a few more 
that have been found necessary later (e.g. `auditd` to
[stop apparmor spew in the logs](2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#stop-apparmor-spew-in-the-logs)):

??? terminal "`# apt install ...`"

    ``` console
    # apt install gdebi-core wget vim curl geeqie playonlinux \
      exfat-fuse clementine id3v2 htop vnstat sox wine smem \
      python-is-python3 exiv2 rename scrot xcalib python3-pip \
      netcat-openbsd python3-selenium lm-sensors sysstat tor \
      unrar ttf-mscorefonts-installer winetricks icc-profiles \
      ffmpeg iotop-c xdotool inxi mpv screen libxxf86vm-dev \
      displaycal intel-gpu-tools redshift-qt auditd -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
      cabextract chromium-browser chromium-chromedriver exiftran fonts-wine
      fuseiso geeqie-common icoutils libasound2-plugins libaudit-common
      libaudit1 libauparse0t64 libcapi20-3t64 libegl-mesa0 libegl1-mesa-dev
      libgbm1 libgl1-mesa-dri libglapi-mesa libglx-mesa0 libgpod-common
      libgpod4t64 liblastfm5-1 liblua5.3-0 libmspack0t64 libmygpo-qt5-1
      libosmesa6 libpython3-dev libpython3.12-dev libsgutils2-1.46-2
      libutempter0 libwine libxatracker2 libxdo3 libxkbregistry0 libz-mingw-w64
      mesa-va-drivers mesa-vdpau-drivers mesa-vulkan-drivers python3-dev
      python3-exceptiongroup python3-h11 python3-natsort python3-outcome
      python3-sniffio python3-trio python3-trio-websocket python3-wsproto
      python3.12-dev redshift tor-geoipdb torsocks tree vim-common vim-runtime
      vim-tiny webp-pixbuf-loader wine64 xxd
    Suggested packages:
      audispd-plugins xpaint libjpeg-progs libterm-readline-gnu-perl
      | libterm-readline-perl-perl libxml-dumper-perl sg3-utils fancontrol
      read-edid i2c-tools libcuda1 winbind python-natsort-doc byobu | screenie
      | iselect libsox-fmt-all mixmaster torbrowser-launcher apparmor-utils nyx
      obfs4proxy ctags vim-doc vim-scripts indent vnstati q4wine wine-binfmt
      dosbox wine64-preloader
    Recommended packages:
      wine32
    The following NEW packages will be installed:
      auditd cabextract chromium-browser chromium-chromedriver clementine curl
      exfat-fuse exiftran exiv2 fonts-wine fuseiso gdebi-core geeqie
      geeqie-common htop icc-profiles icoutils id3v2 intel-gpu-tools inxi
      iotop-c libasound2-plugins libauparse0t64 libcapi20-3t64 libgpod-common
      libgpod4t64 liblastfm5-1 liblua5.3-0 libmspack0t64 libmygpo-qt5-1
      libosmesa6 libpython3-dev libpython3.12-dev libsgutils2-1.46-2
      libutempter0 libwine libxdo3 libxkbregistry0 libxxf86vm-dev libz-mingw-w64
      lm-sensors mpv playonlinux python-is-python3 python3-dev
      python3-exceptiongroup python3-h11 python3-natsort python3-outcome
      python3-pip python3-selenium python3-sniffio python3-trio
      python3-trio-websocket python3-wsproto python3.12-dev redshift redshift-qt
      rename screen scrot sox tor tor-geoipdb torsocks tree
      ttf-mscorefonts-installer unrar vim vim-runtime vnstat webp-pixbuf-loader
      wine wine64 winetricks xcalib xdotool
    The following packages will be upgraded:
      libaudit-common libaudit1 libegl-mesa0 libegl1-mesa-dev libgbm1
      libgl1-mesa-dri libglapi-mesa libglx-mesa0 libxatracker2 mesa-va-drivers
      mesa-vdpau-drivers mesa-vulkan-drivers vim-common vim-tiny xxd
    15 upgraded, 77 newly installed, 0 to remove and 71 not upgraded.
    Need to get 247 MB of archives.
    After this operation, 965 MB of additional disk space will be used.
    ```

### Brave browser

Install from the
[Release Channel](https://brave.com/linux/#release-channel-installation):

``` console
# curl -fsSLo \
  /usr/share/keyrings/brave-browser-archive-keyring.gpg \
  https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

# echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main" \
| tee /etc/apt/sources.list.d/brave-browser-release.list

# apt update && apt install brave-browser -y
```

### Google Chrome

Installing [Google Chrome](https://google.com/chrome) is as
simple as downloading the Debian package and installing it:

``` console
# dpkg -i google-chrome-stable_current_amd64.deb
```

### Continuous Monitoring

Install the
[multi-thread version](../../projects/conmon.md#deploy-to-pcs)
of the `conmon` script as `/usr/local/bin/conmon` and
[run it as a service](../../projects/conmon.md#install-conmon);
create `/etc/systemd/system/conmon.service` as follows:

```ini
[Unit]
Description=Continuous Monitoring

[Service]
ExecStart=/usr/local/bin/conmon
Restart=on-failure
StandardOutput=null

[Install]
WantedBy=multi-user.target
```

Then enable and start the services in `systemd`:

``` console
# systemctl enable conmon.service
# systemctl daemon-reload
# systemctl start conmon.service
# systemctl status conmon.service
```

If not already adjusted, add at least the following lines
to `/etc/hosts` so that `lexicon` is reachable:

``` console
192.168.0.6     lexicon
```

#### Hardware Sensors

All hardware sensors are supported out of the box:

``` console
# sensors -A
iwlwifi_1-virtual-0
temp1:        +37.0°C  

ucsi_source_psy_USBC000:001-isa-0000
in0:           0.00 V  (min =  +0.00 V, max =  +0.00 V)
curr1:         0.00 A  (max =  +0.00 A)

acpitz-acpi-0
temp1:        +43.0°C  

coretemp-isa-0000
Package id 0:  +43.0°C  (high = +100.0°C, crit = +100.0°C)
Core 0:        +36.0°C  (high = +100.0°C, crit = +100.0°C)
Core 4:        +34.0°C  (high = +100.0°C, crit = +100.0°C)
Core 8:        +35.0°C  (high = +100.0°C, crit = +100.0°C)
Core 12:       +34.0°C  (high = +100.0°C, crit = +100.0°C)
Core 16:       +40.0°C  (high = +100.0°C, crit = +100.0°C)
Core 17:       +40.0°C  (high = +100.0°C, crit = +100.0°C)
Core 18:       +40.0°C  (high = +100.0°C, crit = +100.0°C)
Core 19:       +40.0°C  (high = +100.0°C, crit = +100.0°C)
Core 20:       +42.0°C  (high = +100.0°C, crit = +100.0°C)
Core 21:       +42.0°C  (high = +100.0°C, crit = +100.0°C)
Core 22:       +41.0°C  (high = +100.0°C, crit = +100.0°C)
Core 23:       +41.0°C  (high = +100.0°C, crit = +100.0°C)

ucsi_source_psy_USBC000:002-isa-0000
in0:           0.00 V  (min =  +0.00 V, max =  +0.00 V)
curr1:         3.00 A  (max =  +0.00 A)

nvme-pci-0100
Composite:    +25.9°C  (low  = -20.1°C, high = +83.8°C)
                       (crit = +88.8°C)
Sensor 2:     +64.8°C  

```

There is no need to load the `drivetemp` kernel module because
the only storage in this Nuc PC are NVMe drives, which already
show up as the `nvme-pci-0100` controller above.

## System Configuration

The above having covered **installing** software, there are still
system configurations that need to be tweaked.

### APT respositories clean-up

Ubuntu Studio 24.04 seems to consistently need a little
[APT respositories clean-up](2024-09-14-ubuntu-studio-24-04-on-computer-for-a-young-artist.md#apt-respositories-clean-up); just comment out the last line
in `/etc/apt/sources.list.d/dvd.list` to let `noble-security` be
defined (only) in `ubuntu.sources`.

### Ubuntu Pro

When updating the system with `apt full-upgrade -y` a notice comes
up about additional security updates:

``` console
Get more security updates through Ubuntu Pro with 'esm-apps' enabled:
  libdcmtk17t64 libcjson1 libavdevice60 ffmpeg libpostproc57 libavcodec60
  libavutil58 libswscale7 libswresample4 libavformat60 libavfilter9
Learn more about Ubuntu Pro at https://ubuntu.com/pro
```

This being a new system, indeed it's not attached to an Ubuntu Pro
account (the old system was):

``` console
# pro security-status
3131 packages installed:
     1589 packages from Ubuntu Main/Restricted repository
     1539 packages from Ubuntu Universe/Multiverse repository
     3 packages from third parties

To get more information about the packages, run
    pro security-status --help
for a list of available options.

This machine is receiving security patching for Ubuntu Main/Restricted
repository until 2029.
This machine is NOT attached to an Ubuntu Pro subscription.

Ubuntu Pro with 'esm-infra' enabled provides security updates for
Main/Restricted packages until 2034.

Ubuntu Pro with 'esm-apps' enabled provides security updates for
Universe/Multiverse packages until 2034. There are 11 pending security updates.

Try Ubuntu Pro with a free personal subscription on up to 5 machines.
Learn more at https://ubuntu.com/pro
```

After creating an Ubuntu account a token is available to use with
`pro attach`:

``` console
# pro attach ...
Enabling Ubuntu Pro: ESM Apps
Ubuntu Pro: ESM Apps enabled
Enabling Ubuntu Pro: ESM Infra
Ubuntu Pro: ESM Infra enabled
Enabling Livepatch
Livepatch enabled
This machine is now attached to 'Ubuntu Pro - free personal subscription'

SERVICE          ENTITLED  STATUS       DESCRIPTION
anbox-cloud      yes       disabled     Scalable Android in the cloud
esm-apps         yes       enabled      Expanded Security Maintenance for Applications
esm-infra        yes       enabled      Expanded Security Maintenance for Infrastructure
landscape        yes       disabled     Management and administration tool for Ubuntu
livepatch        yes       warning      Current kernel is not covered by livepatch
realtime-kernel* yes       disabled     Ubuntu kernel with PREEMPT_RT patches integrated

 * Service has variants

NOTICES
Operation in progress: pro attach
The current kernel (6.8.0-47-lowlatency, x86_64) is not covered by livepatch.
Covered kernels are listed here: https://ubuntu.com/security/livepatch/docs/kernels
Either switch to a covered kernel or `sudo pro disable livepatch` to dismiss this warning.

For a list of all Ubuntu Pro services and variants, run 'pro status --all'
Enable services with: pro enable <service>

     Account: ponder.stibbons@uu.am
Subscription: Ubuntu Pro - free personal subscription
```

Now the system can be updated *again* with `apt full-upgrade -y`
to receive those additional security updates:

``` console
# apt full-upgrade -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following upgrades have been deferred due to phasing:
  krb5-locales libboost-chrono1.83.0t64 libboost-filesystem1.83.0
  libboost-iostreams1.83.0 libboost-locale1.83.0
  libboost-program-options1.83.0 libboost-thread1.83.0 libgssapi-krb5-2
  libk5crypto3 libkrb5-3 libkrb5support0 libldap-common libldap2
  python3-distupgrade ubuntu-release-upgrader-core
  ubuntu-release-upgrader-qt
The following packages will be upgraded:
  ffmpeg libavcodec60 libavdevice60 libavfilter9 libavformat60 libavutil58
  libcjson1 libdcmtk17t64 libpostproc57 libswresample4 libswscale7
11 upgraded, 0 newly installed, 0 to remove and 16 not upgraded.
11 esm-apps security updates
Need to get 19.3 MB of archives.
After this operation, 9216 B of additional disk space will be used.
```

### Make `dmesg` non-privileged

Since Ubuntu 22.04, `dmesg` has become a privileged operation
by default:

``` console
$ dmesg
dmesg: read kernel buffer failed: Operation not permitted
```

This is controlled by 

``` console
# sysctl kernel.dmesg_restrict
kernel.dmesg_restrict = 1
```

To revert this default, and make it permanent
([source](https://archived.forum.manjaro.org/t/why-did-dmesg-become-a-priveleged-operation-suddenly/86468/3)):

``` console
# echo 'kernel.dmesg_restrict=0' | tee -a /etc/sysctl.d/99-sysctl.conf
```

### Weekly btrfs scrub

To keep BTRFS file systems healthy, it is recommended to
[run a weekly scrub](http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html)
to check everything for consistency. For this, I run
[the script](https://marc.merlins.org/linux/scripts/btrfs-scrub)
from crontab every Saturday night.

``` console
# wget -O /usr/local/bin/btrfs-scrub-all \
  http://marc.merlins.org/linux/scripts/btrfs-scrub

# crontab -e
...
# m h  dom mon dow   command
50 2 * * 6 /usr/local/bin/btrfs-scrub-all
```

[Marc MERLIN](https://marc.merlins.org/) keeps the script updated,
so each systme may benefit from a few modifications, e.g.
1. Remove tests for laptop battery status, when running on a PC.
2. Set the `BTRFS_SCRUB_SKIP` to filter out partitions to skip.

``` bash linenums="1" title="/usr/local/bin/btrfs-scrub-all"
#! /bin/bash

# By Marc MERLIN <marc_soft@merlins.org> 2014/03/20
# License: Apache-2.0
# http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html

which btrfs >/dev/null || exit 0
export PATH=/usr/local/bin:/sbin:$PATH

FILTER='(^Dumping|balancing, usage)'
BTRFS_SCRUB_SKIP="sda"
source /etc/btrfs_config 2>/dev/null
test -n "$DEVS" || DEVS=$(grep '\<btrfs\>' /proc/mounts | awk '{ print $1 }' | sort -u | grep -v $BTRFS_SCRUB_SKIP)
for btrfs in $DEVS
do
    tail -n 0 -f /var/log/syslog | grep "BTRFS" | grep -Ev '(disk space caching is enabled|unlinked .* orphans|turning on discard|device label .* devid .* transid|enabling SSD mode|BTRFS: has skinny extents|BTRFS: device label|BTRFS info )' &
    mountpoint="$(grep "$btrfs" /proc/mounts | awk '{ print $2 }' | sort | head -1)"
    logger -s "Quick Metadata and Data Balance of $mountpoint ($btrfs)" >&2
    # Even in 4.3 kernels, you can still get in places where balance
    # won't work (no place left, until you run a -m0 one first)
    # I'm told that proactively rebalancing metadata may not be a good idea.
    #btrfs balance start -musage=20 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    # but a null rebalance should help corner cases:
    sleep 10
    btrfs balance start -musage=0 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    # After metadata, let's do data:
    sleep 10
    btrfs balance start -dusage=0 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    sleep 10
    btrfs balance start -dusage=20 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
    # And now we do scrub. Note that scrub can fail with "no space left
    # on device" if you're very out of balance.
    logger -s "Starting scrub of $mountpoint" >&2
    echo btrfs scrub start -Bd $mountpoint
    # -r is read only, but won't fix a redundant array.
    #ionice -c 3 nice -10 btrfs scrub start -Bdr $mountpoint
    time ionice -c 3 nice -10 btrfs scrub start -Bd $mountpoint
    pkill -f 'tail -n 0 -f /var/log/syslog'
    logger "Ended scrub of $mountpoint" >&2
done
```

!!! note

    Setting `BTRFS_SCRUB_SKIP="sda"` prevents Btrfs balancing from running
    on USB external storage if attached; not really necessary, but the
    script fails if this variable is left empty.

``` console
# /usr/local/bin/btrfs-scrub-all
<13>Nov 17 22:23:49 root: Quick Metadata and Data Balance of /home (/dev/nvme0n1p4)
Done, had to relocate 0 out of 1536 chunks
Done, had to relocate 0 out of 1536 chunks
Done, had to relocate 0 out of 1536 chunks
<13>Nov 17 22:24:19 root: Starting scrub of /home
btrfs scrub start -Bd /home
Starting scrub on devid 1

Scrub device /dev/nvme0n1p4 (id 1) done
Scrub started:    Sun Nov 17 22:24:19 2024
Status:           finished
Duration:         0:06:50
Total to scrub:   1.48TiB
Rate:             3.71GiB/s
Error summary:    no errors found

real    6m49.944s
user    0m0.002s
sys     4m41.299s
```

The whole process takes about 7 minutes with the 4TB NVMe SSD about
43% full, with 1.5T used.

![Resources used on a 7-minute run of btrfs scrub](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-last-15-min-with-btrfs-scrub.png)

!!! note

    The weekly Btrfs scrub doesn't seem to really need `inn` or any
    of its dependencies, so this installation step was **skipped**:

    ``` console
    # apt install inn -y
    ```

### Make SDDM Look Good

Ubuntu Studio 24.04 uses 
[Simple Desktop Display Manager (SDDM)](https://wiki.archlinux.org/title/SDDM)
([sddm/sddm](https://github.com/sddm/sddm) in GitHub)
which is quite good looking out of the box, but I like to
customize this for each computer.

For most computers my favorite SDDM theme is
[Breeze-Noir-Dark](https://store.kde.org/p/1361460),
which I like to install system-wide.

``` console
# unzip -d /usr/share/sddm/themes Breeze-Noir-Dark.zip
```

!!! warning

    Action icons won’t render if the directory name is changed. If needed,
    change the directory name in the `iconSource` fields in `Main.qml` to
    match final directory name so icons show. This is not the only thing
    that breaks when changing the directory name.

Other than installing this theme, all I really change in it
is the background image to use 
[Super Tuna (1920x1080)](https://www.nme.com/news/music/bts-jin-surprises-fans-super-tuna-birthday-celebration-3111626).

![Homage to The Most Popular Tuna Ever](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/Super-Tuna.jpg)

``` console
# mv Super-Tuna.jpg \
  /usr/share/sddm/themes/Breeze-Noir-Dark/
# cd /usr/share/sddm/themes/Breeze-Noir-Dark/

# vi theme.conf
[General]
type=image
color=#132e43
background=/usr/share/sddm/themes/Breeze-Noir-Dark/Super-Tuna.jpg

# vi theme.conf.user
[General]
type=image
background=Super-Tuna.jpg
```

**Additionally**, as this is new in Ubuntu 24.04, the theme has to
be selected by adding a `[Theme]` section in the system config
in `/usr/lib/sddm/sddm.conf.d/ubuntustudio.conf`

```ini
[General]
InputMethod=

[Theme]
Current="Breeze-Noir-Dark"
EnableAvatars=True
```

[Reportedly](https://superuser.com/questions/1720931/how-to-configure-sddm-in-kubuntu-22-04-where-is-config-file),
you have to **create** the `/etc/sddm.conf.d` directory to add
the *Local configuration* file that allows setting the theme:

``` console
# mkdir /etc/sddm.conf.d
# vi /etc/sddm.conf.d/ubuntustudio.conf
```

Besides setting the theme, it is also good to limit the range of
user ids so that only human users show up:

```ini
[Theme]
Current=Breeze-Noir-Dark

[Users]
MaximumUid=1001
MinimumUid=1000
```

### Bluetooth controller

The following shows up in `dmesg`:

``` console
Bluetooth: Core ver 2.22
NET: Registered PF_BLUETOOTH protocol family
Bluetooth: HCI device and connection manager initialized
Bluetooth: HCI socket layer initialized
Bluetooth: L2CAP socket layer initialized
Bluetooth: SCO socket layer initialized
Bluetooth: hci0: Device revision is 0
Bluetooth: hci0: Secure boot is enabled
Bluetooth: hci0: OTP lock is enabled
Bluetooth: hci0: API lock is enabled
Bluetooth: hci0: Debug lock is disabled
Bluetooth: hci0: Minimum firmware build 1 week 10 2014
Bluetooth: hci0: Bootloader timestamp 2019.40 buildtype 1 build 38
Bluetooth: hci0: DSM reset method type: 0x00
Bluetooth: hci0: Found device firmware: intel/ibt-0040-0041.sfi
Bluetooth: hci0: Boot Address: 0x100800
Bluetooth: hci0: Firmware Version: 60-48.23
Bluetooth: BNEP (Ethernet Emulation) ver 1.3
Bluetooth: BNEP filters: protocol multicast
Bluetooth: BNEP socket layer initialized
Bluetooth: hci0: Waiting for firmware download to complete
Bluetooth: hci0: Firmware loaded in 1536493 usecs
Bluetooth: hci0: Waiting for device to boot
Bluetooth: hci0: Device booted in 15625 usecs
Bluetooth: hci0: Malformed MSFT vendor event: 0x02
Bluetooth: hci0: Found Intel DDC parameters: intel/ibt-0040-0041.ddc
Bluetooth: hci0: Applying Intel DDC parameters completed
Bluetooth: hci0: Firmware timestamp 2023.48 buildtype 1 build 75324
Bluetooth: hci0: Firmware SHA1: 0x23bac558
Bluetooth: MGMT ver 1.22
```

#### Bluetooth headphones

Pairing headphones (Bose QuietComfort SE) was quick and easy. As a fun
bonus, every time the PC connects to the headphones an audible notification
announced *connected to super tuna* which is fun.

## Kernel memory leak

Only in this Nuc PC is Ubuntu Studio 24.04 showing a huge memory leak
and it seems to be in the kernel rather than in any application/s:

``` console
# free -h
               total        used        free      shared  buff/cache   available
Mem:            30Gi        25Gi       1.6Gi       190Mi       4.2Gi       5.0Gi
Swap:          8.0Gi       256Ki       8.0Gi
```

This could have been caused by transferring 1.5 TB of data over the
network, but normally that would increase only the buff/cache usage
and that is not the case here. It is the *actually* `used` memory
that has been growing steadily over hours and is only released upon
rebooting:

![Memory usage in the last 9 hours](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-last-9-h.png)

??? note "Red herring: `fluidsynth`"

    Killing the most memory intensive process had previously made no dent:

    ``` console
    # ps -ax -eo user,pid,pcpu:5,pmem:5,rss:8=R-MEM,vsz:10=V-MEM,tty:6=TTY,stat:5,bsdstart:7,bsdtime:7,args | (read h; echo "$h"; sort -nr -k 4) | head -10 | less -SEX
    USER         PID  %CPU  %MEM    R-MEM      V-MEM TTY    STAT    START    TIME COMMAND
    root     2661652   0.5   0.7   246360    1568376 ?      SLsl    07:57    0:13 /usr/bin/fluidsynth -is /usr/share/sounds/sf3/default-GM.sf3 HOME=/ro>
    sddm     2405392   0.2   0.4   158060    1140264 ?      Sl      00:48    1:03 /usr/bin/sddm-greeter --socket /tmp/sddm-:0-KPzxxr --theme /usr/share>
    colord     12644   0.0   0.3   120748     425728 ?      Ssl     00:40    0:00 /usr/libexec/colord LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/local/>
    root     2381522   0.0   0.2    83072     763252 tty2   Ssl+    00:47    0:14 /usr/lib/xorg/Xorg -nolisten tcp -background none -seat seat0 vt2 -au>
    root         481   0.0   0.2    75596     137064 ?      S<s     00:40    0:19 /usr/lib/systemd/systemd-journald LANG=es_ES.UTF-8 PATH=/usr/local/sb>
    root       14132   0.0   0.1    40788     477384 ?      Ssl     00:54    0:01 /usr/libexec/fwupd/fwupd LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/l>
    root        1680   0.0   0.1    33252    2731468 ?      Ssl     00:40    0:02 /usr/lib/snapd/snapd LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/local>
    minidlna    1645   0.0   0.1    36224     336280 ?      SLsl    00:40    0:00 /usr/sbin/minidlnad -f /etc/minidlna.conf -P /run/minidlna/minidlna.p>
    debian-+   30046   0.3   0.1    61244    1252000 ?      Ssl     00:40    1:26 /usr/bin/tor --defaults-torrc /usr/share/tor/tor-service-defaults-tor>

    # killall /usr/bin/fluidsynth
    # ps -ax -eo user,pid,pcpu:5,pmem:5,rss:8=R-MEM,vsz:10=V-MEM,tty:6=TTY,stat:5,bsdstart:7,bsdtime:7,args | (read h; echo "$h"; sort -nr -k 4) | head -10 | less -SEX
    USER         PID  %CPU  %MEM    R-MEM      V-MEM TTY    STAT    START    TIME COMMAND
    sddm     2405392   0.2   0.4   158060    1140328 ?      Sl      00:48    1:04 /usr/bin/sddm-greeter --socket /tmp/sddm-:0-KPzxxr --theme /usr/share>
    colord     12644   0.0   0.3   120748     425728 ?      Ssl     00:40    0:00 /usr/libexec/colord LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/local/>
    root     2381522   0.0   0.2    83072     763252 tty2   Ssl+    00:47    0:14 /usr/lib/xorg/Xorg -nolisten tcp -background none -seat seat0 vt2 -au>
    root         481   0.0   0.2    76236     137064 ?      S<s     00:40    0:19 /usr/lib/systemd/systemd-journald LANG=es_ES.UTF-8 PATH=/usr/local/sb>
    root       14132   0.0   0.1    40788     477384 ?      Ssl     00:54    0:01 /usr/libexec/fwupd/fwupd LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/l>
    root        1680   0.0   0.1    33252    2731468 ?      Ssl     00:40    0:02 /usr/lib/snapd/snapd LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/local>
    minidlna    1645   0.0   0.1    36224     336280 ?      SLsl    00:40    0:00 /usr/sbin/minidlnad -f /etc/minidlna.conf -P /run/minidlna/minidlna.p>
    debian-+   30046   0.3   0.1    61244    1252000 ?      Ssl     00:40    1:27 /usr/bin/tor --defaults-torrc /usr/share/tor/tor-service-defaults-tor>
    vnstat      5498   0.0   0.0     3584       5476 ?      Ss      00:40    0:03 /usr/sbin/vnstatd -n LANG=es_ES.UTF-8 PATH=/usr/local/sbin:/usr/local>

    # free -h
                  total        used        free      shared  buff/cache   available
    Mem:            30Gi        27Gi       3.6Gi       188Mi       1.0Gi       3.8Gi
    Swap:          8.0Gi       256Ki       8.0Gi
    ```

    That `fluidsynth` process may have been a leftover from having started `ardour`
    earlier, but this was not really a problematic case of
    [Fluidsynth Starting Automatically On Boot; Login Freezing](https://askubuntu.com/questions/1446116/fluidsynth-starting-automatically-on-boot-login-freezing-connected-how-to-so).

??? note "Red herring: `drop_caches`"

    The trick in [linuxatemyram.com](https://www.linuxatemyram.com/)
    to drop caches did not help:

    ``` console
    # echo 3 | sudo tee /proc/sys/vm/drop_caches
    3

    # free -h
                  total        used        free      shared  buff/cache   available
    Mem:            30Gi        26Gi       4.5Gi       188Mi       781Mi       4.6Gi
    Swap:          8.0Gi       256Ki       8.0Gi

    # free -m
                  total        used        free      shared  buff/cache   available
    Mem:           31645       27081        4200         186        1013        4563
    Swap:           8191           0        8191
    ```

    Indeed this only releases the `buff/cache` memory, not that which is `used`:

    ![Memory usage in the last 15 minutes](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-last-15-min.png)

    There was another hint somewhere (Red Hat Oracle documentation)
    about the number of "huge pages" but that is already set to zero:

    ``` console
    # cat /proc/sys/vm/nr_hugepages 
    0
    ```

### Suspect driver: `i915`

After a wider and deeper investigation
(see ~1500 lines in [this log](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/2024-11-16-14-50.txt))
it looks like a memory leak bug in the `i915` driver:

``` console
# apt install smem -y

# smem -twk
Area                           Used      Cache   Noncache 
firmware/hardware                 0          0          0 
kernel image                      0          0          0 
kernel dynamic memory         18.7G       3.1G      15.5G 
userspace memory               2.4G       1.2G       1.3G 
free memory                    9.8G       9.8G          0 
----------------------------------------------------------
                              30.9G      14.1G      16.8G 

# smem -wp
Area                           Used      Cache   Noncache 
firmware/hardware             0.00%      0.00%      0.00% 
kernel image                  0.00%      0.00%      0.00% 
kernel dynamic memory        58.94%     10.13%     48.81% 
userspace memory              7.24%      3.05%      4.19% 
free memory                  33.81%     33.81%      0.00% 

# lsmod | sort -nr -k 2 | head
i915                 4284416  69
xe                   2711552  0
btrfs                2015232  1
mac80211             1720320  1 iwlmvm
kvm                  1409024  1 kvm_intel
cfg80211             1323008  3 iwlmvm,iwlwifi,mac80211
bluetooth            1028096  71 btrtl,btmtk,btintel,btbcm,bnep,btusb,rfcomm
iwlmvm                868352  0
iwlwifi               598016  1 iwlmvm
thunderbolt           516096  0

# uname -a
Linux super-tuna 6.8.0-47-lowlatency #47.1-Ubuntu SMP PREEMPT_DYNAMIC Mon Oct  7 15:01:17 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux

# lspci -nnk
00:00.0 Host bridge [0600]: Intel Corporation Raptor Lake-P/U 4p+8e cores Host Bridge/DRAM Controller [8086:a707]
        DeviceName: Onboard - Other
        Subsystem: Intel Corporation Raptor Lake-P/U 4p+8e cores Host Bridge/DRAM Controller [8086:3037]
        Kernel driver in use: igen6_edac
        Kernel modules: igen6_edac
00:02.0 VGA compatible controller [0300]: Intel Corporation Raptor Lake-P [UHD Graphics] [8086:a720] (rev 04)
        DeviceName: Onboard - Video
        Subsystem: Intel Corporation Raptor Lake-P [UHD Graphics] [8086:3037]
        Kernel driver in use: i915
        Kernel modules: i915, xe
```

Since the `i915` and `xe` drivers are taking more memory than
all others, it looked like this could be a case of memory leak
in the driver (e.g. like
[strongtz/i915-sriov-dkms issue #137](https://github.com/strongtz/i915-sriov-dkms/issues/137)).
However, this system is using the "vanilla" `i915` drivers and,
after the memory has been released by rebooting the system,
the drivers still show about the same size:

``` console
# smem -twk
Area                           Used      Cache   Noncache 
firmware/hardware                 0          0          0 
kernel image                      0          0          0 
kernel dynamic memory          3.6G       2.1G       1.5G 
userspace memory               1.6G     499.5M       1.1G 
free memory                   25.7G      25.7G          0 
----------------------------------------------------------
                              30.9G      28.3G       2.6G 

# smem -wp
Area                           Used      Cache   Noncache 
firmware/hardware             0.00%      0.00%      0.00% 
kernel image                  0.00%      0.00%      0.00% 
kernel dynamic memory        11.84%      6.79%      5.05% 
userspace memory              5.08%      1.58%      3.50% 
free memory                  83.07%     83.07%      0.00%

# lsmod | sort -nr -k 2 | head
i915                 4284416  60
xe                   2711552  0
btrfs                2015232  1
mac80211             1720320  1 iwlmvm
kvm                  1409024  1 kvm_intel
cfg80211             1323008  3 iwlmvm,iwlwifi,mac80211
bluetooth            1028096  44 btrtl,btmtk,btintel,btbcm,bnep,btusb,rfcomm
iwlmvm                868352  0
iwlwifi               598016  1 iwlmvm
thunderbolt           516096  0
```

The fix for
[strongtz/i915-sriov-dkms issue #137](https://github.com/strongtz/i915-sriov-dkms/issues/137)
seems to be
[strongtz/i915-sriov-dkms PR #204](https://github.com/strongtz/i915-sriov-dkms/pull/204/commits/de9f8627a82a13b4b0c7e283eac539cd6c40f409)
but that looks like a non-trivial change.

There is also a one-line patch to
[Fix memory leak by correcting cache object name in error handler](https://patchwork.kernel.org/project/intel-gfx/patch/BYAPR03MB4168C6D020B750EAF8021731ADE22@BYAPR03MB4168.namprd03.prod.outlook.com/)
from May 2024 and is not yet applied to the source file
`drivers/gpu/drm/i915/i915_scheduler.c` in the
`linux-source-6.8.0` package. This one seems a more likely fix,
and a much simpler change to implement.

The patch apperas to have been
[resent in late November, 2024](https://lore.kernel.org/lkml/Z1DNlAPvPNtgpMXO@ashyti-mobl2.lan/T/),
[committed in early December](https://gbmc.googlesource.com/linux/+/2828e5808bcd5aae7fdcd169cac1efa2701fa2dd),
and that may be related to the patch not being (yet) available in the
6.8.0-47 and 6.8.0-49 kernels. The patch may be included in later kernels,
but after an update in early January, 2025 the kernel updated to 6.8.0-51
and yet the linux-source-6.8.0 package on version 6.8.0-**51**.52 still
does not have the patched applied.

!!! todo

    [BuildYourOwnKernel](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel)
    with the patched applied, see if that helps.

### Maybe somewhere else?

If the `i915` is not behind this memory leak, the one other lead from
[this log](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/2024-11-16-14-50.txt) is, by far, the largest
user of kernel memory is `kmalloc-rnd-07-8k`:

``` console
# slabtop -o | head -10
 Active / Total Objects (% used)    : 3424282 / 3461230 (98,9%)
 Active / Total Slabs (% used)      : 507294 / 507294 (100,0%)
 Active / Total Caches (% used)     : 341 / 386 (88,3%)
 Active / Total Size (% used)       : 15543458,94K / 15549019,20K (100,0%)
 Minimum / Average / Maximum Object : 0,01K / 4,49K / 10,25K

  OBJS ACTIVE USE OBJ SIZE SLABS OBJ/SLAB CACHE SIZE NAME                     
1901280 1901280 100%    8,00K 475320        4  15210240K kmalloc-rnd-07-8k
 170310  170106  99%    0,19K   4055       42     32440K dentry
 162825  162299  99%    0,10K   4175       39     16700K buffer_head
```

That's a whooping **14.5 GB** used only by `kmalloc-rnd-07-8k`, which is
released only after a reboot (although soon enough back up to nearly 2 GB):

``` console
# slabtop -o | head -10
 Active / Total Objects (% used)    : 1413878 / 1435036 (98.5%)
 Active / Total Slabs (% used)      : 87550 / 87550 (100.0%)
 Active / Total Caches (% used)     : 338 / 386 (87.6%)
 Active / Total Size (% used)       : 2300293.48K / 2306032.96K (99.8%)
 Minimum / Average / Maximum Object : 0.01K / 1.61K / 10.25K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
256376 256376 100%    8.00K  64094        4   2051008K kmalloc-rnd-12-8k      
114996 114394  99%    0.19K   2738       42     21904K dentry                 
 73100  72807  99%    0.05K    860       85      3440K shared_policy_node     
```

There is nothing like this on `rapture`, where `kmalloc-rnd-07-8k` is only 512K:

``` console
root@rapture:~# slabtop -o | head -10
 Active / Total Objects (% used)    : 7453076 / 7724162 (96.5%)
 Active / Total Slabs (% used)      : 178099 / 178099 (100.0%)
 Active / Total Caches (% used)     : 338 / 385 (87.8%)
 Active / Total Size (% used)       : 2520903.59K / 2613929.42K (96.4%)
 Minimum / Average / Maximum Object : 0.01K / 0.34K / 32.54K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
1435392 1434726  99%    0.19K  34176       42    273408K dentry                 
1334925 1332417  99%    0.05K  15705       85     62820K shared_policy_node     
1063179 1063179 100%    1.06K  35928       30   1149696K btrfs_inode            

# slabtop -o | grep kmalloc-rnd-12-8k
      64     64 100%    8.00K     16        4       512K kmalloc-rnd-12-8k 
```

To compare again, the biggest user of kernel dynamic memory is

*   `btrfs_inode` in `rapture`, with 1,149,792K after a good 20 hours up
*   `kmalloc-rnd-12-8k` in `super-tuna`, with 2,391,360K after barely 1 hour up

**5 hours later** `kmalloc-rnd-12-8k` in `super-tuna` is up **13.71 GB**
again, which amounts for most of the `SUnreclaim` memory.

??? terminal "`kmalloc-rnd-12-8k` up 13.7 GB, `SUnreclaim` up 15.3 GB."

    ``` console
    # smem -twk
    Area                           Used      Cache   Noncache 
    firmware/hardware                 0          0          0 
    kernel image                      0          0          0 
    kernel dynamic memory         16.4G       2.2G      14.2G 
    userspace memory               1.6G     534.1M       1.1G 
    free memory                   12.9G      12.9G          0 
    ----------------------------------------------------------
                                  30.9G      15.6G      15.3G 
    # smem -wp
    Area                           Used      Cache   Noncache 
    firmware/hardware             0.00%      0.00%      0.00% 
    kernel image                  0.00%      0.00%      0.00% 
    kernel dynamic memory        53.19%      7.08%     46.11% 
    userspace memory              5.24%      1.69%      3.55% 
    free memory                  41.57%     41.57%      0.00% 

    # lsmod | sort -nr -k 2 | head -3
    i915                 4284416  59
    xe                   2711552  0
    btrfs                2015232  1

    # slabtop -o | sort -r -n -k 7 | head -10
    1797400 1797400 100%    8.00K 449350        4  14379200K kmalloc-rnd-12-8k      
    22302  22302 100%    1.16K    826       27     26432K ext4_inode_cache       
    117432 116955  99%    0.19K   2796       42     22368K dentry                 
    22900  22835  99%    0.62K    916       25     14656K inode_cache            
    20944  20925  99%    0.57K    748       28     11968K radix_tree_node        
    72324  69525  96%    0.14K   2583       28     10332K kernfs_node_cache      
      954    946  99%   10.25K    318        3     10176K task_struct            
    13754  13041  94%    0.70K    299       46      9568K proc_inode_cache       
    48510  47450  97%    0.19K   1155       42      9240K cred_jar               
    85137  85137 100%    0.10K   2183       39      8732K buffer_head     

    # cat /proc/slabinfo | grep "kmalloc-rnd-12-8k"
    kmalloc-rnd-12-8k 1962444 1962444   8192    4    8 : tunables    0    0    0 : slabdata 490611 490611      0

    # head -3 /proc/slabinfo; cat /proc/slabinfo | grep "kmalloc-rnd-12-8k"
    slabinfo - version: 2.1
    # name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
    QIPCRTR               78     78    832   39    8 : tunables    0    0    0 : slabdata      2      2      0
    kmalloc-rnd-12-8k 1963068 1963068   8192    4    8 : tunables    0    0    0 : slabdata 490767 490767      0

    # free -h
                  total        used        free      shared  buff/cache   available
    Mem:            30Gi        17Gi        11Gi       748Mi       2.7Gi        13Gi
    Swap:          8.0Gi          0B       8.0Gi

    # cat /proc/meminfo 
    MemTotal:       32404656 kB
    MemFree:        12123032 kB
    MemAvailable:   13785908 kB
    Buffers:          137656 kB
    Cached:          2610480 kB
    SwapCached:            0 kB
    Active:          1825832 kB
    Inactive:        1126328 kB
    Active(anon):     970752 kB
    Inactive(anon):        0 kB
    Active(file):     855080 kB
    Inactive(file):  1126328 kB
    Unevictable:      940008 kB
    Mlocked:          212356 kB
    SwapTotal:       8388604 kB
    SwapFree:        8388604 kB
    Zswap:                 0 kB
    Zswapped:              0 kB
    Dirty:               256 kB
    Writeback:             0 kB
    AnonPages:       1143976 kB
    Mapped:           548812 kB
    Shmem:            766728 kB
    KReclaimable:     101908 kB
    Slab:           16161044 kB
    SReclaimable:     101908 kB
    SUnreclaim:     16059136 kB
    KernelStack:       13280 kB
    PageTables:        27176 kB
    ...
    ```

Google web search shows a single search result for `"kmalloc-rnd-12-8k"`
[which does not lead to a conclusive solution](https://discussion.fedoraproject.org/t/kernel-memory-leak-fedora-40-nvidia-drivers-ollama/135038/29).

### Maybe gone after reinstalling?

Out of frustation with this opaque memory leak and a few other details,
decided to reinstall Ubuntu Studio 24.04 anew.

Along the way had to
[update the BIOS](https://www.intel.com/content/www/us/en/support/articles/000005636/intel-nuc.html)
(to version 33) because until that update the systen would not boot
from the USB start-up disk, even though it had booted from it only minutes ago.

Eventually, most to fhe above setup was restored and yet the memory leak
was no longer happening:

![Memory usage in a 2-hour period with memory leak](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-2-h-with-memory-leak.png)

Moreover, it seems the CPU cores are running visibly cooler now:

![Memory usage in a 2-hour period withou memory leak](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-2-h-without-memory-leak.png)

#### But Why?

A few things were different when reinstalling the system.

First, the system language was left as its default value: English.
This *should* be irrelevant, but when the system locazation affects
everything down to the output from commands, it might just happen that
different number format and/or translated messages may trigger bugs.

### **Not** gone after reinstalling

The (or *a*) memory leak is still there, even if only it grows slowly enough
that it takes about 3 days to reach the point where almost all 32 GB of RAM
are taken up by `SUnreclaim` and the biggest memory hoarder is, by far,
`kmalloc-rnd-11-8k`:

```console
# cat /proc/meminfo
...
Slab:           31814260 kB
SReclaimable:      50868 kB
SUnreclaim:     31763392 kB
...

# slabtop -o | head -10
 Active / Total Objects (% used)    : 4817156 / 5002245 (96.3%)
 Active / Total Slabs (% used)      : 1032290 / 1032290 (100.0%)
 Active / Total Caches (% used)     : 342 / 386 (88.6%)
 Active / Total Size (% used)       : 31637718.69K / 31682510.83K (99.9%)
 Minimum / Average / Maximum Object : 0.01K / 6.33K / 10.25K

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
3931151 3931151 100%    8.00K 1010822        4  32346304K kmalloc-rnd-11-8k      
 71145  36213  50%    0.05K    837       85      3348K shared_policy_node     
 67928  67790  99%    0.14K   2426       28      9704K kernfs_node_cache      

# cat /proc/slabinfo | grep "kmalloc-rnd-11-8k"
kmalloc-rnd-11-8k 3931772 3931772   8192    4    8 : tunables    0    0    0 : slabdata 1011434 1011434      0

# head -3 /proc/slabinfo; cat /proc/slabinfo | grep kmalloc-rnd-11-8k
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
QIPCRTR               78     78    832   39    8 : tunables    0    0    0 : slabdata      2      2      0
kmalloc-rnd-11-8k 3931887 3931887   8192    4    8 : tunables    0    0    0 : slabdata 1011528 1011528      0
```

??? terminal "Repeat run or debug commands above shows about the same."

    ```console
    # free -h
                  total        used        free      shared  buff/cache   available
    Mem:            30Gi        30Gi       291Mi        61Mi       152Mi        73Mi
    Swap:          8.0Gi       271Mi       7.7Gi

    # ps -ax -eo user,pid,pcpu:5,pmem:5,rss:8=R-MEM,vsz:10=V-MEM,tty:6=TTY,stat:5,bsdstart:7,bsdtime:7,args | (read h; echo "$h"; sort -nr -k 4) | head -10 | less -SEX
    USER         PID  %CPU  %MEM    R-MEM      V-MEM TTY    STAT    START    TIME COMMAND
    vnstat      2707   0.0   0.0     1536       5516 ?      Ss     Jan  5    0:17 /usr/sbin/vnstatd -n LANG=en_US.UTF-8 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin USER=vnstat LOGNAME=vnstat HOME=/var/lib/vnstat INVOCATION_ID=ac8c9f302e5444b691575435a6157b99 JOURNAL_STREAM=8:29855 STATE_DIRECTORY=/var/lib/vnstat SYSTEMD_EXEC_PID=2707 MEMORY_PRESSURE_WATCH=/sy>
    systemd+    1201   0.0   0.0     1152      91044 ?      Ssl    Jan  5    0:01 /usr/lib/systemd/systemd-timesyncd LANG=en_US.UTF-8 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin NOTIFY_SOCKET=/run/systemd/notify WATCHDOG_PID=1201 WATCHDOG_USEC=180000000 USER=systemd-timesync LOGNAME=systemd-timesync INVOCATION_ID=9f337859e19249b2977af8bb586f2c94 JOURNAL_STREA>
    systemd+    1196   0.0   0.0      384      22008 ?      Ss     Jan  5    0:48 /usr/lib/systemd/systemd-resolved LANG=en_US.UTF-8 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin NOTIFY_SOCKET=/run/systemd/notify WATCHDOG_PID=1196 WATCHDOG_USEC=180000000 USER=systemd-resolve LOGNAME=systemd-resolve INVOCATION_ID=5707dd899b2541a289e315ed6616d70c JOURNAL_STREAM=8>
    syslog      1787   0.0   0.0      640     222508 ?      Ssl    Jan  5    0:02 /usr/sbin/rsyslogd -n -iNONE LANG=en_US.UTF-8 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin NOTIFY_SOCKET=/run/systemd/notify LISTEN_PID=1787 LISTEN_FDS=1 LISTEN_FDNAMES=syslog.socket USER=root INVOCATION_ID=94eeaa8f349e456fa5df905f17729693 JOURNAL_STREAM=8:29727 SYSTEMD_EXEC_PID=>
    rtkit       3291   0.0   0.0     1024      22940 ?      SNsl   Jan  5    0:04 /usr/libexec/rtkit-daemon LANG=en_US.UTF-8 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin NOTIFY_SOCKET=/run/systemd/notify USER=root INVOCATION_ID=6e9a99b6e2ab44019c666b2ae4b886cc JOURNAL_STREAM=8:21269 SYSTEMD_EXEC_PID=3291 MEMORY_PRESSURE_WATCH=/sys/fs/cgroup/system.slice/rtkit->
    root          99   0.2   0.0        0          0 ?      S      Jan  5   11:40 [ksoftirqd/15]
    root         984   0.0   0.0        0          0 ?      I<     Jan  5    0:00 [kworker/R-btrfs]
    root         983   0.0   0.0        0          0 ?      I<     Jan  5    0:00 [kworker/R-btrfs]
    root         982   0.0   0.0        0          0 ?      I<     Jan  5    0:00 [kworker/R-btrfs]

    # echo 3 | tee /proc/sys/vm/drop_caches
    3

    # free -h
                  total        used        free      shared  buff/cache   available
    Mem:            30Gi        30Gi       245Mi        87Mi       187Mi        32Mi
    Swap:          8.0Gi       257Mi       7.7Gi

    # free -m
                  total        used        free      shared  buff/cache   available
    Mem:           31645       31594         269          79         166          50
    Swap:           8191         265        7926

    # sync

    # cat /proc/sys/vm/nr_hugepages
    0

    # smem -twk
    Area                           Used      Cache   Noncache 
    firmware/hardware                 0          0          0 
    kernel image                      0          0          0 
    kernel dynamic memory         30.6G     155.9M      30.4G 
    userspace memory              39.4M      19.6M      19.8M 
    free memory                  278.6M     278.6M          0 
    ----------------------------------------------------------
                                  30.9G     454.1M      30.5G 

    # smem -wp
    Area                           Used      Cache   Noncache 
    firmware/hardware             0.00%      0.00%      0.00% 
    kernel image                  0.00%      0.00%      0.00% 
    kernel dynamic memory        99.00%      0.43%     98.57% 
    userspace memory              0.13%      0.06%      0.07% 
    free memory                   0.86%      0.86%      0.00% 

    # lsmod | sort -nr -k 2 | head
    i915                 4292608  27
    xe                   2715648  0
    btrfs                2019328  1
    mac80211             1720320  1 iwlmvm
    kvm                  1409024  1 kvm_intel
    cfg80211             1323008  3 iwlmvm,iwlwifi,mac80211
    bluetooth            1028096  34 btrtl,btmtk,btintel,btbcm,bnep,btusb,rfcomm
    iwlmvm                872448  0
    iwlwifi               598016  1 iwlmvm
    thunderbolt           516096  0

    # uname -a
    Linux super-tuna 6.8.0-49-lowlatency #49.1-Ubuntu SMP PREEMPT_DYNAMIC Sun Nov 10 10:00:36 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux

    # slabtop -o | head -10
    Active / Total Objects (% used)    : 4817156 / 5002245 (96.3%)
    Active / Total Slabs (% used)      : 1032290 / 1032290 (100.0%)
    Active / Total Caches (% used)     : 342 / 386 (88.6%)
    Active / Total Size (% used)       : 31637718.69K / 31682510.83K (99.9%)
    Minimum / Average / Maximum Object : 0.01K / 6.33K / 10.25K

      OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
    3931151 3931151 100%    8.00K 1010822        4  32346304K kmalloc-rnd-11-8k      
    71145  36213  50%    0.05K    837       85      3348K shared_policy_node     
    67928  67790  99%    0.14K   2426       28      9704K kernfs_node_cache      

    # cat /proc/slabinfo | grep "kmalloc-rnd-11-8k"
    kmalloc-rnd-11-8k 3931772 3931772   8192    4    8 : tunables    0    0    0 : slabdata 1011434 1011434      0

    # head -3 /proc/slabinfo; cat /proc/slabinfo | grep kmalloc-rnd-11-8k
    slabinfo - version: 2.1
    # name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
    QIPCRTR               78     78    832   39    8 : tunables    0    0    0 : slabdata      2      2      0
    kmalloc-rnd-11-8k 3931887 3931887   8192    4    8 : tunables    0    0    0 : slabdata 1011528 1011528      0

    # cat /proc/meminfo
    MemTotal:       32404500 kB
    MemFree:          283740 kB
    MemAvailable:      60448 kB
    Buffers:            2388 kB
    Cached:           102356 kB
    SwapCached:         9336 kB
    Active:            21592 kB
    Inactive:          33280 kB
    Active(anon):       7320 kB
    Inactive(anon):     6100 kB
    Active(file):      14272 kB
    Inactive(file):    27180 kB
    Unevictable:       62500 kB
    Mlocked:              16 kB
    SwapTotal:       8388604 kB
    SwapFree:        8105176 kB
    Zswap:                 0 kB
    Zswapped:              0 kB
    Dirty:               552 kB
    Writeback:             0 kB
    AnonPages:         10912 kB
    Mapped:            18372 kB
    Shmem:             63280 kB
    KReclaimable:      50868 kB
    Slab:           31814260 kB
    SReclaimable:      50868 kB
    SUnreclaim:     31763392 kB
    KernelStack:        9344 kB
    PageTables:        10260 kB
    SecPageTables:         0 kB
    NFS_Unstable:          0 kB
    Bounce:                0 kB
    WritebackTmp:          0 kB
    CommitLimit:    24590852 kB
    Committed_AS:    1989136 kB
    VmallocTotal:   34359738367 kB
    VmallocUsed:       50920 kB
    VmallocChunk:          0 kB
    Percpu:            21888 kB
    HardwareCorrupted:     0 kB
    AnonHugePages:         0 kB
    ShmemHugePages:     8192 kB
    ShmemPmdMapped:        0 kB
    FileHugePages:         0 kB
    FilePmdMapped:         0 kB
    Unaccepted:            0 kB
    HugePages_Total:       0
    HugePages_Free:        0
    HugePages_Rsvd:        0
    HugePages_Surp:        0
    Hugepagesize:       2048 kB
    Hugetlb:               0 kB
    DirectMap4k:      315792 kB
    DirectMap2M:     8658944 kB
    DirectMap1G:    25165824 kB
    ```

??? quote "[Understanding Linux Kernel Memory Statistics](https://blogs.oracle.com/linux/post/understanding-linux-kernel-memory-statistics) explains these values:"

    #### Byte-level memory: VMalloc

    ##### Slab allocator

    The kernel uses various data structures in different kernel components and
    drivers. Some examples are `struct mm_struct` used for each process's address
    space and `struct buffer_head` used in filesystem I/O.
    The Slab allocator manages caches of commonly used objects kept in an
    initialized state available to be used later. This saves time for allocating,
    initialising, and freeing objects.

    The following statistics offer insights into the Slab allocator’s impact.

    *   `Slab` is the total memory used by all Slab caches.
    *   `SReclaimable is` the amount of the Slab that might be reclaimed 
        (such as cache objects like dentry).
    *   `SUnreclaim` is the opposite. When unreclaimable slab memory takes up a
        high percentage of the total memory, system performance may be affected.

    ```
    Slab:            1622084 kB
    SReclaimable:    1000692 kB
    SUnreclaim:       621392 kB
    ```

    Slab memory can grow when a kernel component or a driver requests memory
    but fails to release the memory properly, which is a typical memory leak
    case. Delving deeper into the specifics of `/proc/slabinfo` is crucial
    for identifying the particular slab object that has accumulated substantial
    memory usage.

    ...

    For byte-level memory, `kmalloc` and `vmalloc` are involved. `kmalloc`
    allocates physically contiguous memory regions, while `vmalloc` handles
    memory blocks that may not be available in contiguous memory regions.

### Debug memory leak

If applying the `i915` patch does not help, the next step would be to do a
[Kernel dynamic memory analysis](https://elinux.org/Kernel_dynamic_memory_analysis)
to find out what is leaking memory.

This should be relatively easy since `/sys/kernel/debug/tracing/trace`
is already present, although it only contains the headers. To enable
tracing of memory events, add this to kernel parameters:

``` ini
trace_event=kmem:kmalloc,kmem:kmem_cache_alloc,kmem:kfree,kmem:kmem_cache_free
```

This can be done by adding `/etc/default/grub.d/trace.cfg` with

```ini
# Enable tracing of memory events
GRUB_CMDLINE_LINUX_DEFAULT="$GRUB_CMDLINE_LINUX_DEFAULT trace_event=kmem:kmalloc,kmem:kmem_cache_alloc,kmem:kfree,kmem:kmem_cache_free"
```

After 2 days running with this, there are 745,033 lines in
`/sys/kernel/debug/tracing/trace` but it is not entirely clear what callsites
they point to:

``` console
# grep alloc kmem.log | grep --color=no -o 'call_site=[a-zA-Z_0-9]*' | sed 's/call_site=//' > kmem.txt
# sort kmem.txt | uniq -c | sort -nr | head
 138686 vm_area_dup
  92207 anon_vma_clone
  77047 getname_flags
  75840 alloc_buffer_head
  64657 vm_area_alloc
  58102 mas_alloc_nodes
  49660 __alloc_skb
  44111 security_file_alloc
  44111 alloc_empty_file
  42981 anon_vma_fork
  26055 __anon_vma_prepare
  19385 i915_gem_do_execbuffer
  10631 kvmalloc_node
   9379 seq_open
   8660 single_open
   7014 drm_syncobj_array_wait_timeout
   7014 drm_syncobj_array_find
   6537 jbd2__journal_start
   3071 __d_alloc
   2765 prepare_creds
   2763 security_prepare_creds
   2648 security_inode_alloc
   2527 kmalloc_reserve
   2518 load_elf_binary
   2517 load_elf_phdrs
   2430 alloc_pipe_info
   2361 krealloc
   2102 alloc_extent_state
   1883 genl_family_rcv_msg_attrs_parse
   1804 memcg_alloc_slab_cgroups
   1512 dup_task_struct
   1509 security_task_alloc
   1509 audit_alloc
   1507 mas_dup_build
   1507 dup_mm
   1507 copy_signal
   1507 copy_sighand
   1506 dup_fd
   1506 copy_fs_struct
   1494 alloc_pid
   1400 alloc_vmap_area
   1350 proc_alloc_inode
   1302 getname_kernel
   1264 xas_alloc
   1253 mm_alloc
   1253 alloc_bprm
   1215 alloc_inode
   1173 __i915_request_create
   1171 drm_syncobj_create
   1171 add_fence_array
    865 sg_kmalloc
    831 mempool_alloc_slab
    804 i915_vma_resource_alloc
    798 drm_atomic_state_init
    797 vma_create
```

It seems one would really need to
[wrestle against gcc](https://elinux.org/Kernel_dynamic_memory_analysis#Obtaining_accurate_call_sites_.28or_The_painstaking_task_of_wrestling_against_gcc.29)
to obtain accurate callsites. Moreover, the script to analyze this file
([`trace_analyze.py`](https://github.com/ezequielgarcia/kmem-probe-framework/blob/master/post-process/trace_analyze.py))
is from 2012 and writen in Python 2.

Alternative methods:

*   [Kernel Memory Leak Detector](https://www.kernel.org/doc/html/v4.10/dev-tools/kmemleak.html) would require enabling `CONFIG_DEBUG_KMEMLEAK`
    when compling a new kernel, because `/sys/kernel/debug/kmemleakls`
    is not present.
*   [euspectre/kedr](https://github.com/euspectre/kedr/wiki) seems more
    involved, based on its
    [step-by-step tutorial](https://github.com/euspectre/kedr/wiki/kedr_manual_getting_started).

### Disable swap

Swap was disabled by commenting out the corresponding line in `/etc/fstab`.

The memory leak was detected when the Chrome browser started crashing
consistently despite running only one tab on one web application, and
eventually even SSH sessions where crashing and at that point it seems
the availability of swap lead the system to go into thrashing, because
at that point essentially all the RAM was taken up and Chrome could no
longer allocate enough memory:

![Heavy I/O suggests thrashing happened for several minutes](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-ram-full-after-chrome-crash.png)

In the 82 hours the system had been running until it was no longer stable,
the memory leak had squeezed Chrome (and eventually everything else) out of
all available memory:

![Per-application memory usage shows chrome losing ability to allocate memory after 82 hours](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-user-ram-full-after-82-hours.png)
![System-wide memory usage shows all memory used and nothing available after 82 hours](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-ram-full-after-82-hours.png)

This could have been detected earlier, since the *used* RAM had already
grown to 14 GB in the first 24 hours:

![System-wide memory usage up to 14 GB in the first 24 hours](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-ram-up-14G-after-24-hours.png)

Such a memory leak did not afect
[Raven](./2024-12-27-ubuntu-studio-24-04-on-raven-gaming-pc-and-more.md)
even afte 7 days running the same Ubuntu Studio 24.04, installed from the
same USB media following the same process, running the same desktop and
even more applications; but that PC has an NVidia card, like most others.

![System-wide memory usage stays at healthy levels after several days](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/raven-ram-full-after-96-hours.png)

## Remove CUPS

On several occassions `cupsd` had suddently gone up to 200% CPU usage
and stayed up there for 30+ minutes with no sign of and end to come.

``` console
root        5512  0.0  0.0   2892  1664 ?        Ss   Nov16   0:00 /bin/sh /snap/cups/1067/scripts/run-cupsd
root        9582  0.0  0.0  63780 12032 ?        S    Nov16   0:00 cupsd -f -s /var/snap/cups/common/etc/cups/cups-files.conf -c /var/snap/cups/common/etc/cups/cupsd.conf
root     3659053  0.0  0.0   7160  2048 pts/2    S+   00:29   0:00 grep --color=auto cupsd
root     3689314  194  0.0 184064 12416 ?        Rsl  00:00  56:44 /usr/sbin/cupsd -l
```

This PC won't have any printers connected to it, so CUPS can be removed:

``` console
# snap services
Service                                              Startup   Current   Notes
canonical-livepatch.canonical-livepatchd             enabled   active    -
chromium.daemon                                      disabled  inactive  -
cups.cups-browsed                                    enabled   active    -
cups.cupsd                                           enabled   active    -
firmware-updater.firmware-notifier                   enabled   -         user,timer-activated
firmware-updater.firmware-updater-app                enabled   -         user,dbus-activated
snapd-desktop-integration.snapd-desktop-integration  enabled   -         user

# snap stop cups
Stopped.

# snap disable cups
cups disabled

# ps aux | grep cupsd
root     1865839  0.0  0.0  39304 12544 ?        Ss   00:00   0:00 /usr/sbin/cupsd -l
root     2208813  0.0  0.0   9144  1920 pts/1    S+   00:24   0:00 grep --color=auto cupsd

# tail -f /var/log/cups/*log  | grep -v pam_unix
==> /var/log/cups/access_log <==
localhost - - [09/Jan/2025:00:00:00 +0100] "POST / HTTP/1.1" 200 357 Create-Printer-Subscriptions successful-ok
localhost - - [09/Jan/2025:00:00:00 +0100] "POST / HTTP/1.1" 200 176 Create-Printer-Subscriptions successful-ok
localhost - - [09/Jan/2025:00:00:05 +0100] "POST /admin/ HTTP/1.1" 401 0 - -
localhost - cups-browsed [09/Jan/2025:00:00:05 +0100] "POST /admin/ HTTP/1.1" 200 181 CUPS-Delete-Printer client-error-not-found
localhost - - [09/Jan/2025:00:23:29 +0100] "POST / HTTP/1.1" 401 120 Cancel-Subscription successful-ok
localhost - root [09/Jan/2025:00:23:29 +0100] "POST / HTTP/1.1" 200 120 Cancel-Subscription successful-ok

==> /var/log/cups/error_log <==

# snap services
Service                                              Startup   Current   Notes
canonical-livepatch.canonical-livepatchd             enabled   active    -
chromium.daemon                                      disabled  inactive  -
cups.cups-browsed                                    disabled  inactive  -
cups.cupsd                                           disabled  inactive  -
firmware-updater.firmware-notifier                   enabled   -         user,timer-activated
firmware-updater.firmware-updater-app                enabled   -         user,dbus-activated
snapd-desktop-integration.snapd-desktop-integration  enabled   -         user

# systemctl stop cups.service

# ps aux | grep cupsd
root     2271225  0.0  0.0   9144  2176 pts/1    S+   00:29   0:00 grep --color=auto cupsd
```

This issue repeated after reinstalling, lasting for up to 90 minutes:

![cupsd stays at 200% CPU for 90 minutes](../media/2024-11-15-ubuntu-studio-24-04-on-super-tuna-nuc-pc/super-tuna-cupsd-cpu-200.png)

