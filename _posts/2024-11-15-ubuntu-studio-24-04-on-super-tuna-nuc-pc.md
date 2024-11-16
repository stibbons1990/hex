---
title:  "Ubuntu Studio 24.04 on Super Tuna (Nuc PC)"
date:   2024-11-15 21:11:12 +0200
categories: linux ubuntu installation setup intel nuc
---

Installing Ubuntu Studio 24.04 on an Nuc PC desktop.

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/",""  %}

## Preparation

### Partitions

## Installation

## First boot into Ubuntu Studio 24.04

### Disable Power Saving / Suspend

This desktop will need to continue running as normal even when left alone for
extended periods of time. To avoid interruptions, open the **Energy Saving**
section in **System Settings** and disable **Screen Energy Saving** and
**Suspend session**.

### SSH Server

Ubuntu Studio doesn't enable the SSH server by default, but we want
this to adjust the system remotely:

```
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

```
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp86s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 48:21:0b:57:51:92 brd ff:ff:ff:ff:ff:ff
3: enx5c857e3e1129: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 5c:85:7e:3e:11:29 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.155/24 brd 192.168.0.255 scope global dynamic noprefixroute enx5c857e3e1129
       valid_lft 85319sec preferred_lft 85319sec
    inet6 fe80::ced1:e05d:c0f6:9c66/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
4: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether e0:c2:64:39:17:73 brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
    inet 192.168.0.17/24 brd 192.168.0.255 scope global dynamic noprefixroute wlo1
       valid_lft 85323sec preferred_lft 85323sec
    inet6 fe80::91b8:5761:d1bd:1d52/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

### Install Essential Packages

Start by installing a **subset** of the
[essential packages]({{ site.baseurl }}/2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#install-essential-packages):

```
# apt install gdebi-core wget vim curl geeqie playonlinux \
  exfat-fuse clementine id3v2 htop vnstat sox wine \
  python-is-python3 exiv2 rename scrot xcalib python3-pip \
  netcat-openbsd python3-selenium lm-sensors sysstat tor \
  unrar ttf-mscorefonts-installer winetricks icc-profiles \
  ffmpeg iotop-c xdotool inxi mpv screen libxxf86vm-dev \
  displaycal intel-gpu-tools redshift-qt -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
wget is already the newest version (1.21.4-1ubuntu4.1).
wget set to manually installed.
netcat-openbsd is already the newest version (1.226-1ubuntu2).
netcat-openbsd set to manually installed.
sysstat is already the newest version (12.6.1-2).
sysstat set to manually installed.
ffmpeg is already the newest version (7:6.1.1-3ubuntu5).
ffmpeg set to manually installed.
displaycal is already the newest version (3.9.11-2ubuntu0.24.04.1).
displaycal set to manually installed.
The following additional packages will be installed:
  cabextract chromium-browser chromium-chromedriver exiftran fonts-wine fuseiso geeqie-common
  icoutils libasound2-plugins libcapi20-3t64 libgpod-common libgpod4t64 liblastfm5-1
  liblua5.3-0 libmspack0t64 libmygpo-qt5-1 libosmesa6 libpython3-dev libpython3.12-dev
  libsgutils2-1.46-2 libutempter0 libwine libxdo3 libxkbregistry0 libz-mingw-w64 python3-dev
  python3-exceptiongroup python3-h11 python3-natsort python3-outcome python3-sniffio
  python3-trio python3-trio-websocket python3-wsproto python3.12-dev redshift tor-geoipdb
  torsocks tree vim-runtime webp-pixbuf-loader wine64
Suggested packages:
  xpaint libjpeg-progs libterm-readline-gnu-perl | libterm-readline-perl-perl
  libxml-dumper-perl sg3-utils fancontrol read-edid i2c-tools libcuda1 winbind
  python-natsort-doc byobu | screenie | iselect libsox-fmt-all mixmaster torbrowser-launcher
  apparmor-utils nyx obfs4proxy ctags vim-doc vim-scripts vnstati q4wine wine-binfmt dosbox
  wine64-preloader
Recommended packages:
  wine32
The following NEW packages will be installed:
  cabextract chromium-browser chromium-chromedriver clementine curl exfat-fuse exiftran exiv2
  fonts-wine fuseiso gdebi-core geeqie geeqie-common htop icc-profiles icoutils id3v2
  intel-gpu-tools inxi iotop-c libasound2-plugins libcapi20-3t64 libgpod-common libgpod4t64
  liblastfm5-1 liblua5.3-0 libmspack0t64 libmygpo-qt5-1 libosmesa6 libpython3-dev
  libpython3.12-dev libsgutils2-1.46-2 libutempter0 libwine libxdo3 libxkbregistry0
  libxxf86vm-dev libz-mingw-w64 lm-sensors mpv playonlinux python-is-python3 python3-dev
  python3-exceptiongroup python3-h11 python3-natsort python3-outcome python3-pip
  python3-selenium python3-sniffio python3-trio python3-trio-websocket python3-wsproto
  python3.12-dev redshift redshift-qt rename screen scrot sox tor tor-geoipdb torsocks tree
  ttf-mscorefonts-installer unrar vim vim-runtime vnstat webp-pixbuf-loader wine wine64
  winetricks xcalib xdotool
0 upgraded, 75 newly installed, 0 to remove and 16 not upgraded.
Need to get 215 MB of archives.
After this operation, 964 MB of additional disk space will be used.
```

### Brave browser

Install from the
[Release Channel](https://brave.com/linux/#release-channel-installation):

```
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

```
# dpkg -i google-chrome-stable_current_amd64.deb
```

### Continuous Monitoring

Install the
[multi-thread version]({{ site.baseurl }}/conmon/#deploy-to-pcs)
of the `conmon` script as `/usr/local/bin/conmon` and
[run it as a service]({{ site.baseurl }}/conmon/#install-conmon);
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

```
# systemctl enable conmon.service
# systemctl daemon-reload
# systemctl start conmon.service
# systemctl status conmon.service
```

If not already adjusted, add at least the following lines
to `/etc/hosts` so that `lexicon` is reachable:

```
192.168.0.6     lexicon
```

#### Hardware Sensors

All hardware sensors are supported out of the box:

```
# sensors -A
coretemp-isa-0000
Package id 0:  +56.0°C  (high = +100.0°C, crit = +100.0°C)
Core 0:        +50.0°C  (high = +100.0°C, crit = +100.0°C)
Core 4:        +47.0°C  (high = +100.0°C, crit = +100.0°C)
Core 8:        +52.0°C  (high = +100.0°C, crit = +100.0°C)
Core 12:       +49.0°C  (high = +100.0°C, crit = +100.0°C)
Core 16:       +54.0°C  (high = +100.0°C, crit = +100.0°C)
Core 17:       +54.0°C  (high = +100.0°C, crit = +100.0°C)
Core 18:       +54.0°C  (high = +100.0°C, crit = +100.0°C)
Core 19:       +54.0°C  (high = +100.0°C, crit = +100.0°C)
Core 20:       +55.0°C  (high = +100.0°C, crit = +100.0°C)
Core 21:       +55.0°C  (high = +100.0°C, crit = +100.0°C)
Core 22:       +55.0°C  (high = +100.0°C, crit = +100.0°C)
Core 23:       +55.0°C  (high = +100.0°C, crit = +100.0°C)

ucsi_source_psy_USBC000:001-isa-0000
in0:           0.00 V  (min =  +0.00 V, max =  +0.00 V)
curr1:         0.00 A  (max =  +0.00 A)

acpitz-acpi-0
temp1:        +58.0°C  

iwlwifi_1-virtual-0
temp1:        +39.0°C  

ucsi_source_psy_USBC000:002-isa-0000
in0:           0.00 V  (min =  +0.00 V, max =  +0.00 V)
curr1:         3.00 A  (max =  +0.00 A)

nvme-pci-0100
Composite:    +30.9°C  (low  = -20.1°C, high = +83.8°C)
                       (crit = +88.8°C)
Sensor 2:     +47.9°C  

```

There is no need to load the `drivetemp` kernel module because
the only storage in this Nuc PC are NVMe drives, which already
show up as the `nvme-pci-0100` controller above.

## System Configuration

The above having covered **installing** software, there are still
system configurations that need to be tweaked.

### APT respositories clean-up

Ubuntu Studio 24.04 seems to consistently need a little
[APT respositories clean-up]({{ site.baseurl }}2024/09/14/ubuntu-studio-24-04-on-computer-for-a-young-artist.html#apt-respositories-clean-up); just comment out the last line
in `/etc/apt/sources.list.d/dvd.list` to let `noble-security` be
defined (only) in `ubuntu.sources`.

### Ubuntu Pro

When updating the system with `apt full-upgrade -y` a notice comes
up about additional security updates:

```
Get more security updates through Ubuntu Pro with 'esm-apps' enabled:
  libdcmtk17t64 libcjson1 libavdevice60 ffmpeg libpostproc57 libavcodec60
  libavutil58 libswscale7 libswresample4 libavformat60 libavfilter9
Learn more about Ubuntu Pro at https://ubuntu.com/pro
```

This being a new system, indeed it's not attached to an Ubuntu Pro
account (the old system was):

```
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

```
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

```
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

### Make SDDM Look Good

Ubuntu Studio 24.04 uses 
[Simple Desktop Display Manager (SDDM)](https://wiki.archlinux.org/title/SDDM)
([sddm/sddm](https://github.com/sddm/sddm) in GitHub)
which is quite good looking out of the box, but I like to
customize this for each computer.

For most computers my favorite SDDM theme is
[Breeze-Noir-Dark](https://store.kde.org/p/1361460),
which I like to install system-wide.

```
# unzip -d /usr/share/sddm/themes Breeze-Noir-Dark.zip
```

**Note:** action icons won’t render if the directory name is
changed. If needed, change the directory name in the `iconSource` fields in `Main.qml` to match final directory name
so icons show. This is not the only thing that breaks when
changing the directory name.

Other than installing this theme, all I really change in it
is the background image to use 
[The Rapture [3440x1440]](https://www.flickr.com/photos/douglastofoli/27740699244/).

```
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

```
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

### Make `dmesg` non-privileged

Since Ubuntu 22.04, `dmesg` has become a privileged operation
by default:

```
$ dmesg
dmesg: read kernel buffer failed: Operation not permitted
```

This is controlled by 

```
# sysctl kernel.dmesg_restrict
kernel.dmesg_restrict = 1
```

To revert this default, and make it permanent
([source](https://archived.forum.manjaro.org/t/why-did-dmesg-become-a-priveleged-operation-suddenly/86468/3)):

```
# echo 'kernel.dmesg_restrict=0' | tee -a /etc/sysctl.d/99-sysctl.conf
```

### Weekly btrfs scrub

To keep BTRFS file systems healthy, it is recommended to
[run a weekly scrub](http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html)
to check everything for consistency. For this, I run
[the script](https://marc.merlins.org/linux/scripts/btrfs-scrub)
from crontab every Saturday night.

```
# wget -O /usr/local/bin/btrfs-scrub-all \
  http://marc.merlins.org/linux/scripts/btrfs-scrub

# apt install inn -y

# crontab -e
...
# m h  dom mon dow   command
50 2 * * 6 /usr/local/bin/btrfs-scrub-all
```

[Marc MERLIN](https://marc.merlins.org/) keeps the script updated,
so each systme may benefit from a few modifications, e.g.
1. Remove tests for laptop battery status, when running on a PC.
2. Set the `BTRFS_SCRUB_SKIP` to filter out partitions to skip.

```bash
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

**Note:** setting `BTRFS_SCRUB_SKIP="sda"` prefents Btrfs balancing from running
every week on the larger 6TB RAID 1 array of HDD, because that takes too long.

```
# /usr/local/bin/btrfs-scrub-all
<13>Nov 16 00:25:47 root: Quick Metadata and Data Balance of /home (/dev/nvme0n1p4)
Done, had to relocate 0 out of 6 chunks
Done, had to relocate 0 out of 6 chunks
Done, had to relocate 3 out of 6 chunks
<13>Nov 16 00:26:17 root: Starting scrub of /home
btrfs scrub start -Bd /home
Starting scrub on devid 1

Scrub device /dev/nvme0n1p4 (id 1) done
Scrub started:    Sat Nov 16 00:26:17 2024
Status:           finished
Duration:         0:00:00
Total to scrub:   45.99MiB
Rate:             45.99MiB/s
Error summary:    no errors found

real    0m0.025s
user    0m0.001s
sys     0m0.018s
```

The whole process takes about 25 seconds because the 4TB NVMe SSD is
mostly empty, with only 52 MB used.

### Stop apparmor spew in the logs

Although also somewhat visible in Ubuntu Studio 22.04, the *log spam*
from `audit` seems to flood `dmesg` output a lot more in 24.04.

No worries, it is rather easy to
[stop apparmor spew in dmesg](https://askubuntu.com/questions/1336845/how-to-stop-apparmor-spew-in-the-logs),
just install `auditd` and (most of) the messages will go to
`/var/log/kern.log` instead.

```
# apt install auditd -y
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libauparse0t64
Suggested packages:
  audispd-plugins
The following NEW packages will be installed:
  auditd libauparse0t64
0 upgraded, 2 newly installed, 0 to remove and 5 not upgraded.
Need to get 274 kB of archives.
After this operation, 893 kB of additional disk space will be used.
```

### Stop Memory Leaks

Only in this Nuc PC is Ubuntu Studio 24.04 showing a huge memory leak
and it seems to be in the kernel rather than in any application/s:

```
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

![Memory usage in the last 9 hours]({{ media }}/super-tuna-last-9-h.png)

Killing the most memory intensive process had previously made no dent:

```
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

The trick in [linuxatemyram.com](https://www.linuxatemyram.com/)
to drop caches did not help:

```
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

![Memory usage in the last 15 minutes]({{ media }}/super-tuna-last-15-min.png)

There was another hint somewhere (Red Hat Oracle documentation)
about the number of "huge pages" but that is already set to zero:

```
# cat /proc/sys/vm/nr_hugepages 
0
```

After a wider and deeper investigation
(see ~1500 lines in [this log]({{ media }}/2024-11-16-14-50.txt))
it turned out be a memory leak bug in the `i915` driver:

```
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

#### Maybe in the i915 driver

Since the `i915` and `xe` drivers are taking more memory than
all others, it looked like this could be a case of memory leak
in the driver (e.g. like
[strongtz/i915-sriov-dkms issue #137](https://github.com/strongtz/i915-sriov-dkms/issues/137)).
However, this system is using the "vanilla" `i915` drivers and,
after the memory has been released by rebooting the system,
the drivers still show about the same size:

```
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

**TODO:** Try [BuildYourOwnKernel](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel)
with that one-line patch and see if that helps. Maybe reinstall the system.

#### Maybe somewhere else

If the `i915` is not behind this memory leak, the one other lead from
[this log]({{ media }}/2024-11-16-14-50.txt) is, by far, the largest
user of kernel memory is `kmalloc-rnd-07-8k`:

```
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

```
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

```
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
- `btrfs_inode` in `rapture`, with 1,149,792K after a good 20 hours up
- `kmalloc-rnd-12-8k` in `super-tuna`, with 2,391,360K after barely 1 hour up

### Bluetooth controller and devices

The following shows up in `dmesg`:

```
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

**TODO: pair headphones**
