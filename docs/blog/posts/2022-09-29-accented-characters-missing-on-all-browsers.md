---
date: 2022-09-29
categories:
 - linux
 - locales
 - chrome
 - firefox
title: Accented characters missing on all browsers
---

At some point all browsers in my PC started to refuse showing
accented characters (e.g. **è**) when the correct keys are pressed
(first **`** then **e**). This started happening before reinstalling the OS ([Ubuntu Studio 22.04](https://ubuntustudio.org/2022/04/ubuntu-studio-22-04-lts-released/)).
Reinstalling the system, which came about for other reasons,
did not help.

<!-- more --> 

This problem was specific to web browsers only, even though it
affected all of them equally. Typing accented characters continued
to work fine everywhere else, e.g. in Konsole and Libreoffice.
For a while the workaround was to type those characters in Konsole
and copy them over.

Soon enough, *I had enough* and started to search for a proper solution.

Running each browser from Konsole, or watching `~/.xsession-errors`,
Chrome and Firefox produced different warning messages each time
I tried to type an accented character:

``` console
$ tail -f ~/.xsession-errors
(google-chrome-stable:14941): Gdk-WARNING **: 15:19:37.785: gdk_window_set_user_time called on non-toplevel
** (firefox:2091768): WARNING **: 15:19:57.550: Error converting text from IM to UTF-8: Invalid byte sequence in conversion input
```

Searching for the warning from Chrome message, there are very few matching posts. [This one](https://unix.stackexchange.com/questions/180067/chrome-gdk-gdk-window-set-user-time-called-on-non-toplevel)
suggested to unsetting the `GTK_IM_MODULE` variable,
which I had set to `xim`:

``` console
$ echo $GTK_IM_MODULE
xim
$ unset GTK_IM_MODULE
```

This didn’t seem to help, at least not without restarting Chrome.
A permanent solution could be to either fix `xim` or prevent the
`GTK_IM_MODULE` variable from pointing to `xim` at all.

The variable seems to be set in
`/usr/share/im-config/data/80_kinput2.rc` which is part of the
`im-config` package.

To get a better idea of what `xim` was doing, I run
`im-config -a -s` (without changing anything). This suggested
I had manually selected `xim` via `~/.xinputrc` and indeed that
as the case, although this looks like the config was generated
rather than manually created by me:

``` console
$ cat .xinputrc 
# im-config(8) generated on Sun, 24 Apr 2016 07:53:00 +0200
run_im xim
# im-config signature: 29c43083ff25c566c8c3ff6659294e6c  -

$ ls -l .xinputrc
-rw-rw-r-- 1 coder coder 130 Apr 24  2016 .xinputrc
```

Considering this config dates back to a long long time ago,
I removed it and restarted the desktop to see if that would
help... and it did!

While search for related posts I also found
[a tangentially related hint](https://fpkanarias.blogspot.com/2016/07/que-le-ha-pasado-las-tildes-en-ubuntu.html)
to run `dpkg-reconfigure locales` and add `es_ES.UTF-8`
(and/or others as desired) as a supported locale/s for the system
(and regenerate them).

And then I remembered the System Notification Helper popup message
telling me that
[*Language support is incomplete, additional packages
are required*](https://unix.stackexchange.com/questions/421066/popup-language-support-is-incomplete-what-packages-does-it-want-to-install).
Since I was feeling generous and this only takes about 120 MB of
disk space, I just installed the missing packages:

``` console
# apt install $(check-language-support)
The following NEW packages will be installed:
  hunspell-en-au hunspell-en-ca hunspell-en-gb hunspell-en-us
  hunspell-en-za hyphen-en-ca hyphen-en-gb hyphen-en-us
  libreoffice-help-common libreoffice-help-en-gb
  libreoffice-help-en-us libreoffice-l10n-en-gb
  libreoffice-l10n-en-za mythes-en-au mythes-en-us
  thunderbird-locale-en-gb thunderbird-locale-en-us
```
