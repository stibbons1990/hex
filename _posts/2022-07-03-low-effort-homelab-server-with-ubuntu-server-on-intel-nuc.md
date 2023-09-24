---
title:  "Low-effort homelab server with Ubuntu Server on Intel NUC"
date:   2022-07-03 22:07:03 +0200
categories: linux hardware ubuntu server btrfs intelnuc
---

*Need. More. Server. Need. More. POWER!!!*

{% assign media = site.baseurl | append: "/assets/media/" | append:  page.path | replace: ".md","" | replace: "_posts/","" %}

But only a little bit, maybe *just enough* to run a Minecraft
server, which refuses to start on my **Raspberry Pi 4**
because it has only a meagre 2 GB of RAM.

I had known about Intel NUC tiny PCs for a while, and
how handy they can be to have a dedicated physical PC for
experimentation. There was a very real possibility
that I would have to set one up as a *light gaming* PC in the
near future, so I thought cutting my teeth on a *simpler*
server setup would be a good way to get acquainted with this
hardware platform and its Linux support.

## Hardware

Ubuntu's list of
[Recommended and Certified Hardware](https://ubuntu.com/appliance/hardware#intel-nuc)
is quite short and includes only *very* old models,
and searching the web for this model in relation to Linux
or Ubuntu tends to yield mostly useless results (e.g. shop
listings, or forum threads related to other models).

Not being at all sure this would be a good choice of platform,
I kept this build low-cost by choosing the low-end on the CPU:

*  [Intel® NUC 11 Performance kit - NUC11PAHi3](https://www.intel.com/content/www/us/en/products/sku/205033/intel-nuc-11-performance-kit-nuc11pahi3/specifications.html) ($285)
*  [VENGEANCE® Series 32GB RAM (2 x 16GB) DDR4 SODIMM](https://www.corsair.com/ww/en/p/memory/cmsx32gx4m2a3200c22/vengeancea-series-32gb-2x16gb-ddr4-sodimm-3200mhz-cl22-memory-kit-cmsx32gx4m2a3200c22) ($120)
*  [SSD 970 EVO Plus NVMe® M.2 2 TB SSD](https://www.samsung.com/us/computing/memory-storage/solid-state-drives/ssd-970-evo-plus-nvme-m-2-2-tb-mz-v7s2t0b-am/) ($180)

### Update #1 (2022-10-12)

Encouraged by a sudden price fall (by 15% down to $300), and
spurred by the recent
[failure of 6TB HDD RAID](https://stibbons1990.github.io/hex/2022/09/27/undead-yes-unraid-no.html),
I added a
[Crucial MX500 4TB 3D NAND SATA SSD](https://www.crucial.com/ssd/mx500/ct4000mx500ssd1) to serve as an backup to some of my
precious files in that cursed RAID.

Turns out, [Crucial MX500 SSD are problematic]({{ site.baseurl }}/2022/10/12/crucial-mx500-ssd-found-problematic.html).

### Update #2 (2023-09-23)

Often I wonder whether 32 GB of RAM was too much, and it
probably was. I hardly ever see any process *use* more than
5 GB, the whole system hardly ever has more thn 10 GB *used*.
On the other hand, maybe it *is* better to have enough RAM for
those rare occasions when it's needed.
[Just once in the last 30 days](http://lexicon:3000/d/wbPxv76nz/lexicon?orgId=1&from=1694032836167&to=1694034280178),
did one process (InfluxDB) got up to 17 GB of *used* RAM,
most likely because I made an *unreasonable request* such as
retrieving too many days' worth of data.

![RAM usage chart shows InfluxDB taking up to 21 GB at peak]({{ media }}/ram-used-17gb-influxdb.png)

## Ubuntu Desktop 22.04

Although the intended use for this mini PC is to serve as a
server, this being my first experience with this hardware
platform I tried first installing
[Ubuntu Desktop 22.04](https://help.ubuntu.com/stable/ubuntu-help/) to see what may be different on this hardware.

The most important difference between Intel NUC is that,
starting precisely at this *generation* (11),
[NUC 11 Kits & Mini PCs no longer support Legacy BIOS Support](https://www.intel.com/content/www/us/en/support/articles/000057401/intel-nuc.html). It seems even in the previous
generation, this was already the case for
*Performance Kits* (NUC10ixFN).
This means **Secure Boot is required**, which posed a bit of
a challenge for me since I had never had to set this up.

With that in mind, the installation itself was simple as it
tends to be the case with Ubuntu:

1. Boot from USB, with the option to **Try or Install Ubuntu**
1. Select language (English) and then **Install Ubuntu**
1. Select keyboard layout and *update and other software* to
   *Install third-party software for graphics and Wifi hardware and additional media formats*, on top of an
   otherwise *Minimal installation*.
1. Configure **Secure Boot** (simply provide a password).
1. Create a new partition table on `/dev/nvme0n1` with
   4 *primary* partitions:
   1. 512 MB EFI System Partition mounted on `/boot/efi`
   1. 50,000 MB **btrfs** mounted on `/`
   1. 50,000 MB **btrfs** without mount point (for future use)
   1. 1,899,886 MB (remainder) **btrfs** mounted on `/home`
1. Select **Install Now** to confirm and create the new
   partition table and install everything. Confirm location
   (time zone), first non-root user name (`ponder`) and
   computer name (`lexicon`).
1. Once it's done, select **Restart**
   (remove install media and hit `Enter`).

### First boot & MOK management

The first time the system boots, after installing for the
first time, the **MOK management** blue menu appears.
This is the point where
[the user installs Ubuntu on a new system](https://wiki.ubuntu.com/UEFI/SecureBoot#The_user_installs_Ubuntu_on_a_new_system)
and the correct option to choose is **Enroll MOK**.
I was midly confused, hadn't read the documentation carefully
before hand, and didn't know which option to chose. After a few
seconds of inactivity, the menu time out and the system boots.

Once the system has started, 

```
# dmesg | egrep -i 'secure|mok|boot'
[    0.000000] Command line: BOOT_IMAGE=/@/boot/vmlinuz-5.15.0-40-generic root=UUID=a61cf166-64e3-48ce-be02-07981ef82e33 ro rootflags=subvol=@ quiet splash vt.handoff=7
[    0.000000] efi: ACPI=0x40cbe000 ACPI 2.0=0x40cbe014 TPMFinalLog=0x40cc8000 SMBIOS=0x41568000 SMBIOS 3.0=0x41567000 MEMATTR=0x34ac0298 ESRT=0x351e3d18 MOKvar=0x3111c000 RNG=0x4151df18 TPMEventLog=0x310f5018 
[    0.000000] secureboot: Secure boot enabled
[    0.000000] Kernel is locked down from EFI Secure Boot mode; see man kernel_lockdown.7
[    0.018245] secureboot: Secure boot enabled
[    0.069027] smpboot: Allowing 4 CPUs, 0 hotplug CPUs
[    0.069056] Booting paravirtualized kernel on bare hardware
[    0.069204] Kernel command line: BOOT_IMAGE=/@/boot/vmlinuz-5.15.0-40-generic root=UUID=a61cf166-64e3-48ce-be02-07981ef82e33 ro rootflags=subvol=@ quiet splash vt.handoff=7
[    0.069250] Unknown kernel command line parameters "splash BOOT_IMAGE=/@/boot/vmlinuz-5.15.0-40-generic", will be passed to user space.
[    0.150970] smpboot: Estimated ratio of average max frequency by base frequency (times 1024): 1399
[    0.150970] smpboot: CPU0: 11th Gen Intel(R) Core(TM) i3-1115G4 @ 3.00GHz (family: 0x6, model: 0x8c, stepping: 0x1)
[    0.150970] x86: Booting SMP configuration:
[    0.151228] smpboot: Max logical packages: 1
[    0.151228] smpboot: Total of 4 processors activated (23961.60 BogoMIPS)
[    0.659056] ACPI: \_SB_.PC00.LPCB.H_EC: Boot DSDT EC used to handle transactions
[    1.003909] ACPI: \_SB_.PC00.LPCB.H_EC: Boot DSDT EC initialization complete
[    1.003960] pci 0000:00:02.0: vgaarb: setting as boot VGA device
[    1.292708] smpboot: Estimated ratio of average max frequency by base frequency (times 1024): 1399
[    1.305119] efifb: showing boot graphics
[    1.339073] Loaded X.509 cert 'Canonical Ltd. Secure Boot Signing: 61482aa2830d0ab2ad5af10b7250da9033ddcef0'
[    1.341583] integrity: Loading X.509 certificate: UEFI:MokListRT (MOKvar table)
[    1.403263]     BOOT_IMAGE=/@/boot/vmlinuz-5.15.0-40-generic
[    4.704413] systemd[1]: Condition check resulted in First Boot Complete being skipped.
[    5.271931] Bluetooth: hci0: Bootloader revision 0.4 build 0 week 30 2018
[    5.272963] Bluetooth: hci0: Secure boot is enabled
[    7.059956] Bluetooth: hci0: Waiting for device to boot
[    7.074922] Bluetooth: hci0: Device booted in 14646 usecs
```

Secure Boot is enabled and there is only one key enrolled:

```
# mokutil --sb-state
SecureBoot enabled

# mokutil --list-enrolled
[key 1]
SHA1 Fingerprint: 76:a0:92:06:58:00:bf:37:69:01:c3:72:cd:55:a9:0e:1f:de:d2:e0
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            b9:41:24:a0:18:2c:92:67
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=GB, ST=Isle of Man, L=Douglas, O=Canonical Ltd., CN=Canonical Ltd. Master Certificate Authority
        Validity
            Not Before: Apr 12 11:12:51 2012 GMT
            Not After : Apr 11 11:12:51 2042 GMT
        Subject: C=GB, ST=Isle of Man, L=Douglas, O=Canonical Ltd., CN=Canonical Ltd. Master Certificate Authority
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:bf:5b:3a:16:74:ee:21:5d:ae:61:ed:9d:56:ac:
                    bd:de:de:72:f3:dd:7e:2d:4c:62:0f:ac:c0:6d:48:
                    08:11:cf:8d:8b:fb:61:1f:27:cc:11:6e:d9:55:3d:
                    39:54:eb:40:3b:b1:bb:e2:85:34:79:ca:f7:7b:bf:
                    ba:7a:c8:10:2d:19:7d:ad:59:cf:a6:d4:e9:4e:0f:
                    da:ae:52:ea:4c:9e:90:ce:c6:99:0d:4e:67:65:78:
                    5d:f9:d1:d5:38:4a:4a:7a:8f:93:9c:7f:1a:a3:85:
                    db:ce:fa:8b:f7:c2:a2:21:2d:9b:54:41:35:10:57:
                    13:8d:6c:bc:29:06:50:4a:7e:ea:99:a9:68:a7:3b:
                    c7:07:1b:32:9e:a0:19:87:0e:79:bb:68:99:2d:7e:
                    93:52:e5:f6:eb:c9:9b:f9:2b:ed:b8:68:49:bc:d9:
                    95:50:40:5b:c5:b2:71:aa:eb:5c:57:de:71:f9:40:
                    0a:dd:5b:ac:1e:84:2d:50:1a:52:d6:e1:f3:6b:6e:
                    90:64:4f:5b:b4:eb:20:e4:61:10:da:5a:f0:ea:e4:
                    42:d7:01:c4:fe:21:1f:d9:b9:c0:54:95:42:81:52:
                    72:1f:49:64:7a:c8:6c:24:f1:08:70:0b:4d:a5:a0:
                    32:d1:a0:1c:57:a8:4d:e3:af:a5:8e:05:05:3e:10:
                    43:a1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                AD:91:99:0B:C2:2A:B1:F5:17:04:8C:23:B6:65:5A:26:8E:34:5A:63
            X509v3 Authority Key Identifier: 
                AD:91:99:0B:C2:2A:B1:F5:17:04:8C:23:B6:65:5A:26:8E:34:5A:63
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: 
                Digital Signature, Certificate Sign, CRL Sign
            X509v3 CRL Distribution Points: 
                Full Name:
                  URI:http://www.canonical.com/secure-boot-master-ca.crl
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        3f:7d:f6:76:a5:b3:83:b4:2b:7a:d0:6d:52:1a:03:83:c4:12:
        a7:50:9c:47:92:cc:c0:94:77:82:d2:ae:57:b3:99:04:f5:32:
        3a:c6:55:1d:07:db:12:a9:56:fa:d8:d4:76:20:eb:e4:c3:51:
        db:9a:5c:9c:92:3f:18:73:da:94:6a:a1:99:38:8c:a4:88:6d:
        c1:fc:39:71:d0:74:76:16:03:3e:56:23:35:d5:55:47:5b:1a:
        1d:41:c2:d3:12:4c:dc:ff:ae:0a:92:9c:62:0a:17:01:9c:73:
        e0:5e:b1:fd:bc:d6:b5:19:11:7a:7e:cd:3e:03:7e:66:db:5b:
        a8:c9:39:48:51:ff:53:e1:9c:31:53:91:1b:3b:10:75:03:17:
        ba:e6:81:02:80:94:70:4c:46:b7:94:b0:3d:15:cd:1f:8e:02:
        e0:68:02:8f:fb:f9:47:1d:7d:a2:01:c6:07:51:c4:9a:cc:ed:
        dd:cf:a3:5d:ed:92:bb:be:d1:fd:e6:ec:1f:33:51:73:04:be:
        3c:72:b0:7d:08:f8:01:ff:98:7d:cb:9c:e0:69:39:77:25:47:
        71:88:b1:8d:27:a5:2e:a8:f7:3f:5f:80:69:97:3e:a9:f4:99:
        14:db:ce:03:0e:0b:66:c4:1c:6d:bd:b8:27:77:c1:42:94:bd:
        fc:6a:0a:bc
```

As is tytpically the case, the system required installing a few
updates. After doing this, rebooting was required again.
After rebooting, MOK management was not offered.
No drivers seem to be missing, so everything seems to be good.

For a moment I thought I'd need to
[set up a new MOK password](https://forums.linuxmint.com/viewtopic.php?t=274365)
(`update-secureboot-policy --enroll-key`) but didn't need to.
I still feel like I don't *really* understand,
and should read more about,
[what exactly is MOK in Linux for?](https://unix.stackexchange.com/questions/535434/what-exactly-is-mok-in-linux-for)

## Ubuntu Server 22.04

With the Secure Boot setup out of the way, it was soon time to
install
[Ubuntu Server 22.04](https://cdimage.ubuntu.com/ubuntu-server/jammy/daily-live/pending/)
as the final, permanent system to run this server.

The installation process is a bit different:

1. Boot from USB, with the option to **Try or Install Ubuntu**
   * Wait for the `cloud-init` scripts to do finish (a few minutes).
1. Select language (English) and then **Install Ubuntu**
   * Update to the new installer.
1. Select keyboard layout.
1. Select Type of install: **Ubuntu Server (not minimized)**
1. Configure network interfaces:
   * Setup `enp89s0 eth` with **192.168.0.121**
   * Leave `wlo1 wlan` *not connected*
1. For **Storage configuration** select **Custom storage layout**
   * Use partition 1 (512 MB) as **EFI System Partition** mounted on `/boot/efi`
   * Use partition 2 (50,000 MB) **btrfs** without mount point (alrady used for [Ubuntu Studio](#ubuntu-desktop-2204))
   * Use partition 3 (50,000 MB) **btrfs** mounted on `/`
   * Use partition 4 (remainder) **btrfs** mounted on `/home`
     (**not** formatting)
1. Select **Install Now** to confirm the partition selection.
1. Confirm location (time zone), first non-root user name
   (`ponder`) and computer name (`lexicon`).
1. Under **SSH Setup** select
   * Install OpenSSH server: **Yes**
   * Import SSH identity: **No**
1. Select Featured server snaps: **none**
1. Once it's done, select **Restart**
   (remove install media and hit `Enter`).

At this point both Ubuntu desktop and server are installed.
The newly installed grub only boots the server installation,
which is fine because there is no need for the desktop anymore.

### Tweak Bash prompt

Tweak Bash prompt for `root` to make user name
<span style="color: red">red</span>,
host name
<span style="color: blue">blue</span>
and path
<span style="color: green">green</span>;
with this in `.bashrc`:

```bash
if [ "$color_prompt" = yes ]; then
    PS1='\[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\[\033[01;31m\]\u@\[\033[01;34m\]\h\[\033[00m\] \[\033[01;32m\]\w \$\[\033[00m\] '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h \w \$ '
fi
```

Other users' Bash prompt is left as default, which renders all
green. The idea is that `root`'s prompt is visually different,
to remind me that *with great power comes great responsibility*.

### Add local hosts to `/etc/hosts`

This being the latest system added to the network, its
`/etc/hosts` will contain all the other systems' LAN IPs:

```
# cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 lexicon

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Google (forced) Safe Search
# https://support.google.com/websearch/answer/186669
216.239.38.120 www.google.com #forcesafesearch
216.239.38.120 www.google.ch  #forcesafesearch

# K8s tests.
192.168.6.1     k8s.example.com
192.168.6.1     webapp.k8s.example.com

# Ethernet
10.0.0.2        rapture-lan
10.0.0.3        computer-lan
10.0.0.4        pi-f1-lan
10.0.0.5        smart-computer-lan
10.0.0.6        lexicon-lan
192.168.0.2     rapture
192.168.0.3     computer
192.168.0.148   pi-f1
192.168.0.8     smart-computer-lan

# Wi-Fi
192.168.0.12    pi3a
192.168.0.41    pi-f1-w
192.168.0.101   pi-z1
192.168.0.95    smart-computer
192.168.0.95    smart-computer-wlan
192.168.0.143   lexicon-wlan
192.168.0.216   pi-z2

# UniFi APs
unify-ap-lr     192.168.0.143
unify-ap-lite   192.168.0.69
```

### Enable NTP 

I like to have all machines on the same clock,
and for this there's nothing like good old NTP:

```
# apt install ntp ntpdate -y
```

### Setup SSH

Since this server would be reachable by SSH from outside the LAN,
it is imperative that **no** password *ever* will work on it.

Instead, only a few trusted certificates will be able to log in,
i.e. those added to `.ssh/authorized_keys` for each user.

#### Add authorized keys to SSH authentication

In order to disable password authentication, first add a few
trusted keys to `/root/.ssh/authorized_keys` and (optionally)
a few [more / other] to `/home/ponder/.ssh/authorized_keys`

#### Disable SSH password authentication

```
# vi /etc/ssh/sshd_config
PasswordAuthentication no
PermitEmptyPasswords no
# systemctl restart ssh
# systemctl restart sshd
```

**Note:** the reason for doing this is obvious after looking at
`/var/log/auth.log` once the SSH port has been exposed externally
for a while. Before doing this, there were **100,240** failed attempts to ssh in as `root` from **2,765 IPs**.

#### Setup Fail2Ban

On top of disabling password authentication, the SSH server will
be less busy is those pesky *bad actors* are blocked from
reaching it's port. To do this, install
[fail2ban](https://github.com/fail2ban/fail2ban).

[How to install fail2ban on Ubuntu Server 22.04: Jammy Jellyfish](https://www.techrepublic.com/article/install-fail2ban-ubuntu-server/)
explains this in more details; install with `apt` and setup:

```
# apt-get install fail2ban -y
# systemctl enable --now fail2ban
```

Within *seconds* a few IPs are banned already:

```
# iptables -L
...
Chain f2b-sshd (1 references)
target     prot opt source               destination
REJECT     all  --  45.118.145.178       anywhere             reject-with icmp-port-unreachable
REJECT     all  --  207.200.202.35.bc.googleusercontent.com  anywhere             reject-with icmp-port-unreachable
REJECT     all  --  68.183.235.43        anywhere             reject-with icmp-port-unreachable
REJECT     all  --  138.68.72.245        anywhere             reject-with icmp-port-unreachable
REJECT     all  --  207.154.220.120      anywhere             reject-with icmp-port-unreachable
RETURN     all  --  anywhere             anywhere
```

I like to spice it up to make a little more *trigger-happy*:

```
# vi /etc/fail2ban/jail.conf
# "bantime" is the number of seconds that a host is banned.
bantime  = 1000m
# A host is banned if it has generated "maxretry" during the last "findtime"
# seconds.
findtime  = 100m
# "maxretry" is the number of failures before a host get banned.
maxretry = 3
# "bantime.increment" allows to use database for searching of previously banned ip's to increase a 
# default ban time using special formula, default it is banTime * 1, 2, 4, 8, 16, 32...
bantime.increment = true
# systemctl restart fail2ban
```

Fail2ban can be setup for other services, see Gitea's guide to
[Fail2ban setup to block users after failed login attempts](https://docs.gitea.com/next/administration/fail2ban-setup)

Other resources for future consideration (GitHub repositories):

*  [awesome-selfhosted/awesome-selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted)
*  [kahun/awesome-sysadmin](https://github.com/kahun/awesome-sysadmin)
*  [pluja/awesome-privacy](https://github.com/pluja/awesome-privacy)

### Multiple Static IPs on LAN

There are 2 physical Local Area Networks in this environment,
a wire network connected directly to the router, and a wireless
network with a
[UniFi AC Long-Range](https://store.ui.com/us/en/products/unifi-ac-lr)
access point that *bridges* both networks.

This has the side effect that using IP addresses in the DHCP
range (leased by the router) may lead to data flowing through
the wireless network *despite* the hosts being physically
connected through the wired network.

To avoid this, LAN interfaces on all systems (that have them)
are assigned a *second* IP address on a different range, so
the traffic between them *can't jump* over the wireless network.

During installation, network setup defaulted to DHCP on Ethernet
and *no* wifi setup:

*  `/etc/netplan/00-installer-config.yaml`
    ```yaml
    # This is the network config written by 'subiquity'
    network:
    ethernets:
        enp89s0:
        dhcp4: true
    version: 2
    ```
*  `/etc/netplan/00-installer-config-wifi.yaml`
    ```yaml
    # This is the network config written by 'subiquity'
    network:
    version: 2
    wifis: {}
    ```

This lead to the system getting a leased IP in the
192.168.0.0/24 range and, most importantly,
the relevant DNS servers:

```
# ip a | grep enp89s0
2: enp89s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 192.168.0.121/24 metric 100 brd 192.168.0.255 scope global dynamic enp89s0

# resolvectl status
Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (enp89s0)
    Current Scopes: DNS
         Protocols: +DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 62.2.24.158
       DNS Servers: 62.2.24.158 62.2.17.61
        DNS Domain: v.cablecom.net

Link 3 (wlo1)
Current Scopes: none
     Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
```

To setup both addresses as static, need to grab the DNS servers
and create a new netplan configuration in
`/etc/netplan/00-installer-config.yaml`

```yaml
# Dual static IP on LAN, nothing else.
network:
  version: 2
  renderer: networkd
  ethernets:
    enp89s0:
      dhcp4: no
      dhcp6: no
      # Ser IP address & subnet mask
      addresses: [ 10.0.0.6/24, 192.168.0.6/24 ]
      # Set default gateway
      routes:
       - to: default
         via: 192.168.0.1
      # Set DNS name servers
      nameservers:
        addresses: [62.2.24.158, 62.2.17.61]
```

```
# netplan apply
# ip a | grep enp89s0
2: enp89s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    inet 10.0.0.6/24 brd 10.0.0.255 scope global enp89s0
    inet 192.168.0.6/24 brd 192.168.0.255 scope global enp89s0
```

### Weekly btrfs scrub

To keep BTRFS file systems healthy, it is recommended to
[run a weekly scrub](http://marc.merlins.org/perso/btrfs/post_2014-03-19_Btrfs-Tips_-Btrfs-Scrub-and-Btrfs-Filesystem-Repair.html)
to check everything for consistency. For this, I run
[the script]()
from crontab every Saturday morning, early enough that it will
be done by the time anyone wakes up.

```
# crontab -l | grep btrfs
# m h  dom mon dow   command
50 5 * * 6 /usr/local/bin/btrfs-scrub-all
```

The whole process takes less than 10 minutes with a 2TB NVMe SSD:

![Disk I/O and SSD temperatures chart show btrfs scrub taking less than 10 minutes, maxing out NVMe SSD bandwidth]({{ media }}/lexicon-btrfs-scrub-grafana.png)

## Conclusion

I quite like this hardware!

Secure Boot was a little confusing and intimidating at first,
now it's just a little confusing but at least not so scary.

With an Intel NUC Mini PC you get *literally* and Intel architecture (`x84_64`) and familiar hardware without much
in the way of platform-specific quirks.

It does come at a cost though, this system costed nearly $600
which is more than triple the cost of a modest Raspberry Pi 4
setup. On the other hand, Raspberry Pi computers are good
*up to a point* and past that an Intel NUC offers a really good
*power to size* ratio, packing quite some compute power in a
still quite small package.

## Future Updates

Watch this space for links to future posts!

*  **2022-10-12** [Crucial MX500 SSD found problematic]({{ site.baseurl }}/2022/10/12/crucial-mx500-ssd-found-problematic.html)
*  **2023-09-16** [Migrating a Plex Media Server to Kubernetes]({{ site.baseurl }}/2023/09/16/migrating-a-plex-media-server-to-kubernetes.html)
