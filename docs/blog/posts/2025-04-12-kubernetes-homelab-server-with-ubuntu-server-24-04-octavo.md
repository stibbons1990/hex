---
date: 2025-04-12
draft: true
categories:
 - Linux
 - Hardware
 - Ubuntu
 - Server
 - Btrfs
 - Intel NUC
title: Kubernetes homelab server with Ubuntu Server 24.04 (octavo)
---

*Need. More. Server. Need. More. POWER!!!*

Just a bit more, maybe *quite a bit* more, to run services that are more
CPU-intensive than those already running in the current 
[Single-node Kubernetes cluster on an Intel NUC: lexicon](./2023-03-25-single-node-kubernetes-cluster-on-ubuntu-server-lexicon.md).

<!-- more --> 

## Hardware

Ubuntu's list of
[Recommended and Certified Hardware](https://ubuntu.com/appliance/hardware#intel-nuc)
is just as short and *ancient* as it was 2 years ago, but NUC systems have a now
long history of being well suppored under Linux, so this time I'm taking chances:

*  [ASUS NUC 13 Pro Tall PC Kit RNUC13ANHI700000I w/ Intel Core i7-1360P](https://webshop.asus.com/de/Mini-PCs/ASUS-NUC-13-Pro-Tall-PC-Kit-RNUC13ANHI700000I/90AR00C1-M000F0) ($570)
*  [Kingston FURY Impact 1x 32GB, 3200 MHz, DDR4-RAM, SODIMM](https://www.kingston.com/en/memory/gaming/kingston-fury-impact-ddr4-memory?speed=3200mt%2Fs&total%20(kit)%20capacity=32gb&kit=single%20module&dram%20density=16gbit) ($110)
*  [Kingston FURY Renegade 4000 GB, M.2 2280](https://www.kingston.com/en/ssd/gaming/kingston-fury-renegade-nvme-m2-ssd) ($250)

### Bootable USB stick

[Get Ubuntu Server](https://ubuntu.com/download/server) (24.04.**2** LTS) and 
[create a bootable USB stick on Ubuntu](https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#1-overview).

## Install Ubuntu Server 24.04

[Installing Ubuntu Server 24.04](https://ubuntu.com/tutorials/install-ubuntu-server)
went smoothly and without any problems, the NUC booted from the USB stick and secure
boot, enabled by default, never presented any problem. Screen and USB keyword worked
seamlessly through a DisplayPort+USB+Ethernet
[Cable Matters Hub](https://www.cablematters.com/pc-940-126-usb-c-dual-monitor-hub-with-dual-4k-displayport-2x-usb-20-fast-ethernet-and-60w-power-delivery.aspx#)
on a Thunderbolt port in the NUC.

Once the intaller boots, the installation steps were:

1.  Choose language and keyboard layout.
1.  Choose **Ubuntu Server** (default, not *(minimized)*).
    *  Checked the option to *Search for third-party drivers*.
1.  Networking: DHCP on wired network.
    *  IP address `.155` is assgined to the `enx5c857e3e1129` interface, the
       RTL8153 Gigabit Ethernet NIC in the Cable Matters Hub.
    *  The `enp86s0` interface is the NUC's integrated 2.5Gbps NIC (Intel I226-V),
       which during the installation had no network cable attached.
1.  Pick a local Ubuntu mirror to install packages from.
1.  Setup a **Custom storage layout** as follows
    1.  Select the disk (Kingston FURY Renegade 4000 GB) to **Use As Boot Device**.
        This automatically creates a 1GB partition for `/boot/efi` (formatted as `btrfs`).
    1.  Create a 60G partition to mount as `/` (formatted as `btrfs`).
    1.  Create a 60G partition to reverse for a future OS.
    1.  Create a 60G partition to mount as `/var/lib` (formatted as `btrfs`).
    1.  Create a  partition using all remaining space (3.46T) to mount as `/home`
        (formatted as `btrfs`).
1.  Confirm partitions & changes.
1.  Set up a Profile: username (`ponder`), hostname (`octavo`) and password.
1.  Skip Upgrade to Ubuntu Pro (to be done later).
1.  Install OpenSSH server and allow password authentication (for now).
1.  A selection of snap packages is available at this point, none were selected.
1.  Confirm all previous choices and start to **install software**.
1.  Once the installation is complete, remove the UBS stick and hit Enter to reboot.

### Tweak OpenSSH server

Set the `root` password by first escalating with `sudo su -` and then using the
`passwd` command to set the password. This password will hardly ever be used, but
should be set so that it can be use in case of emergency.

Copy SSH public keys into the `.ssh/authorized_keys` of both `ponder` and `root`,
then disable password authentication in the OpenSSH server:

``` console
# vi /etc/ssh/sshd_config
PasswordAuthentication no
# systemctl restart ssh
```

Test SSH connections as both `ponder` and `root` from all the relevant hosts in the LAN.

#### Setup Fail2Ban

On top of disabling password authentication, the SSH server will be less busy is those
pesky *bad actors* are blocked from reaching it's port. To do this, install
[fail2ban](https://github.com/fail2ban/fail2ban):

??? terminal "`apt install fail2ban -y`"

    ``` console
    # apt install fail2ban -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
      python3-pyasyncore python3-pyinotify whois
    Suggested packages:
      mailx monit sqlite3 python-pyinotify-doc
    The following NEW packages will be installed:
      fail2ban python3-pyasyncore python3-pyinotify whois
    0 upgraded, 4 newly installed, 0 to remove and 1 not upgraded.
    Need to get 496 kB of archives.
    After this operation, 2,572 kB of additional disk space will be used.
    Get:1 http://ch.archive.ubuntu.com/ubuntu noble/main amd64 python3-pyasyncore all 1.0.2-2 [10.1 kB]
    Get:2 http://ch.archive.ubuntu.com/ubuntu noble-updates/universe amd64 fail2ban all 1.0.2-3ubuntu0.1 [409 kB]
    Get:3 http://ch.archive.ubuntu.com/ubuntu noble/main amd64 python3-pyinotify all 0.9.6-2ubuntu1 [25.0 kB]
    Get:4 http://ch.archive.ubuntu.com/ubuntu noble/main amd64 whois amd64 5.5.22 [51.7 kB]
    Fetched 496 kB in 0s (2,059 kB/s) 
    Selecting previously unselected package python3-pyasyncore.
    (Reading database ... 86910 files and directories currently installed.)
    Preparing to unpack .../python3-pyasyncore_1.0.2-2_all.deb ...
    Unpacking python3-pyasyncore (1.0.2-2) ...
    Selecting previously unselected package fail2ban.
    Preparing to unpack .../fail2ban_1.0.2-3ubuntu0.1_all.deb ...
    Unpacking fail2ban (1.0.2-3ubuntu0.1) ...
    Selecting previously unselected package python3-pyinotify.
    Preparing to unpack .../python3-pyinotify_0.9.6-2ubuntu1_all.deb ...
    Unpacking python3-pyinotify (0.9.6-2ubuntu1) ...
    Selecting previously unselected package whois.
    Preparing to unpack .../whois_5.5.22_amd64.deb ...
    Unpacking whois (5.5.22) ...
    Setting up whois (5.5.22) ...
    Setting up python3-pyasyncore (1.0.2-2) ...
    Setting up fail2ban (1.0.2-3ubuntu0.1) ...
    /usr/lib/python3/dist-packages/fail2ban/tests/fail2banregextestcase.py:224: SyntaxWarning: invalid escape sequence '\s'
      "1490349000 test failed.dns.ch", "^\s*test <F-ID>\S+</F-ID>"
    /usr/lib/python3/dist-packages/fail2ban/tests/fail2banregextestcase.py:435: SyntaxWarning: invalid escape sequence '\S'
      '^'+prefix+'<F-ID>User <F-USER>\S+</F-USER></F-ID> not allowed\n'
    /usr/lib/python3/dist-packages/fail2ban/tests/fail2banregextestcase.py:443: SyntaxWarning: invalid escape sequence '\S'
      '^'+prefix+'User <F-USER>\S+</F-USER> not allowed\n'
    /usr/lib/python3/dist-packages/fail2ban/tests/fail2banregextestcase.py:444: SyntaxWarning: invalid escape sequence '\d'
      '^'+prefix+'Received disconnect from <F-ID><ADDR> port \d+</F-ID>'
    /usr/lib/python3/dist-packages/fail2ban/tests/fail2banregextestcase.py:451: SyntaxWarning: invalid escape sequence '\s'
      _test_variants('common', prefix="\s*\S+ sshd\[<F-MLFID>\d+</F-MLFID>\]:\s+")
    /usr/lib/python3/dist-packages/fail2ban/tests/fail2banregextestcase.py:537: SyntaxWarning: invalid escape sequence '\['
      'common[prefregex="^svc\[<F-MLFID>\d+</F-MLFID>\] connect <F-CONTENT>.+</F-CONTENT>$"'
    /usr/lib/python3/dist-packages/fail2ban/tests/servertestcase.py:1375: SyntaxWarning: invalid escape sequence '\s'
      "`{ nft -a list chain inet f2b-table f2b-chain | grep -oP '@addr-set-j-w-nft-mp\s+.*\s+\Khandle\s+(\d+)$'; } | while read -r hdl; do`",
    /usr/lib/python3/dist-packages/fail2ban/tests/servertestcase.py:1378: SyntaxWarning: invalid escape sequence '\s'
      "`{ nft -a list chain inet f2b-table f2b-chain | grep -oP '@addr6-set-j-w-nft-mp\s+.*\s+\Khandle\s+(\d+)$'; } | while read -r hdl; do`",
    /usr/lib/python3/dist-packages/fail2ban/tests/servertestcase.py:1421: SyntaxWarning: invalid escape sequence '\s'
      "`{ nft -a list chain inet f2b-table f2b-chain | grep -oP '@addr-set-j-w-nft-ap\s+.*\s+\Khandle\s+(\d+)$'; } | while read -r hdl; do`",
    /usr/lib/python3/dist-packages/fail2ban/tests/servertestcase.py:1424: SyntaxWarning: invalid escape sequence '\s'
      "`{ nft -a list chain inet f2b-table f2b-chain | grep -oP '@addr6-set-j-w-nft-ap\s+.*\s+\Khandle\s+(\d+)$'; } | while read -r hdl; do`",
    Created symlink /etc/systemd/system/multi-user.target.wants/fail2ban.service → /usr/lib/systemd/system/fail2ban.service.
    Setting up python3-pyinotify (0.9.6-2ubuntu1) ...
    Processing triggers for man-db (2.12.0-4build2) ...
    Scanning processes...                                                                                         
    Scanning processor microcode...                                                                               
    Scanning linux images...                                                                                      

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    ```

Enable the service so it starts after each system restart:

``` console
# systemctl enable --now fail2ban
Synchronizing state of fail2ban.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
```

The default configuration should be enough because this system will probably not expose
its port 22 to the Internet, but instead accessed remotely via [Tailscale](#tailscale).
If port 22 is later expoed to the Internet, see
[lexicon's Fail2ban setup](./2022-07-03-low-effort-homelab-server-with-ubuntu-server-on-intel-nuc.md#setup-fail2ban).

### Tweak network config

This system will use static IP addresses `.8` so they can be added to the `/etc/hosts`
file of the relevant hosts in the LAN, then the file can be copied into the new server,
preserving the line that points `127.0.0.1` to itself.

While using the Cable Matters hub, the LAN IP address is setup on the `enx5c857e3e1129`
interface:

``` console hl_lines="10 12"
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp86s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 48:21:0b:6d:3e:9b brd ff:ff:ff:ff:ff:ff
3: enx5c857e3e1129: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 5c:85:7e:3e:11:29 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.155/24 metric 100 brd 192.168.0.255 scope global dynamic enx5c857e3e1129
       valid_lft 85920sec preferred_lft 85920sec
    inet6 fe80::5e85:7eff:fe3e:1129/64 scope link 
       valid_lft forever preferred_lft forever
4: wlo1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d0:65:78:a5:8b:dd brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
```

??? terminal "`dmesg` lines for Intel I226-V 2.5Gbps NIC (`igc`) and USB NIC (`r8152`):"

    ``` dmesg
    [    1.736823] Intel(R) 2.5G Ethernet Linux Driver
    [    1.738154] Copyright(c) 2018 Intel Corporation.
    [    1.741460] igc 0000:56:00.0: enabling device (0000 -> 0002)
    [    1.744312] igc 0000:56:00.0: PTM enabled, 4ns granularity
    [    1.756421] ahci 0000:00:17.0: version 3.0
    [    1.756759] xhci_hcd 0000:00:0d.0: xHCI Host Controller
    --
    [    1.821412] intel-lpss 0000:00:15.1: enabling device (0004 -> 0006)
    [    1.823953] idma64 idma64.1: Found Intel integrated DMA 64-bit
    [    1.842492] igc 0000:56:00.0: 4.000 Gb/s available PCIe bandwidth (5.0 GT/s PCIe x1 link)
    [    1.843094] igc 0000:56:00.0 eth0: MAC: 48:21:0b:6d:3e:9b
    [    2.056525] typec port0: bound usb3-port6 (ops connector_ops)
    [    2.058975] typec port0: bound usb2-port1 (ops connector_ops)
    [    2.061054] usb 3-1: new full-speed USB device number 2 using xhci_hcd
    [    2.086270] ata2: SATA link down (SStatus 4 SControl 300)
    [    2.092729] igc 0000:56:00.0 enp86s0: renamed from eth0
    [    2.100868] ucsi_acpi USBC000:00: UCSI_GET_PDOS failed (-95)
    [    2.114965] nvme 0000:01:00.0: platform quirk: setting simple suspend
    [    2.116031] nvme nvme0: pci function 0000:01:00.0
    [    2.125670] nvme nvme0: Shutdown timeout set to 10 seconds
    [    2.129358] nvme nvme0: 16/0/0 default/read/poll queues
    --
    [    4.164720] usbcore: registered new device driver r8152-cfgselector
    [    4.352207] r8152-cfgselector 3-6.2: reset high-speed USB device number 6 using xhci_hcd
    [    4.581005] r8152 3-6.2:1.0: load rtl8153a-4 v2 02/07/20 successfully
    [    4.621215] r8152 3-6.2:1.0 eth0: v1.12.13
    [    4.624728] usbcore: registered new interface driver r8152
    [    4.660226] usbcore: registered new interface driver cdc_ether
    [    4.669749] r8152 3-6.2:1.0 enx5c857e3e1129: renamed from eth0
    [    4.720488] usb 3-6.4.1: new high-speed USB device number 8 using xhci_hcd
    [    4.808357] raid6: avx2x4   gen() 29500 MB/s
    [    4.825355] raid6: avx2x2   gen() 36875 MB/s
    [    4.842356] raid6: avx2x1   gen() 35157 MB/s
    [    4.842362] raid6: using algorithm avx2x2 gen() 36875 MB/s
    ```

Servers are better setup with static IP addresses in a know range, and for this system
the `.8 `addresses have been reserved and can be setup in the Netplan configuration:

``` yaml title="/etc/netplan/50-cloud-init.yaml" hl_lines="4 8 16 20"
network:
  version: 2
  ethernets:
    enx5c857e3e1129:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.7/24, 192.168.0.7/24 ]
      # Set default gateway
      routes:
       - to: default
         via: 192.168.0.1
      # Set DNS name servers
      nameservers:
        addresses: [62.2.24.158, 62.2.17.61]
    enp86s0:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.8/24, 192.168.0.8/24 ]
      # Set default gateway
      # Set DNS name servers
      nameservers:
        addresses: [62.2.24.158, 62.2.17.61]
```

!!! note
    
    Adding `routes` to both Ethernet interfaces will result in warnings, and is not
    necessary so long as one can SSH in from another host in the LAN.

After applying the configuration with `netplan apply` both Ethernet NICs will always have
static IP addresses:

``` console hl_lines="10 14"
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp86s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 48:21:0b:6d:3e:9b brd ff:ff:ff:ff:ff:ff
3: enx5c857e3e1129: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 5c:85:7e:3e:11:29 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.7/24 brd 10.0.0.255 scope global enx5c857e3e1129
       valid_lft forever preferred_lft forever
    inet 192.168.0.7/24 brd 192.168.0.255 scope global enx5c857e3e1129
       valid_lft forever preferred_lft forever
    inet6 fe80::5e85:7eff:fe3e:1129/64 scope link 
       valid_lft forever preferred_lft forever
4: wlo1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d0:65:78:a5:8b:dd brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
```

The server can now be relocated to the **2.5Gbps** network, where no screen is available.
Once relocated to the 2.5Gbps network, no longer using the Cable Matters hub, and finished
booting, SSH back into the server using the `.8` address:

``` console hl_lines="8 10"
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp86s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 48:21:0b:6d:3e:9b brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.8/24 brd 10.0.0.255 scope global enp86s0
       valid_lft forever preferred_lft forever
    inet 192.168.0.8/24 brd 192.168.0.255 scope global enp86s0
       valid_lft forever preferred_lft forever
    inet6 fe80::4a21:bff:fe6d:3e9b/64 scope link 
       valid_lft forever preferred_lft forever
3: wlo1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether d0:65:78:a5:8b:dd brd ff:ff:ff:ff:ff:ff
    altname wlp0s20f3
```

To re-enable the server's access to the Internet, move the `routes` to the `enp86s0`
interface and `netplan apply` once again:

``` yaml title="/etc/netplan/50-cloud-init.yaml"
network:
  version: 2
  ethernets:
    enx5c857e3e1129:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.7/24, 192.168.0.7/24 ]
    enp86s0:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.8/24, 192.168.0.8/24 ]
      # Set default gateway
      routes:
       - to: default
         via: 192.168.0.1
      # Set DNS name servers
      nameservers:
        addresses: [62.2.24.158, 62.2.17.61]
```

At this point the network configuration is finalized.

### Set correct timezone

The installation process does not offer setting the system timezone and defaults to UTC:

``` console
# timedatectl
               Local time: Sun 2025-04-13 20:47:28 UTC
           Universal time: Sun 2025-04-13 20:47:28 UTC
                 RTC time: Sun 2025-04-13 20:47:28
                Time zone: Etc/UTC (UTC, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

Since this is not the local timezone anywhere, it is better (more convenient) to set the
local timezone, e.g.

``` console
# timedatectl set-timezone "Europe/Amsterdam"

# timedatectl
               Local time: Sun 2025-04-13 22:48:45 CEST
           Universal time: Sun 2025-04-13 20:48:45 UTC
                 RTC time: Sun 2025-04-13 20:48:45
                Time zone: Europe/Amsterdam (CEST, +0200)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

### Update system packages

Updating system packages is recommended after installing from USB media:

``` console
# apt update && apt full-upgrade -y
```

### Tweak Bash prompt

Tweak Bash prompt for `root` to make user name
<span style="color: red">red</span>,
host name
<span style="color: blue">blue</span>
and path
<span style="color: green">green</span>;
with this in `.bashrc`:


``` bash title=".bashrc" linenums="36" hl_lines="4 18 20"
# uncomment for a colored prompt, if the terminal has the capability; turned
# off by default to not distract the user: the focus in a terminal window
# should be on the output of commands, not on the prompt
force_color_prompt=yes

if [ -n "$force_color_prompt" ]; then
    if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then
        # We have color support; assume it's compliant with Ecma-48
        # (ISO/IEC-6429). (Lack of such support is extremely rare, and such
        # a case would tend to support setf rather than setaf.)
        color_prompt=yes
    else
        color_prompt=
    fi
fi

if [ "$color_prompt" = yes ]; then
    PS1='\[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\[\033[01;31m\]\u@\[\033[01;34m\]\h\[\033[00m\] \[\033[01;32m\]\w \$\[\033[00m\] '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h \w \$ '
fi
unset color_prompt force_color_prompt
```

Other users' Bash prompt is left as default, which renders all
green. The idea is that `root`'s prompt is visually different,
to remind me that *with great power comes great responsibility*.

### Upgrade to Ubuntu Pro

This step was skipped during the
[installation process](#install-ubuntu-server-2404)
so the server is not attached to an Ubuntu Pro account:

``` console
# pro security-status
684 packages installed:
    684 packages from Ubuntu Main/Restricted repository

To get more information about the packages, run
    pro security-status --help
for a list of available options.

This machine is receiving security patching for Ubuntu Main/Restricted
repository until 2029.
This machine is NOT attached to an Ubuntu Pro subscription.

Ubuntu Pro with 'esm-infra' enabled provides security updates for
Main/Restricted packages until 2034.

Try Ubuntu Pro with a free personal subscription on up to 5 machines.
Learn more at https://ubuntu.com/pro
```

Take the command and token from <https://ubuntu.com/pro/dashboard> and
attach the server to an active Ubuntu Pro account:

``` console
# pro attach ______________________________
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
livepatch        yes       enabled      Canonical Livepatch service
realtime-kernel* yes       disabled     Ubuntu kernel with PREEMPT_RT patches integrated
usg              yes       disabled     Security compliance and audit tools

 * Service has variants

NOTICES
Operation in progress: pro attach

For a list of all Ubuntu Pro services and variants, run 'pro status --all'
Enable services with: pro enable <service>

     Account: ponder.stibbons@uu.am
Subscription: Ubuntu Pro - free personal subscription
```

### Weekly btrfs scrub

To keep BTRFS file systems healthy, it is recommended to
[run a weekly scrub](http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html)
to check everything for consistency. For this, run
[the script](https://marc.merlins.org/linux/scripts/btrfs-scrub)
from `crontab` every Saturday morning, early enough that it will
be done by the time anyone wakes up:

??? code "`/usr/local/bin/btrfs-scrub-all`"

    ``` bash
    #! /bin/bash

    # By Marc MERLIN <marc_soft@merlins.org> 2014/03/20
    # License: Apache-2.0
    # http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html

    which btrfs >/dev/null || exit 0
    export PATH=/usr/local/bin:/sbin:$PATH

    FILTER='(^Dumping|balancing, usage)'
    test -n "$DEVS" || DEVS=$(grep '\<btrfs\>' /proc/mounts | awk '{ print $1 }' | sort -u)
    for btrfs in $DEVS
    do
        tail -n 0 -f /var/log/syslog | grep -i "BTRFS" | grep -Evi '(disk space caching is enabled|unlinked .* orphans|turning on discard|device label .* devid .* transid|enabling SSD mode|BTRFS: has skinny extents|BTRFS: device label)' &
        mountpoint="$(grep "$btrfs" /proc/mounts | awk '{ print $2 }' | sort | head -1)"
        logger -s "Quick Metadata and Data Balance of $mountpoint ($btrfs)" >&2
        # Even in 4.3 kernels, you can still get in places where balance
        # won't work (no place left, until you run a -m0 one first)
        btrfs balance start -musage=0 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
        btrfs balance start -musage=20 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
        # After metadata, let's do data:
        btrfs balance start -dusage=0 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
        btrfs balance start -dusage=20 -v $mountpoint 2>&1 | grep -Ev "$FILTER"
        # And now we do scrub. Note that scrub can fail with "no space left
        # on device" if you're very out of balance.
        logger -s "Starting scrub of $mountpoint" >&2
        echo btrfs scrub start -Bd $mountpoint
        ionice -c 3 nice -10 btrfs scrub start -Bd $mountpoint
        pkill -f 'tail -n 0 -f /var/log/syslog'
        logger "Ended scrub of $mountpoint" >&2
    done
    ```

``` console
# crontab -l | grep btrfs
# m h  dom mon dow   command
50 5 * * 6 /usr/local/bin/btrfs-scrub-all
```

### Stop `apparmor` spew in the logs

As seen in previous installs of Ubuntu 24.04 on Rapture and Raven, there is some
log spam from `audit` in the `dmesg` logs.
[To stop this](./2024-11-03-ubuntu-studio-24-04-on-rapture-gaming-pc-and-more.md#stop-apparmor-spew-in-the-logs) 
simply install `auditd`:

??? terminal "`apt install auditd`"

    ``` console
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
    0 upgraded, 2 newly installed, 0 to remove and 1 not upgraded.
    Need to get 274 kB of archives.
    After this operation, 893 kB of additional disk space will be used.
    Get:1 http://ch.archive.ubuntu.com/ubuntu noble-updates/main amd64 libauparse0t64 amd64 1:3.1.2-2.1build1.1 [58.9 kB]
    Get:2 http://ch.archive.ubuntu.com/ubuntu noble-updates/main amd64 auditd amd64 1:3.1.2-2.1build1.1 [215 kB]
    Fetched 274 kB in 0s (1,148 kB/s)
    Selecting previously unselected package libauparse0t64:amd64.
    (Reading database ... 87398 files and directories currently installed.)
    Preparing to unpack .../libauparse0t64_1%3a3.1.2-2.1build1.1_amd64.deb ...
    Adding 'diversion of /lib/x86_64-linux-gnu/libauparse.so.0 to /lib/x86_64-linux-gnu/libauparse.so.0.usr-is-merged by libauparse0t64'
    Adding 'diversion of /lib/x86_64-linux-gnu/libauparse.so.0.0.0 to /lib/x86_64-linux-gnu/libauparse.so.0.0.0.usr-is-merged by libauparse0t64'
    Unpacking libauparse0t64:amd64 (1:3.1.2-2.1build1.1) ...
    Selecting previously unselected package auditd.
    Preparing to unpack .../auditd_1%3a3.1.2-2.1build1.1_amd64.deb ...
    Unpacking auditd (1:3.1.2-2.1build1.1) ...
    Setting up libauparse0t64:amd64 (1:3.1.2-2.1build1.1) ...
    Setting up auditd (1:3.1.2-2.1build1.1) ...
    Created symlink /etc/systemd/system/multi-user.target.wants/auditd.service → /usr/lib/systemd/system/auditd.service.
    Processing triggers for man-db (2.12.0-4build2) ...
    Processing triggers for libc-bin (2.39-0ubuntu8.4) ...
    Scanning processes...                                                                                         
    Scanning processor microcode...                                                                               
    Scanning linux images...                                                                                      

    Running kernel seems to be up-to-date.

    The processor microcode seems to be up-to-date.

    No services need to be restarted.

    No containers need to be restarted.

    No user sessions are running outdated binaries.

    No VM guests are running outdated hypervisor (qemu) binaries on this host.
    ```

### NAS NFS mount

Add the [NFS mount](./2025-04-18-synology-ds423-for-the-homelab-luggage.md#nfs)
in `/etc/fstab` to mount `/home/nas`, create the directory and mount it.

## Continuous Monitoring

[Install Continuous Monitoring](../../projects/conmon.md#install-conmon) and
report metrics to `lexicon` on its `NodePort` (30086).

## Remote Access

### Cloudflare

### Tailscale

## Kubernetes
