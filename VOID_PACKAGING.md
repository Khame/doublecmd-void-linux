# Void Linux packaging: doublecmd maintainer notes

This repository holds the install docs and release notes. The packaging source
is not here. The single copy is `srcpkgs/doublecmd/` in
[Khame/void-packages](https://github.com/Khame/void-packages).

Users should read [README.md](README.md) instead.

## Where the files live

| Path (in `Khame/void-packages`) | What it is |
|---|---|
| `srcpkgs/doublecmd/template` | the `xbps-src` template: version, checksum, dependencies, `do_build`, `do_install` |
| `srcpkgs/doublecmd/patches/0001-0011` | the toolchain patches |
| `srcpkgs/doublecmd/PATCHES.md` | per-patch reasoning and drop conditions |
| `srcpkgs/doublecmd/INSTALL.msg` | shown by `xbps-install`: Wayland and the two harmless console lines |
| `docs/` | signed packages, `x86_64-repodata`, the public key and the landing page |

## About the template

No `build_style`. Double Commander is built by its own `build.sh`, which calls
`lazbuild` per component and plugin; the template exports
`lazbuild="/usr/bin/lazbuild --lazarusdir=/usr/lib/lazarus"` and runs
`./build.sh release qt5`.

Why `--lazarusdir`: Void's `lazarus` ships no `environmentoptions.xml`, so
`lazbuild` cannot find its own data directory and aborts with
`invalid Lazarus directory ""`. Why the export works at all: patch 0001
stops `build.sh` from overwriting the variable.

Why Qt5 and not Qt6: Void has no `qt6pas`. The Lazarus package is built with
`LCL_PLATFORM=qt5` and `qt5pas` exists as a runtime package, so Qt5 is the
widgetset that is actually supportable here.

`nopie=yes` because FPC emits non-PIE executables.

The install layout mirrors upstream's `install/linux/install.sh` with Void's
paths: the binary and its data in `/usr/lib/doublecmd` (Void has no lib64),
translations, pixmaps, highlighters and docs in `/usr/share/doublecmd`, with
symlinks from the lib directory back into share, exactly like upstream's own
packages. `/usr/bin/doublecmd` is a symlink. Only the built plugin files and
their language data are installed, not the plugin sources.

Dependencies:

- `hostmakedepends="fpc lazarus"` — the compilers, which run on the build host.
- `makedepends="qt5pas-devel qt5-tools-devel qt5-svg-devel openssl-devel"` —
  the widgetset bindings and the SSL library DC links against.
- `depends="qt5pas qt5-wayland hicolor-icon-theme"`. `qt5-wayland` is a hard
  dependency on purpose: Void templates have no `optdepends` variable (it is
  silently ignored), and without the plugin a Wayland session without
  XWayland cannot start the program at all.

## Building locally

```sh
git clone --depth=1 https://github.com/void-linux/void-packages.git
cd void-packages
./xbps-src binary-bootstrap

git clone --depth=1 https://github.com/Khame/void-packages.git /tmp/khame-void-packages
cp -a /tmp/khame-void-packages/srcpkgs/doublecmd srcpkgs/

./xbps-src pkg doublecmd
sudo xbps-install --repository=hostdir/binpkgs doublecmd
```

The package ends up in `hostdir/binpkgs/`. There is no `check` phase; upstream
ships no test suite for the GUI program.

## Publishing a release

1. Compare the upstream release with `version=` in the template. If it has not
   changed, stop here.
2. Update `version=` and `checksum=`, and go through `PATCHES.md`: a new
   upstream version may make patches obsolete or need more API. Patch 0007
   removes the LCL version floor, so new 3.0 API would no longer be caught
   by the dependency check and would only show up as a compile error — a
   version bump needs a full compile, not just a checksum update.
3. Raise `revision=` when only the template or the patches change.
4. Run `xlint srcpkgs/doublecmd/template` and fix everything it reports.
5. Build, install and start it once. Watch the console output.
6. Sign the package you just built:

   ```sh
   pkg=$(ls -t hostdir/binpkgs/doublecmd-*.xbps | head -1)
   xbps-rindex -S --privkey="$HOME/.config/Khame-void-repo-private.pem" \
     --signedby="Khame <khame.dev@protonmail.com>" "$pkg"
   ```

7. Delete the old package and its signature from `docs/`, copy the new pair in
   and rebuild the index. Never pass a `doublecmd-*.xbps` glob to
   `xbps-rindex -a`; it also matches the old package and may register that
   version instead.

   ```sh
   rm -f docs/doublecmd-*.xbps docs/doublecmd-*.xbps.sig2
   cp "$pkg" "$pkg.sig2" docs/
   xbps-rindex -a "docs/$(basename "$pkg")"
   xbps-rindex -s --privkey="$HOME/.config/Khame-void-repo-private.pem" \
     --signedby="Khame <khame.dev@protonmail.com>" docs
   ```

8. Commit and push `Khame/void-packages`. GitHub Pages serves `docs/` from
   `main`, so the repository appears at
   `https://khame.github.io/void-packages/`.
9. Attach the same `.xbps` to a release in this repository and update the
   version numbers in `README.md`.

Packages are signed with the Khame repository RSA key. The private key stays
outside both repositories, for example at
`~/.config/Khame-void-repo-private.pem` with mode `600`. The public key is
copied to `docs/Khame-void-repo.pub` and `xbps-rindex -s` embeds it in the
index. Nobody has to put a `.pem` into `/var/db/xbps/keys/`: `xbps` imports the
key from the index on the first `xbps-install -S` and stores it as
`<fingerprint>.plist`, currently
`35:2b:06:7c:52:04:76:b1:4a:de:a0:8b:49:9c:e2:66.plist`. I never support
`allow-unsigned`.

`x86_64-repodata` is signed and carries the public key; `xbps-install -S`
verifies it and imports the key from it. Each package additionally has its
own `.sig2`. Only x86_64 is published.

## Things that bit me

- `vbin <file> <second arg>` takes a **file name**, not a directory:
  `_vbin` runs `vinstall "$file" 755 usr/bin "$targetfile"`. So
  `vbin doublecmd usr/lib/doublecmd` installed the binary as
  `/usr/bin/doublecmd`, and the `ln -sf` that followed replaced it with a
  dangling symlink. Use `vinstall doublecmd 755 usr/lib/doublecmd`.
- Several Pascal sources use CRLF. A patch written with LF endings fails with
  `1 out of 1 hunk FAILED`. Generate the diff from bytes, keeping CRLF, and
  verify with a test build; `patch` does not warn about the mismatch.
- After a failed build, `rm -rf masterdir-*/builddir/doublecmd-*` leaves the
  `.xbps-<pkgname>` marker, and the next run dies with
  `cannot access wrksrc`. The clean way is `./xbps-src clean doublecmd`.
  `-f` forces a stage, it does not re-extract the source.
- `xbps-src` refuses to run as root and needs no root at all: it uses
  `xbps-uunshare`. Do not wrap it in `sudo`.
- `optdepends` does not exist in Void. It is ignored without a warning and
  lands nowhere in the package metadata. Runtime options go into
  `INSTALL.msg`, or into `depends` when the program is broken without them.

There is no CI on these repositories. Releases are built by hand.

## Author and license

Khame ([@Khame](https://github.com/Khame)),
[khame.dev@protonmail.com](mailto:khame.dev@protonmail.com)

MIT for the material written here. Upstream Double Commander is
GPL-2.0-or-later.
