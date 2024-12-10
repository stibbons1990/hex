---
date: 2023-10-15
categories:
 - linux
 - ubuntu
 - python
 - displaycal
title: Calibrating screen color with DisplayCAL
---

Calibrating screen color is *mostly optional* these days, if
you buy a good screen that comes out of factory with a good
calibration profile. Nonetheless, it is recommended to
re-calibrate every year or so to account for display ageing,
so a few years ago I purchased a
[Datacolor Spyder5 Express](https://www.bhphotovideo.com/lit_files/116444.pdf)
which is known to work on Linux, with
[Argyll Color Management System](https://www.argyllcms.com/).

<!-- more --> 

What doesn't always work so well is
[DisplayCal](https://displaycal.net/). In fact,
[the original project is dead](https://hub.displaycal.net/forums/topic/displaycal-is-dead-time-to-move-on-what-alternative-you-choosed/)
and
[was dropped from Ubuntu 20.04](https://wiki.ubuntu.com/FocalFossa/ReleaseNotes/UbuntuStudio)
but was still possible to build with python2.7 packages.
That is no longer possible in Ubuntu 22.04, but there is
a Python 3 fork:
[eoyilmaz/displaycal-py3](https://github.com/eoyilmaz/displaycal-py3)

## System requirements

Running DisplayCAL requires building its depedencies (through
PIP) so a few development packages are necessary, on top of
ArgyllCMS:

```
# apt install -y build-essential dbus libglib2.0-dev pkg-config libgtk-3-dev libxxf86vm-dev python3.10-venv argyll
```

## Python requirements

Building dependencies is necessary in either case, but unless
you *want* to build DisplayCAL itself from source, you can
skip directly to
[run the latest release](#run-latest-release).

```
$ git clone https://github.com/eoyilmaz/displaycal-py3
$ python -m venv ./displaycal_venv
$ source ./displaycal_venv/bin/activate
$ cd ./displaycal-py3/
$ pip install -r requirements.txt
  Running setup.py install for wxPython ... |
```

The instalation of `wxPython` takes a while, it will keep the
CPU busy and hot for several minutes (about 10 minutes on an
AMD Ryzen 5 1600X).

## Build from source

```
$ python -m build
$ pip install dist/DisplayCAL-3.9.*.whl
```

Sadly this fails to build, despite having installed all the
above requirements:

```
$ python -m build
* Creating venv isolated environment...
* Installing packages in isolated environment... (setuptools)
* Getting build dependencies for sdist...
Trying to get git version information...
Trying to get git information...
Generating __version__.py
Version 3.9.11
['egg_info']
*** /home/artist/Downloads/displaycal_venv/lib/python3.10/site-packages/pyproject_hooks/_in_process/_in_process.py egg_info
using distutils
/tmp/build-env-uooasz97/lib/python3.10/site-packages/setuptools/config/setupcfg.py:293: _DeprecatedConfig: Deprecated config in `setup.cfg`
!!

        ********************************************************************************
        The license_file parameter is deprecated, use license_files instead.

        By 2023-Oct-30, you need to update your project and remove deprecated calls
        or your builds will no longer be supported.

        See https://setuptools.pypa.io/en/latest/userguide/declarative_config.html for details.
        ********************************************************************************

!!
  parsed = self.parsers.get(option_name, lambda x: x)(value)
warning: no files found matching 'MANIFEST'
warning: no files found matching 'use-distutils'
warning: no files found matching '_in_process.py'
warning: no files found matching '_in_process.cfg'
warning: no files found matching 'dist/copyright'
warning: no files found matching 'DisplayCAL/ref/DCDM'
warning: no files found matching 'X'Y'Z'.icm'
warning: no files found matching 'DisplayCAL/ref/XYZ'
warning: no files found matching 'D50'
warning: no files found matching '(ICC'
warning: no files found matching 'PCS'
warning: no files found matching 'encoding).icm'
warning: no files found matching 'DisplayCAL/ref/XYZ'
warning: no files found matching 'D50.icm'
warning: no files found matching 'dist/net.displaycal.DisplayCAL.appdata.xml'
warning: no files found matching 'misc/displaycal.desktop'
warning: no files found matching 'misc/z-displaycal-apply-profiles.desktop'
warning: no previously-included files matching '*~' found anywhere in distribution
warning: no previously-included files matching '*.backup' found anywhere in distribution
warning: no previously-included files matching '*.bak' found anywhere in distribution
* Building sdist...
Trying to get git version information...
Trying to get git information...
Generating __version__.py
Version 3.9.11
['sdist', '--formats', 'gztar', '--dist-dir', '/home/artist/Downloads/displaycal-py3/dist/.tmp-70quu7wi']
*** /home/artist/Downloads/displaycal_venv/lib/python3.10/site-packages/pyproject_hooks/_in_process/_in_process.py sdist --formats gztar --dist-dir /home/artist/Downloads/displaycal-py3/dist/.tmp-70quu7wi
using distutils
/tmp/build-env-uooasz97/lib/python3.10/site-packages/setuptools/config/setupcfg.py:293: _DeprecatedConfig: Deprecated config in `setup.cfg`
!!

        ********************************************************************************
        The license_file parameter is deprecated, use license_files instead.

        By 2023-Oct-30, you need to update your project and remove deprecated calls
        or your builds will no longer be supported.

        See https://setuptools.pypa.io/en/latest/userguide/declarative_config.html for details.
        ********************************************************************************

!!
  parsed = self.parsers.get(option_name, lambda x: x)(value)
warning: no files found matching 'MANIFEST'
warning: no files found matching 'use-distutils'
warning: no files found matching '_in_process.py'
warning: no files found matching '_in_process.cfg'
warning: no files found matching 'DisplayCAL/ref/DCDM'
warning: no files found matching 'X'Y'Z'.icm'
warning: no files found matching 'DisplayCAL/ref/XYZ'
warning: no files found matching 'D50'
warning: no files found matching '(ICC'
warning: no files found matching 'PCS'
warning: no files found matching 'encoding).icm'
warning: no files found matching 'DisplayCAL/ref/XYZ'
warning: no files found matching 'D50.icm'
warning: no files found matching 'misc/displaycal.desktop'
warning: no files found matching 'misc/z-displaycal-apply-profiles.desktop'
warning: no previously-included files matching '*~' found anywhere in distribution
warning: no previously-included files matching '*.backup' found anywhere in distribution
warning: no previously-included files matching '*.bak' found anywhere in distribution
* Building wheel from sdist
* Creating venv isolated environment...
* Installing packages in isolated environment... (setuptools)
* Getting build dependencies for wheel...
['egg_info']
*** /home/artist/Downloads/displaycal_venv/lib/python3.10/site-packages/pyproject_hooks/_in_process/_in_process.py egg_info
using distutils
/tmp/build-env-po6qurti/lib/python3.10/site-packages/setuptools/config/setupcfg.py:293: _DeprecatedConfig: Deprecated config in `setup.cfg`
!!

        ********************************************************************************
        The license_file parameter is deprecated, use license_files instead.

        By 2023-Oct-30, you need to update your project and remove deprecated calls
        or your builds will no longer be supported.

        See https://setuptools.pypa.io/en/latest/userguide/declarative_config.html for details.
        ********************************************************************************

!!
  parsed = self.parsers.get(option_name, lambda x: x)(value)
warning: no files found matching 'MANIFEST'
warning: no files found matching 'use-distutils'
warning: no files found matching '_in_process.py'
warning: no files found matching '_in_process.cfg'
warning: no files found matching 'misc/displaycal.desktop'
warning: no files found matching 'misc/z-displaycal-apply-profiles.desktop'
warning: no previously-included files found matching 'misc/Argyll'
warning: no previously-included files found matching 'misc/*.rules'
warning: no previously-included files found matching 'misc/*.usermap'
warning: no previously-included files matching '*~' found anywhere in distribution
warning: no previously-included files matching '*.backup' found anywhere in distribution
warning: no previously-included files matching '*.bak' found anywhere in distribution
* Installing packages in isolated environment... (wheel)
* Building wheel...
['bdist_wheel', '--dist-dir', '/home/artist/Downloads/displaycal-py3/dist/.tmp-1xx40_u9']
*** /home/artist/Downloads/displaycal_venv/lib/python3.10/site-packages/pyproject_hooks/_in_process/_in_process.py bdist_wheel --dist-dir /home/artist/Downloads/displaycal-py3/dist/.tmp-1xx40_u9
using distutils
/tmp/build-env-po6qurti/lib/python3.10/site-packages/setuptools/config/setupcfg.py:293: _DeprecatedConfig: Deprecated config in `setup.cfg`
!!

        ********************************************************************************
        The license_file parameter is deprecated, use license_files instead.

        By 2023-Oct-30, you need to update your project and remove deprecated calls
        or your builds will no longer be supported.

        See https://setuptools.pypa.io/en/latest/userguide/declarative_config.html for details.
        ********************************************************************************

!!
  parsed = self.parsers.get(option_name, lambda x: x)(value)
DisplayCAL/RealDisplaySizeMM.c: In function ‘get_displays’:
DisplayCAL/RealDisplaySizeMM.c:871:61: warning: pointer targets in passing argument 11 of ‘XRRGetOutputProperty’ differ in signedness [-Wpointer-sign]
  871 |                                     &ret_type, &ret_format, &ret_len, &ret_togo, &atomv) == Success
      |                                                             ^~~~~~~~
      |                                                             |
      |                                                             long int *
In file included from DisplayCAL/RealDisplaySizeMM.c:33:
/usr/include/X11/extensions/Xrandr.h:340:38: note: expected ‘long unsigned int *’ but argument is of type ‘long int *’
  340 |                       unsigned long *nitems, unsigned long *bytes_after,
      |                       ~~~~~~~~~~~~~~~^~~~~~
DisplayCAL/RealDisplaySizeMM.c:871:71: warning: passing argument 12 of ‘XRRGetOutputProperty’ from incompatible pointer type [-Wincompatible-pointer-types]
  871 |                                     &ret_type, &ret_format, &ret_len, &ret_togo, &atomv) == Success
      |                                                                       ^~~~~~~~~
      |                                                                       |
      |                                                                       long unsigned int **
In file included from DisplayCAL/RealDisplaySizeMM.c:33:
/usr/include/X11/extensions/Xrandr.h:340:61: note: expected ‘long unsigned int *’ but argument is of type ‘long unsigned int **’
  340 |                       unsigned long *nitems, unsigned long *bytes_after,
      |                                              ~~~~~~~~~~~~~~~^~~~~~~~~~~
DisplayCAL/RealDisplaySizeMM.c:1036:53: warning: pointer targets in passing argument 10 of ‘XGetWindowProperty’ differ in signedness [-Wpointer-sign]
 1036 |                             &ret_type, &ret_format, &ret_len, &ret_togo, &atomv) == Success
      |                                                     ^~~~~~~~
      |                                                     |
      |                                                     long int *
In file included from DisplayCAL/RealDisplaySizeMM.c:27:
/usr/include/X11/Xlib.h:2696:5: note: expected ‘long unsigned int *’ but argument is of type ‘long int *’
 2696 |     unsigned long*      /* nitems_return */,
      |     ^~~~~~~~~~~~~~
DisplayCAL/RealDisplaySizeMM.c:1036:63: warning: pointer targets in passing argument 11 of ‘XGetWindowProperty’ differ in signedness [-Wpointer-sign]
 1036 |                             &ret_type, &ret_format, &ret_len, &ret_togo, &atomv) == Success
      |                                                               ^~~~~~~~~
      |                                                               |
      |                                                               long int *
In file included from DisplayCAL/RealDisplaySizeMM.c:27:
/usr/include/X11/Xlib.h:2697:5: note: expected ‘long unsigned int *’ but argument is of type ‘long int *’
 2697 |     unsigned long*      /* bytes_after_return */,
      |     ^~~~~~~~~~~~~~
DisplayCAL/RealDisplaySizeMM.c: In function ‘enumerate_displays’:
DisplayCAL/RealDisplaySizeMM.c:1364:56: warning: pointer targets in passing argument 1 of ‘PyBytes_FromStringAndSize’ differ in signedness [-Wpointer-sign]
 1364 |               (value = PyString_FromStringAndSize(dp[i]->edid, dp[i]->edid_len)) != NULL) {
      |                                                   ~~~~~^~~~~~
      |                                                        |
      |                                                        unsigned char *
In file included from /usr/include/python3.10/Python.h:82,
                 from DisplayCAL/RealDisplaySizeMM.c:1:
/usr/include/python3.10/bytesobject.h:34:50: note: expected ‘const char *’ but argument is of type ‘unsigned char *’
   34 | PyAPI_FUNC(PyObject *) PyBytes_FromStringAndSize(const char *, Py_ssize_t);
      |                                                  ^~~~~~~~~~~~
error: can't copy '/tmp/build-via-sdist-b6k1fhs3/DisplayCAL-3.9.11/DisplayCAL/../misc/displaycal.desktop': doesn't exist or not a regular file

ERROR Backend subprocess exited when trying to invoke build_wheel
```

Switching to the development branch (`git checkout develop`)
leads to the same build error, and
[other branches](https://github.com/eoyilmaz/displaycal-py3/branches)
don't seem likely to fix this either.

## Run latest release

Instead of building from source, you can simply run
[the latest release](https://github.com/eoyilmaz/displaycal-py3/releases/):

```
$ wget https://github.com/eoyilmaz/displaycal-py3/releases/download/3.9.11/DisplayCAL-3.9.11.tar.gz
$ tar xfz DisplayCAL-3.9.11.tar.gz
$ cd DisplayCAL-3.9.11/
$ ./DisplayCAL.pyw 
```

However, this *won't work* without first installing all the
above requirements, it will fail because `send2trash` is
missing and `wxPython` is too old:

```
$ ./DisplayCAL.pyw 
Acquired lock file: <DisplayCAL.main.AppLock object at 0x7fb42b873970>
DisplayCAL.pyw 3.9.11 2023-06-05T17:07:58Z
ubuntu 22.04 jammy x86_64
Python 3.10.12 (main, Jun 11 2023, 05:26:28) [GCC 11.4.0]
Faulthandler 
wxPython 4.0.7 gtk3 (phoenix) wxWidgets 3.0.5
Encoding: utf-8
File system encoding: utf-8
Loading /home/artist/.config/DisplayCAL/DisplayCAL.ini
listening
writing to lock file: port: 15411
/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/edid.py:45: Warning: No module named 'DisplayCAL.lib64.python310.RealDisplaySizeMM'
  warnings.warn(str(exception), Warning)
Traceback (most recent call last):
  File "/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/main.py", line 549, in main
    _main(module, name, app_lock_file_name)
  File "/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/main.py", line 534, in _main
    from DisplayCAL.display_cal import main
  File "/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/display_cal.py", line 45, in <module>
    from send2trash import send2trash
ModuleNotFoundError: No module named 'send2trash'
┌──────────────────────────────────────────────────────────────────────────────┐
│ Traceback (most recent call last):                                           │
│   File "/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/main.py", line   │
│ 549, in main                                                                 │
│     _main(module, name, app_lock_file_name)                                  │
│   File "/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/main.py", line   │
│ 534, in _main                                                                │
│     from DisplayCAL.display_cal import main                                  │
│   File "/home/artist/Downloads/DisplayCAL-3.9.11/DisplayCAL/display_cal.py", │
│ line 45, in <module>                                                         │
│     from send2trash import send2trash                                        │
│ ModuleNotFoundError: No module named 'send2trash'                            │
└──────────────────────────────────────────────────────────────────────────────┘

(DisplayCAL.pyw:943597): Gtk-WARNING **: 12:39:12.107: Theme directory places/128 of theme ubuntustudio-dark has no size field


(DisplayCAL.pyw:943597): Gtk-WARNING **: 12:39:12.107: Theme directory places/scalable of theme ubuntustudio-dark has no size field

Exiting DisplayCAL
Ran application exit handlers
```

To install the requirements, use only these two steps from
those required to
[build from source](#build-from-source),
*without* using a virtual environment:

```
$ cd ../displaycal-py3/
$ pip install -r requirements.txt
  Running setup.py install for wxPython ... |
```

Building `wxPython` takes a while, it will keep the CPU busy
and hot for several minutes (about 10 minutes on an AMD
Ryzen 5 1600X).

Once the required libraries are installed,
[the latest release](https://github.com/eoyilmaz/displaycal-py3/releases/) runs without problems:

```
$ ./DisplayCAL.pyw 
```

There may be several pop-up windows, which may not be
visible if the Spyder5 instrument is on the center of the
screen. Once those are dealt with, the calibration process
can be started.

## Loading the latest profile

[Redshift](http://jonls.dk/redshift/)
adjusts the color temperature of your screen according to
your surroundings. This may help your eyes hurt less if you
are working in front of the screen at night.

While Redshift is active, color profiles are replaced with
the darker, warmer tone. When Redshift is temporarily
disabled, you need to re-load the color profile to use it.

To do this, simply run `xcalib` with the latest ICC profile.
The one-line `xcalib-latest` makes this very easy:

```bash
#!/bin/sh
xcalib "$(ls -t $HOME/.local/share/icc/*.icc | head -1)"
```

Moreover, the point of having this simple command in a simple
Bash script is to run this script even *more* easily by
clicking on a Desktop launcher:

```ini
[Desktop Entry]
Name=Restore Color
Comment=Restore color calibratio profile
Exec=/home/artist/bin/xcalib-latest
Icon=/home/artist/Desktop/.icons/DisplayCAL.png
Type=Application
Terminal=false
StartupNotify=false
X-KDE-SubstituteUID=false
```

![Launcher icon for DisplayCAL](../media/2023-10-15-calibrating-screen-color-with-displaycal/DisplayCAL.png)

**Note:** Redshift must be disabled (or stopped) **before**
using this launcher; otherwise it will very quickly override
the color profile.