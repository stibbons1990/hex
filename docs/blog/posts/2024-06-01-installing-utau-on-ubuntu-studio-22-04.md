---
date: 2024-06-01
categories:
 - linux
 - kubernetes
 - docker
 - server
 - self-hosted
 - music
 - streaming
 - media
 - navidrome
title: Installing Utau on Ubuntu Studio 22.04
---

The young artist wanted to try
[Utau](https://es.wikipedia.org/wiki/Utau),
which as of May 24 was on version **v0.4.19のインストーラー修正版**.

<!-- more -->

Downloaded the installer
[utau0419cInstaller.zip](https://utau2008.xrea.jp/downloads/utau0419cInstaller.zip)
and followed more or less steps found in the (now defunct)
Utau forum at
[utaforum.net/threads/utau-in-ubuntu.620](https://utaforum.net/threads/utau-in-ubuntu.620/).

Run [PlayOnLinux](https://www.playonlinux.com/) with the `LANG`
set to Japanese, as required by Utau:

```
LANG=ja_JP.utf8 playonlinux
```

Created a **64-bit** environment running **Windows 10** and run
the installer, clicked on an **N** button because everything else
was unreadable. Eventually the installer finished, created a
shortcut for `utau.exe`.

Sadly, the result of running the program after installation was
only a UI full of unreadable *blocky* characters.

Tried to fix this by intalling `allfonts` following steps in
[activating Winetricks](https://xn--deepinenespaol-1nb.org/wiki/activar-winetricks-en-unidades-de-playonlinux/):

```
WINEPREFIX=~/.PlayOnLinux/wineprefix/UTAU winetricks allfonts
```

After this, running the problem with `playonlinux` was OK.

[Installing UTAU in Linux](https://ale4710.neocities.org/words/utau-on-linux/) are other instructions that might work (better),
possibly to be tested in the future.
