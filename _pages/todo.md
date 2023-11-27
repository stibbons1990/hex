---
title: TODO
permalink: /todo/
---

This a loose collections of things or ideas to try.

Encrypt portable external disks
- https://opensource.com/article/21/3/encryption-luks
- https://www.jwillikers.com/encrypt-an-external-disk-on-linux
- https://www.wikihow.com/Encrypt-an-External-Hard-Drive-on-Linux
- https://linux.fernandocejas.com/docs/how-to/encrypt-an-external-hard-drive
- https://linuxhint.com/encrypt-data-usb-linux/
- https://superuser.com/questions/1602752/encrypt-an-external-hard-drive-with-readwrite-access-on-both-windows-and-linux
- https://forums.linuxmint.com/viewtopic.php?t=377430
- https://theawesomegarage.com/blog/encrypt-external-hard-drives-with-linux
- https://proton.me/blog/usb-encryption
- https://linux.tips/tutorials/how-to-encrypt-a-usb-drive-on-linux-operating-system


Make a script to watch out for `acme` jobs in K8s
to redirect port 80 to them with `iptables`.

```
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port ____
```

Kiosk browser + Grafana for physical display on lexicon
- Maybe run X + single-command in .xsession (pi-z2?)
- Can't power display off/on with vbetool dpms because can't disable Secure Boot 
[1](https://access.redhat.com/solutions/6969947);
remounting /dev won't help 
[2](https://superuser.com/questions/1555396/trouble-with-shutting-down-screen-in-ubuntu-server)

[RetroDECK](http://retrodeck.net)

Flameshot (screenshots app)

Automated the boring stuff with Python

Godot Engine? Cocos2d?

Find good, child-friendly article about inclusive language

Try Mass Effect with mods, e.g. MEUITM 2.

[Monigote Fantasy](https://bitbrosgames.itch.io/monigote-fantasy)
