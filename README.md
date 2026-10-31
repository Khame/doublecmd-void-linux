# Double Commander for Void Linux

Double Commander packaged for Void Linux: the twin-panel file manager, its
plugins, translations and documentation. Built from source with the toolchain
Void actually ships.

## Why this exists

Void has no `doublecmd` package. The reason is the toolchain, not the program:
Double Commander 1.2.9 asks for FPC >= 3.2.2 and Lazarus >= 3.0, and Void ships
`fpc 3.2.0` with `lazarus 2.2.0`. The
[package request](https://github.com/void-linux/void-packages/issues/23964)
has been open since 2020 with no merged template.

This package closes the gap with eleven small patches that keep 1.2.9
buildable on the older compiler and widgetset. None of them changes what the
program does; what each one changes and when it can be dropped is documented
in `PATCHES.md` in the packaging repository.

It is a personal project, not an official Void package, and there is no
support promise behind it. Bug reports and patches are welcome anyway.

## Install

### Signed XBPS repository

```sh
# add the repository
sudo sh -c 'echo "repository=https://khame.github.io/void-packages/" > /etc/xbps.d/10-khame.conf'

# sync the index, answer Y when xbps asks to import the key
sudo xbps-install -S

sudo xbps-install doublecmd
```

The key sits inside the signed index, so xbps imports it without an extra
step. It ends up as
`/var/db/xbps/keys/35:2b:06:7c:52:04:76:b1:4a:de:a0:8b:49:9c:e2:66.plist`.
The same key is published as
[Khame-void-repo.pub](https://khame.github.io/void-packages/Khame-void-repo.pub)
if you want to compare it yourself.

Updates come with `sudo xbps-install -Su`. Only x86_64 is built.

### Release asset

Import the key first, then:

```sh
sudo xbps-install --binary-package-url https://github.com/Khame/doublecmd-void-linux/releases/download/v1.2.9/doublecmd-1.2.9_1.x86_64.xbps
```

This is for one install. Updates are manual. The release also has the `.sig2`
file and the public key, fingerprint
`35:2b:06:7c:52:04:76:b1:4a:de:a0:8b:49:9c:e2:66`.

### From source

The package lives in
[Khame/void-packages](https://github.com/Khame/void-packages), not here.

```sh
git clone --depth=1 https://github.com/void-linux/void-packages.git
cd void-packages
./xbps-src binary-bootstrap

git clone --depth=1 https://github.com/Khame/void-packages.git /tmp/khame-void-packages
cp -a /tmp/khame-void-packages/srcpkgs/doublecmd srcpkgs/

./xbps-src pkg doublecmd
sudo xbps-install --repository=hostdir/binpkgs doublecmd
```

Do this when you want to change the template or build another revision.

## What gets installed

The `doublecmd` binary in `/usr/lib/doublecmd/` with `/usr/bin/doublecmd`
pointing at it, the WCX/WDX/WFX/WLX/DSX plugins (zip, sevenzip, unrar, deb,
rpm, cpio, base64, audioinfo, ftp, samba, mplayer, locate), the Polish and
other translations, the syntax highlighters, the icon theme, the manual page,
the desktop entry and the polkit action for running as root.

The Qt5 widgetset is used, so the program depends on `qt5pas`. Wayland is
native through `qt5-wayland`; X11 sessions use xcb.

## Patches

Eleven patches, all forced by the older toolchain:

1. `build.sh` discards a preset `lazbuild`, which is how the template passes
   `--lazarusdir`; the patch makes the preset win.
2. `FS_IOC_GETFLAGS` is `$80086601`, too large for FPC 3.2.0's signed
   `TIOCtlRequest`. The patch keeps the bit pattern as a `LongInt`.
3. FPC 3.2.0 has no `StrToUIntDef`; the patch adds a local one.
4. and 10. `TSpeedButton.Alignment` needs Lazarus 3.0. The toolbar label
   draws left-aligned anyway, so that is the fixed value.
5. and 6. The bundled JCL and the sevenzip plugin declare `IInterface`
   methods as `winapi`, which FPC 3.2.0 on Linux rejects against the `cdecl`
   RTL declaration.
7. `doublecmd.lpi` requires LCL >= 3.0; the patch lowers the minimum to 2.2.
8. `TSynIniSyn.CommentTypes` needs Lazarus 3.0. The assignment is dropped, so
   `#` lines in INI files are not coloured as comments.
9. `TCustomGrid.TrySmoothScrollBy` needs Lazarus 3.0. The thumbnail view uses
   `ScrollBy` instead.
11. `TCustomImage.Images/ImageWidth/ImageIndex` need Lazarus 3.0. The HiDPI
   helper assigns the bitmap to `Picture` directly.

Full reasoning, verification notes and drop conditions:
[PATCHES.md](https://github.com/Khame/void-packages/blob/main/srcpkgs/doublecmd/PATCHES.md).
Nothing has been sent upstream, because upstream meets its own requirements.

## Requirements

Void Linux x86_64 with glibc. musl is not supported by Lazarus and was not
tried. Qt5 comes in through the package dependencies.

## Tested on

ASUS ROG Strix G513QM: Ryzen 9 5900HX, Radeon Vega plus RTX 3060 Mobile,
16 GB RAM. Void x86_64 with glibc, XBPS 0.60.7, runit, Plasma 6.7.5 on
Wayland. Double Commander 1.2.9 starts there natively on Wayland, browses,
packs and unpacks archives through the bundled plugins, and survives
suspend/resume.

## Repository layout

- `README.md`: this file
- `VOID_PACKAGING.md`: how the template is put together, how a release is built
- `LICENSE`: MIT for what I wrote here

The build files are not copied here. The single copy is
[`srcpkgs/doublecmd/`](https://github.com/Khame/void-packages/tree/main/srcpkgs/doublecmd).

## Upstream

Project: [doublecmd/doublecmd](https://github.com/doublecmd/doublecmd),
[homepage](https://doublecmd.sourceforge.io/)

Packaged version: `1.2.9_1` (upstream release tarball, not a git snapshot)

License: GPL-2.0-or-later

## Author

Khame ([@Khame](https://github.com/Khame)),
[khame.dev@protonmail.com](mailto:khame.dev@protonmail.com)
