---
date: 2023-09-23
categories:
 - steam
 - proton
 - videogames
 - ea
 - bioware
title: "Getting started with Dragon Age: Origins - Ultimate Edition"
---

[Dragon Age: Origins - Ultimate Edition](https://store.steampowered.com/app/47810/Dragon_Age_Origins__Ultimate_Edition/)
was on sale since last weekend, and today I *fell for it*.

Before purchasing the game, I checked all recent
[Proton reports for Dragon Age: Origins - Ultimate Edition](https://www.protondb.com/app/47810),
because *one does not simply* run Windows games on Linux.
At least, not *always* without a little tinkering.

<!-- more --> 

Among all the recent recommendations, this one worked for me:

``` console
__GLX_VENDOR_LIBRARY_NAME=nvidia PROTON_FORCE_LARGE_ADDRESS_AWARE=1 RADV_TEX_ANISO=16 PROTON_USE_D9VK=1 gamemoderun %command%
```

Adding `__NV_PRIME_RENDER_OFFLOAD=1` caused the game to get
stuck in the splash screen, so I had to remove that.

Although I'm not sure it helped, I did try installing
NVidia's Physx library:

``` console
$ winetricks physx
```

The installation seems to have been made only for the default
`WINEPREFIX` in `$HOME` while the game uses the one in
`$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx` so
the correct command would be

``` console
$ WINEPREFIX=$HOME/.local/share/Steam/steamapps/compatdata/47810/pfx winetricks physx
```

Also, as reported by
[thehoagie](https://www.protondb.com/users/597403899),

> The game tries to use a launcher. Bypass it because it
> doesn't work. Change the line in this xml file:

``` console
$ vi "${HOME}/.local/share/Steam/steamapps/common/Dragon Age Ultimate Edition/data/DAOriginsLauncher.xml"
```

and change line **247** from

``` xml
<xmlbutton name="play" action="condition" value="FirstRunCheck">
```

to

``` xml
<xmlbutton name="play" action="execute" file="${BINARIES_DIR}\DAOrigins.exe" autoquit="true">
```

With that, the game finally launches. There is a pop-up dialog
about DLCs needing to be activated, which is one of those
things that EA has broken by not maintaining things working.

Years ago there was an option in Steam to obtain the "CD keys",
but it appears that option is no longer feasible and there is
no clear workaround that will work in 2023, except getting the
games from GOG:

*  [What does the "Dragon Age: Origins - Ultimate Edition DLC CD Key" do?](https://steamcommunity.com/app/47810/discussions/0/558752450420490163)
*  [How to get all the DLC](https://steamcommunity.com/app/47810/discussions/0/1698300679774459377/)
*  [Downloadable content (Origins)](https://dragonage.fandom.com/wiki/Downloadable_content_(Origins))
