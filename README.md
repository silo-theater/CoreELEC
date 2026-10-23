# Silo-theater OS (CoreELEC fork)

This repository is the build system for the Silo-theater distribution. It is a
fork of [CoreELEC](https://coreelec.org) 22 (`coreelec-22`) that replaces Kodi
with [silo-shell](https://github.com/silo-theater/silo-theater), a Qt Quick
interface and player for [Silo](https://github.com/Silo-Server/silo-server)
servers, with Jellyfin as a fallback.

> [!NOTE]
> The changes below are proposed. Today this branch still builds stock
> CoreELEC 22 with Kodi.

## Target hardware

- **Reference box:** Ugoos AM6B+ (Amlogic S922X), using the
  `g12b_s922x_ugoos_am6` device trees from the `Amlogic-no` device.
- **Project and device:** `PROJECT=Amlogic-ce DEVICE=Amlogic-no`.
- Other Amlogic SoCs supported by CoreELEC 22 stay buildable. We do not change
  CoreELEC's device, DTB or bootloader layout, so more boxes can follow.

## What changes from CoreELEC

| Area | CoreELEC 22 today | Silo-theater |
| --- | --- | --- |
| Frontend | Kodi (`kodi.service`) | `silo-shell` package and systemd unit. Kodi stays installed as a fallback and test reference until silo-shell is stable, then is removed. |
| Graphics | GLES via `opengl-meson`, no display server | Unchanged. Adds Qt 6 (`qtbase`, `qtdeclarative`, `qtshadertools`) built for EGLFS on the Mali fbdev driver. |
| Video decode | `libamcodec`, `media_modules-aml` | Unchanged. silo-shell uses the same decode path Kodi uses. |
| Kernel | Amlogic vendor 5.15 (`linux-amlogic`) with `common_drivers` | Unchanged base, plus forward-ported kernel patches from the p3i fork. |
| Audio | ALSA, HDMI passthrough from Kodi | ALSA only for HDMI output. Compressed formats are sent bit-perfect as IEC 61937; no software mixer in the passthrough path. |
| CEC | libcec built without the Linux CEC API (`CEC_FRAMEWORK_SUPPORT="no"`) | To be decided on the AM6B+: libcec on the kernel CEC framework (`CONFIG_CEC_MESON_G12A_AO=y` is already enabled), or the current adapter. |
| Bluetooth | BlueZ, PipeWire available | BlueZ driven by silo-shell. Remotes first, then game controllers and headphones (PCM only). |

## Repositories

Package sources will point at the Silo-theater forks instead of
`github.com/CoreELEC/*`, so our patches have one home:

| Package | Repository |
| --- | --- |
| `linux` | [silo-theater/linux-amlogic](https://github.com/silo-theater/linux-amlogic) |
| `media_modules-aml` | [silo-theater/media_modules-aml](https://github.com/silo-theater/media_modules-aml) |
| `kodi` (fallback only) | [silo-theater/xbmc](https://github.com/silo-theater/xbmc) |
| `silo-shell` (new) | [silo-theater/silo-theater](https://github.com/silo-theater/silo-theater) |

Patches from pannal's p3i fork of CoreELEC are sorted into kernel, distro and
Kodi changes. Kernel and distro patches are forward-ported here. Kodi patches
are used only as a reference for the silo-shell player.

## Building

The build works the same as CoreELEC:

```sh
PROJECT=Amlogic-ce DEVICE=Amlogic-no ARCH=aarch64 make image
```

See [docs/development.md](docs/development.md) for the development setup
and release process, and [packages/README.md](packages/README.md) for the
package format.

## Roadmap

Milestones and status are tracked in the
[silo-theater README](https://github.com/silo-theater/silo-theater#milestones).

## Upstream

This fork tracks the `coreelec-22` branch of
[CoreELEC/CoreELEC](https://github.com/CoreELEC/CoreELEC). Issues with
Silo-theater changes belong in this project, not on the CoreELEC forum.

- [CONTRIBUTING.md](CONTRIBUTING.md): how to report issues and submit pull requests
- [STANDARDS.md](STANDARDS.md): coding standards for build scripts and package files

## License

CoreELEC original code is released under
[GPLv2](https://www.gnu.org/licenses/gpl-2.0.html).

## Copyright

As CoreELEC includes code from many upstream projects it includes many copyright owners. CoreELEC makes NO claim of copyright on any upstream code. Patches to upstream code have the same license as the upstream project, unless specified otherwise. For a complete copyright list please checkout the source code to examine license headers. Unless expressly stated otherwise all code submitted to the CoreELEC project (in any form) is licensed under [GPLv2](https://www.gnu.org/licenses/gpl-2.0.html). You are absolutely free to retain copyright. To retain copyright simply add a copyright header to each submitted code page. If you submit code that is not your own work it is your responsibility to place a header stating the copyright.
