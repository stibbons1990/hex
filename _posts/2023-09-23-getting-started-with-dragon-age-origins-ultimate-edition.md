---
title:  "Getting started with Dragon Age: Origins - Ultimate Edition"
date:   2023-09-23 06:06:06 +0200
categories: steam proton videogames ea bioware
---

[Dragon Age: Origins - Ultimate Edition](https://store.steampowered.com/app/47810/Dragon_Age_Origins__Ultimate_Edition/)
was on sale since last weekend, and today I *fell for it*.

Before purchasing the game, I checked all recent
[Proton reports for Dragon Age: Origins - Ultimate Edition](https://www.protondb.com/app/47810),
because *one does not simply* run Windows games on Linux.
At least, not always without a little tinkering.

Among all the recent recommendations, this one worked for me:

```
__GLX_VENDOR_LIBRARY_NAME=nvidia PROTON_FORCE_LARGE_ADDRESS_AWARE=1 RADV_TEX_ANISO=16 PROTON_USE_D9VK=1 gamemoderun %command%
```

Adding `__NV_PRIME_RENDER_OFFLOAD=1` caused the game to get
stack in the splash screen, so I had to remove that.

Also, as reported by
[thehoagie](https://www.protondb.com/users/597403899),

> The game tries to use a launcher. Bypass it because it
> doesn't work. Change the line in this xml file:

```bash
vi "${HOME}/.local/share/Steam/steamapps/common/Dragon Age Ultimate Edition/data/DAOriginsLauncher.xml"
```

and change line **247** from

```xml
<xmlbutton name="play" action="condition" value="FirstRunCheck">
```

to

```xml
<xmlbutton name="play" action="execute" file="${BINARIES_DIR}\DAOrigins.exe" autoquit="true">
```
