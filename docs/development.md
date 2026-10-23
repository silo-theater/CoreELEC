# Development and release guide

This guide covers setting up a Silo-theater development environment and
taking the distribution to a shippable release. It describes the state of
the repositories on 2026-10-22; update it as the steps below land.

## Where things stand

The build system can build stock CoreELEC 22 from our forks today.
Everything Silo-specific (Qt for EGLFS, `silo-shell`, branding, signing, an
update channel, CI) still has to be created.

| Repo | Branch | State |
| --- | --- | --- |
| [silo-theater/CoreELEC](https://github.com/silo-theater/CoreELEC) | `coreelec-22` | Stock CoreELEC 22.0 rc1 (`OS_STAGE="rc"`). `linux`, `media_modules-aml` and `kodi` fetch from the silo-theater forks. No CI, no Silo packages. |
| [silo-theater/linux-amlogic](https://github.com/silo-theater/linux-amlogic) | `5.15.196_20260306` | Pinned at `f231c781`. |
| [silo-theater/media_modules-aml](https://github.com/silo-theater/media_modules-aml) | `5.15.196_20260225` | Pinned at `c15aebb1`. |
| [silo-theater/xbmc](https://github.com/silo-theater/xbmc) | `aml-5.15.196-22.0` | Pinned at `887e3b74`. Fallback frontend and source of the player code. |
| [silo-theater/silo-theater](https://github.com/silo-theater/silo-theater) | `main` | README, AGENTS.md and LICENSE only. No code yet. |

Open items in the READMEs:

- The forks are already pinned, so that milestone 1 task is done.
- The silo-theater LICENSE file is AGPL-3.0 while the plan says
  GPL-2.0-or-later. Settle this before `silo-shell` code lands.
- The AM6B+ uses the `g12b_s922x_ugoos_am6b` device tree, not the `am6` one
  named in this repository's README.

The path to a release:

1. Build and boot stock CoreELEC 22 from our forks on the AM6B+.
2. Put that build in CI so every push produces an image.
3. Add Qt 6 for EGLFS and a `silo-shell` package, and iterate on the box.
4. Rebrand the distribution and give it its own version scheme.
5. Sign releases and stand up an update channel the box checks.
6. Run the hardware test matrix and ship.

## 1. Set up a build host

Use an x86-64 Linux machine (or VM) and build inside CoreELEC's own Docker
image, so every developer and CI runner has the same toolchain.

| Resource | Minimum | Why |
| --- | --- | --- |
| CPU | 8 cores (16+ recommended) | The first full build compiles the cross toolchain, kernel, Kodi and every package. |
| RAM | 16 GB | Kodi, the kernel and Qt link with LTO. |
| Free disk | 150 GB | Build tree, `sources/` cache and ccache for one device. Add 50 GB for Qt and a second build tree. |
| OS | Ubuntu 24.04 host, or any host with Docker | `tools/docker` has Dockerfiles for Ubuntu 22.04 to 26.04 and Debian 12 and 13. |

The sizes are estimates from comparable CoreELEC builds, not measurements on
this fork. Record real numbers from the first build and update this table.

Install Docker and git, then clone the build system and build the container
once:

```sh
git clone https://github.com/silo-theater/CoreELEC.git
cd CoreELEC
git checkout coreelec-22
docker build --pull -t silo-build tools/docker/noble
git remote add upstream https://github.com/CoreELEC/CoreELEC.git
```

Clone the other repos only if you will change them. The build fetches
`linux-amlogic`, `media_modules-aml` and `xbmc` as tarballs by commit hash.
Use full clones (no `--depth`), because bumping a pin needs history.

For `silo-shell` work on a desktop, install Qt 6.8 or newer, CMake 3.21+,
Ninja and the FFmpeg development packages. Desktop builds are for UI and
backend work only; the Amlogic player code runs only on the box.

## 2. Build and boot the image

One command builds a flashable image for every `Amlogic-no` box, including
the AM6B+:

```sh
docker run --rm -v "$(pwd)":/build -w /build -it \
  -e PROJECT=Amlogic-ce -e DEVICE=Amlogic-no -e ARCH=aarch64 \
  silo-build make image
```

- **Output** lands in `target/`: an `.img.gz` to flash, plus a `.tar` and
  `.tar.sha256` used for updates. The name comes from `DISTRONAME`,
  `OS_VERSION`, `OS_CODENAME` and `OS_STAGE` in `distributions/CoreELEC/`
  and `scripts/image`.
- **Build tree** is `build.CoreELEC-Amlogic-no.aarch64-22.0-devel/`.
  Downloaded sources go to `sources/`; keep that directory between builds
  and in CI.
- **One package:** `scripts/clean <pkg>` then `scripts/build <pkg>`, with
  the same `PROJECT`, `DEVICE` and `ARCH`. `make image` afterwards repacks
  only what changed.
- **Build types:** unset gives `devel` with a timestamp;
  `BUILD_PERIODIC=nightly` gives `nightly_<date>`; `OFFICIAL=yes` takes
  `OS_STAGE` and `BUILD_NUM` from the version file. CI uses nightly,
  releases use official.

To boot it on the AM6B+:

1. Write the `.img.gz` to an SD card or USB stick (balenaEtcher, or
   `zcat <image>.img.gz | dd of=<device>`).
2. On the card's `COREELEC` partition, copy
   `device_trees/g12b_s922x_ugoos_am6b.dtb` to `dtb.img`.
3. Boot from the card. Install to eMMC with `ceemmc` only once the image is
   trusted.
4. Enable SSH: set `coreelec='ssh'` in `config.ini` on the `COREELEC`
   partition, or use Kodi's settings. Log in as `root` with the password
   from `ROOT_PASSWORD` (`coreelec` today).
5. Debug without a serial console. Add `textmode` and `nosplash` to the
   `coreelec=` line in `config.ini` to print boot messages on the HDMI
   screen, and `debugging` for verbose logs. Once the box is up, read
   `dmesg` and `journalctl -b` over SSH. If a build does not boot, fix the
   `COREELEC` partition from a PC, or boot a second card holding the last
   good image. A UART adapter is not required.

The gate for this step is milestone 1's first task: stock CoreELEC 22, built
from our forks, boots to Kodi on the AM6B+ and plays a 4K HDR file with
TrueHD passthrough.

## 3. Work on the kernel, decoder modules and Kodi

The three forks are pinned by commit, so a change reaches the image only
after a pin bump.

| Package | package.mk | Fork branch |
| --- | --- | --- |
| `linux` | `projects/Amlogic-ce/packages/linux/package.mk` | `5.15.196_20260306` |
| `media_modules-aml` | `projects/Amlogic-ce/packages/linux-drivers/amlogic/media_modules-aml/package.mk` | `5.15.196_20260225` |
| `kodi` | `projects/Amlogic-ce/packages/mediacenter/kodi/package.mk` | `aml-5.15.196-22.0` |

To iterate locally without pushing every try:

1. Run `scripts/unpack <pkg>`, edit the tree in
   `build.<...>/build/<pkg>-<hash>/`, then `scripts/build <pkg>` and
   `make image`. Turn the edit into a commit in the fork once it works;
   `scripts/clean <pkg>` throws the unpacked tree away.
2. Kernel only: copy `target/*.kernel` to `kernel.img` on the `COREELEC`
   partition, or install the full `.tar` through `/storage/.update/`.
3. Decoder modules ship inside the system image, so a module change needs
   the `.tar` update.

To bump a pin, in a commit of its own:

1. Push the change to the fork branch.
2. Set `PKG_VERSION` to the new full hash. Fill in `PKG_SHA256` from the
   downloaded tarball before the first release.
3. Commit as `<package>: bump package to <short hash>` and rebuild.

Rules from [AGENTS.md](../AGENTS.md) that apply here:

- Kernel, audio and display changes need device evidence in the PR: what
  was played, the HDMI mode and the AVR reading.
- Forward-port p3i kernel and distro patches one topic per commit, crediting
  the original author. p3i Kodi patches are reference only.
- Do not change libcec's `CEC_FRAMEWORK_SUPPORT="no"` or the kernel's
  `CONFIG_CEC_MESON_G12A_AO=y` until the CEC choice is tested on the AM6B+
  with a TV and an AVR.
- Keep Kodi building; it is the fallback and the reference player.
- Merge `coreelec-22` from upstream with a merge commit, never a rebase.

## 4. Build silo-shell and Qt for the box

Neither Qt for EGLFS nor a `silo-shell` package exists yet, so this step is
new packages in this repository plus a CMake project in silo-theater.

### Qt 6 packages

The only Qt in the tree is `packages/addons/addon-depends/qt6` (Qt 6.11.0),
a static build for add-ons with EGL, EGLFS, OpenGL ES and Qt Quick switched
off on the target. Do not edit it. Add new packages instead, for example
under `packages/silo/`:

- `qtbase` with EGLFS, OpenGL ES 2 and 3, evdev, libudev, fontconfig and
  harfbuzz on, as shared libraries, built against `opengl-meson` (Mali
  fbdev EGL). EGLFS uses the `eglfs_mali` integration.
- `qtshadertools` and `qtdeclarative`, each with a host build for the tools.
- Other modules, such as `qtsvg`, only when the UI needs them.

The gate is milestone 1's Qt spike: a QML test app holding 60 fps on
`/dev/fb0` while hardware video plays on the plane underneath.

### silo-shell repository layout

Set up silo-theater as one CMake project (CMake 3.21+, C++20, Qt 6):

| Directory | Contents |
| --- | --- |
| `src/ui/` | QML and the C++ models behind it |
| `src/player/` | Demux (FFmpeg), clock, A/V sync, subtitles (libass) |
| `libsilo-aml/` | Code extracted from `silo-theater/xbmc` `aml-5.15.196-22.0`, kept close to upstream |
| `src/backends/` | `MediaBackend` with Silo and, later, Jellyfin |
| `src/services/` | BlueZ (D-Bus) and libcec |
| `tools/silo-play/` | Headless player CLI for the player spike and test suite |
| `tests/` | Unit tests, including the silo-server contract fixtures |
| `packaging/` | systemd unit and default config |

Add a CMake option such as `SILO_AML=OFF` so the UI and backends build on a
desktop with a stub video sink.

### silo-shell package

Add `packages/silo/silo-shell/package.mk`. It pins silo-theater by hash
like the forks, uses `PKG_TOOLCHAIN="cmake"`, and depends on
`qtbase qtdeclarative ffmpeg libass libamcodec alsa-lib libcec bluez dbus`.
The build system passes the cross toolchain file, `${CMAKE_CONF}`.

Ship a `silo-shell.service` that conflicts with `kodi.service`. Kodi stays
installed and only one frontend is enabled. Milestone 2's gate is the image
booting into `silo-shell` with Kodi startable as a fallback.

### Inner loop on the box

`/usr` is a read-only squashfs, so test builds run from `/storage`:

```sh
scripts/build silo-shell
scp build.CoreELEC-Amlogic-no.aarch64-22.0-devel/install_pkg/silo-shell-*/usr/bin/silo-shell \
  root@am6b:/storage/bin/
ssh root@am6b 'systemctl stop kodi && \
  QT_QPA_PLATFORM=eglfs QT_QPA_EGLFS_INTEGRATION=eglfs_mali /storage/bin/silo-shell'
```

Until Qt is in the image, also copy its libraries to `/storage/lib` and set
`LD_LIBRARY_PATH` and `QT_PLUGIN_PATH`. Build with `DEBUG=silo-shell` to
run under `gdbserver`.

## 5. Set up the hardware test bench

Every playback, kernel, audio or display change needs evidence from the
AM6B+, so the bench is part of the development environment.

| Item | Used for |
| --- | --- |
| Two or more Ugoos AM6B+ (one for daily work, one on the last release) | Reference box |
| Spare SD card or USB stick with the last good image | Recovery when a test build does not boot |
| AVR that shows the incoming format (Atmos, DTS:X, sample rate) | Proof of bit-perfect passthrough |
| TV with HDR10, HLG and Dolby Vision | Mode switching and HDR checks |
| HDMI analyser (optional) | Exact HDMI mode, InfoFrames and audio channel status |
| The TV and AVR above, with CEC on | CEC matrix: power, input switch, volume, remote keys |
| A Bluetooth remote, a game controller, headphones | Milestone 5, in that priority |
| Silo server with the reference media, plus a Jellyfin server | Backend tests |

Reference media: TrueHD Atmos, DTS-HD MA, DTS:X, DD+, LPCM 7.1; 23.976, 24,
25, 50 and 59.94 Hz; HDR10, HLG and Dolby Vision profiles 5, 7 and 8. Keep
the set on the Silo server and on a local disk.

Each hardware PR records the file played, the HDMI mode, the AVR or
analyser reading for passthrough formats, and, for display changes, the
frame rates and HDR formats checked. When a result looks wrong, play the
same file in Kodi on the same box.

## 6. Set up CI

None of our repos builds anything in CI yet. Image builds need a
self-hosted runner: a standard GitHub-hosted runner has about 14 GB of free
disk and a 6-hour job limit, far short of a cold CoreELEC build.
`silo-shell` checks fit on hosted runners.

| Repo | Trigger | Job | Runner |
| --- | --- | --- | --- |
| silo-theater | Push and PR | Desktop build with `SILO_AML=OFF`, unit and contract-fixture tests, clang-format | GitHub-hosted |
| silo-theater | Push and PR | Cross build of `silo-shell` against a cached CoreELEC toolchain | GitHub-hosted or self-hosted |
| CoreELEC | PR | `make image` for `Amlogic-ce` / `Amlogic-no`; ShellCheck on changed scripts | Self-hosted |
| CoreELEC | Nightly | `BUILD_PERIODIC=nightly make image`, publish to the nightly channel | Self-hosted |
| CoreELEC | Weekly | Build the other CoreELEC 22 devices so they keep building | Self-hosted |
| Forks | PR | Kernel and module build via the CoreELEC job with the PR's hash pinned | Self-hosted |

Runner setup:

- One x86-64 machine with the specs from step 1, Docker and the
  `silo-build` image, registered to the silo-theater organization.
- Persist `sources/` and ccache (`CCACHE_DIR`) between runs, and record how
  long a warm build takes.
- Run jobs in a fresh container but keep those two caches on the host.
- Keep signing keys off this runner.

A hardware-in-the-loop job (an AM6B+ that installs the nightly `.tar` and
runs `silo-play` against the reference files) can follow later. It is not
needed for the first release.

## 7. Make it shippable

A box running today's image identifies as CoreELEC, would update itself from
CoreELEC's servers, and nothing in the update path is signed. The pieces
below must exist before a public beta.

### Branding and versions

- Add `distributions/Silo-theater/` (`options`, `version`, splash, logo) and
  build with `DISTRO=Silo-theater`. `config/options` already reads
  `DISTRO`, so CoreELEC's own files stay untouched.
- Set `DISTRONAME`, `DESCRIPTION`, the SSH greeting, a root password policy
  (or force a password at first boot) and `MEDIACENTER` there.
- Give Silo-theater its own version, for example `OS_MAJOR=0`,
  `OS_MINOR=1`, `OS_STAGE=beta`, and record the CoreELEC base in
  `/etc/os-release`.

### Signing

The initramfs update code checks only the `.md5` files inside the `.tar`,
which guards against corruption, not tampering. Add signatures:

1. Create an Ed25519 key with minisign or signify. Keep the private key
   offline or in a GitHub environment secret that needs manual approval.
2. The release job signs each `.tar` and `.img.gz` and publishes the
   `.sha256` and signature next to them.
3. Bake the public key into the image. The on-box updater refuses any
   `.tar` whose signature does not verify before it writes to
   `/storage/.update/`.

The AM6B+ boots CoreELEC from the Ugoos bootloader through
`aml_autoscript`, so no bootloader signing is involved.

### Update channel

Today's update paths point at CoreELEC: the Kodi settings add-on
(`service.coreelec.settings`) checks CoreELEC's update server, and
`update-ce nightly` reads `relkai.coreelec.org`. The add-on is fetched at
build time, so confirm its URL when you change it. For Silo-theater images:

- Publish a JSON manifest per channel (`stable`, `beta`, `nightly`) listing,
  per device, the version, image URL, size, SHA-256 and signature URL.
  GitHub Pages can serve the manifest and GitHub Releases the files.
- `silo-shell` reads its channel's manifest, downloads, verifies the
  signature, places the `.tar` in `/storage/.update/` and reboots. The
  existing initramfs code applies it, keeps the `canupdate.sh` check and
  recovers from a failed update.
- Add a `projects/Amlogic-ce` override of `update-ce` that reads the same
  manifest, for SSH updates while the UI is unfinished.
- Disable the CoreELEC add-on's update check in Silo-theater builds.

### Release process

1. Bump the fork pins and the `silo-shell` hash, each in its own commit, and
   tag this repository `silo-<version>`.
2. Build with `DISTRO=Silo-theater OFFICIAL=yes` on the self-hosted runner.
   Record every fork's commit in the release notes.
3. Run the hardware matrix from step 5 on the release candidate.
4. Sign in the protected release job, upload to GitHub Releases, and update
   the channel manifest last.
5. Publish the matching source (`make src-pkg` and the tagged commits) to
   meet the GPL.

## Checklist to a first release

Items within a group can run in parallel.

**Environment (milestone 1)**

- [ ] Decide the silo-shell license and fix the README
- [ ] Fix the README's AM6B+ device tree name and tick the pinning task
- [ ] Build host with Docker and the `silo-build` image; record build time
      and disk use
- [ ] Build stock CoreELEC 22 from our forks and boot it on the AM6B+
- [ ] Self-hosted runner with cached `sources/` and ccache; PR and nightly
      image jobs
- [ ] Test bench and reference media set

**Spikes (milestone 1)**

- [ ] Qt packages for EGLFS on `opengl-meson`; QML at 60 fps over video
- [ ] `silo-play`: 4K HEVC HDR with TrueHD passthrough via `libsilo-aml`
- [ ] Silo device login and a v3 start returning `original_http` with
      passthrough for a TrueHD file
- [ ] CMake layout and desktop CI for silo-theater

**Bootable shell (milestone 2)**

- [ ] `silo-shell` package and systemd unit, with Kodi still startable
- [ ] `distributions/Silo-theater` with its own name, version and
      `MEDIACENTER`

**Ship path (before the milestone 6 beta)**

- [ ] Signing key, protected release job and published public key
- [ ] Channel manifests, the on-box updater and the `update-ce` override
- [ ] CoreELEC add-on update check disabled
- [ ] `PKG_SHA256` filled in for the fork pins
- [ ] Bit-perfect suite, display, CEC and Bluetooth matrices pass on the
      release candidate
- [ ] Release notes, signed images and source release, then the manifest
