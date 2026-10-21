# AGENTS.md

Guidance for AI agents working on the Silo-theater distribution. This
repository is a fork of CoreELEC 22 (`coreelec-22`) that will replace Kodi with
[silo-shell](https://github.com/silo-theater/silo-theater). See
[README.md](README.md) for what changes from CoreELEC, and the
[silo-theater README](https://github.com/silo-theater/silo-theater#milestones)
for milestones.

## Target

- Build with `PROJECT=Amlogic-ce DEVICE=Amlogic-no ARCH=aarch64 make image`.
- The reference box is the Ugoos AM6B+ (S922X, `g12b_s922x_ugoos_am6*` device
  trees). Every other device and SoC that CoreELEC 22 builds must keep
  building.
- Kernel: Amlogic vendor 5.15 (`linux-amlogic`) plus `common_drivers`. Video
  decode: `media_modules-aml` and `libamcodec`. Graphics: `opengl-meson`, no
  display server.

## How to change this repository

- **Add, don't edit.** Put Silo-theater work in new packages (for example
  `silo-shell`, Qt 6 packages) or in `projects/Amlogic-ce` overrides. Edit an
  existing upstream package only when there is no other way, and keep the
  edit small. This keeps weekly merges from `coreelec-22` cheap.
- **Pin our forks.** Package sources for `linux`, `media_modules-aml` and
  `kodi` are moving to the `silo-theater` forks. Pin them by commit hash, as
  CoreELEC pins its own. Bump them in a commit of their own (`<package>: bump package to
  <short hash>`).
- **Keep Kodi buildable.** Kodi stays in the image as a fallback and test
  reference until the silo-theater README says it has been removed.
- **p3i patches.** Kernel and distro patches from pannal's p3i fork are
  forward-ported here, one patch or topic per commit, crediting the original
  author. Kodi patches from p3i are not applied.
- **Upstream merges.** Merge `coreelec-22` with a merge commit; never rebase
  or force-push shared branches.

## Requirements to protect

- **Bit-perfect HDMI audio.** Do not add a sound server, mixer or resampler
  in front of the HDMI ALSA device. Bluetooth audio goes through a separate
  PCM path.
- **CEC.** CoreELEC 22 builds libcec with `CEC_FRAMEWORK_SUPPORT="no"`
  (no Linux CEC API), while the kernel enables `CONFIG_CEC_MESON_G12A_AO`.
  The choice between the two is open; do not change it without testing on
  the AM6B+ against a TV and an AVR.
- **Bluetooth.** BlueZ stays in the image. Remotes are the first priority,
  then game controllers, then headphones.

## Coding standards

Follow [STANDARDS.md](STANDARDS.md) for build scripts, device scripts and
`package.mk` files, and [packages/README.md](packages/README.md) for package
variables. In short:

- Build scripts: `#!/bin/bash`, `${VAR}` with braces, `$()` not backticks,
  `=` for string tests, ShellCheck clean.
- Device scripts: `#!/bin/sh`, POSIX only (busybox ash), ShellCheck with
  `--shell=sh`.
- Lines under 90 columns, except long dependency lists.

## Scope, comments and commits

- Make the smallest change that fully solves the task. Do not fix unrelated
  problems, reformat, or add speculative hardening.
- Comments explain what is not obvious. Do not describe how the change was
  developed.
- Commit subjects follow the existing style (`<package>: <change>`). Wrap the
  body at 76 columns and say why the change is needed.
- Keep one concern per pull request.

## Testing

- A change to the image must build for `Amlogic-ce` / `Amlogic-no` and boot
  on the AM6B+.
- Kernel, audio or display changes need evidence from the device: what was
  played, the HDMI mode, and the AVR reading for passthrough formats.

## Related repositories

| Repository | Role |
| --- | --- |
| [silo-theater/silo-theater](https://github.com/silo-theater/silo-theater) | `silo-shell` source, architecture and milestones |
| [silo-theater/linux-amlogic](https://github.com/silo-theater/linux-amlogic) | Amlogic 5.15 kernel |
| [silo-theater/media_modules-aml](https://github.com/silo-theater/media_modules-aml) | Amlogic decoder modules and firmware |
| [silo-theater/xbmc](https://github.com/silo-theater/xbmc) | CoreELEC's Kodi; fallback frontend and player reference |
