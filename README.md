# chromium-headless-shell (Linux amd64 / arm64)

Self-contained chrome-headless-shell for Linux: the browser binary plus the
system libraries it needs, so it runs on a clean host with no root and nothing
installed.

## What this solves

Chromium requires system libraries (libnss3, libnspr4, libexpat, etc.) that
live in `/usr/lib` and can only be installed with root. This repository builds
archives that bundle those libraries alongside the browser binary so they load
from a user-writable directory via `LD_LIBRARY_PATH` — no sudo, no package
manager, no system modification.

This repository provides both `amd64` and `arm64` from
[chromedp/headless-shell](https://github.com/chromedp/headless-shell), which
builds Chromium from source for headless use on both architectures. Google's
Chrome for Testing ships a `linux-arm64` chrome-headless-shell too (since
version 153), but that binary needs 28 system libraries directly and 40 with
their dependencies — X11, GLib, ATK, D-Bus, systemd, udev, SELinux, GBM,
ALSA — while chromedp's needs four (NSS, NSPR, expat, libgcc_s). Fewer
bundled libraries means fewer ways to break on a host.

## What ships

Each release publishes **two archives per architecture**:

| Archive | Contents |
|---------|----------|
| `chromium-headless-shell-linux-<arch>.tar.gz` | Browser binary + GL/Vulkan sidecars + resource paks + locales + Chromium's `LICENSE` |
| `chromium-headless-shell-libs-linux-<arch>.tar.gz` | System libraries (NSS/NSPR/expat/libgcc_s) + their copyright notices in `LICENSES/` |

Architectures: `amd64`, `arm64`.

## Platforms

- **Linux (glibc 2.31+)** — libraries are extracted on Ubuntu 20.04
  (glibc 2.31). Runs on any host with glibc 2.31 or newer: Ubuntu 20.04+,
  Debian 11+, Fedora 34+, RHEL 8+, etc.
- **musl (Alpine)** — not covered. glibc-linked libraries do not load on musl.
- **Windows / macOS** — not needed. Both ship self-contained browser bundles
  natively.

## How it works

1. The browser binary is extracted from the
   [chromedp/headless-shell](https://hub.docker.com/r/chromedp/headless-shell)
   Docker image (multi-arch, pushed daily from Chromium source).

2. System library dependencies are determined by running `ldd` on the binary
   inside a clean Ubuntu 20.04 container. The closed set of non-floor libraries
   (everything except libc, libm, libpthread, libdl, and the dynamic linker)
   is copied out.

3. A launch guard verifies the binary starts with the bundled libraries on
   four clean container images (Ubuntu 20.04, Ubuntu 24.04, Debian 12,
   Fedora 41) with nothing installed. If the binary fails to launch, the
   build fails loudly.

4. Both archives are published as GitHub Release assets.

## Usage

Both archives are required: the browser binary will not start without the
bundled system libraries.

### arm64

```bash
wget https://github.com/gabion-dev/chromium-headless-shell/releases/latest/download/chromium-headless-shell-linux-arm64.tar.gz
wget https://github.com/gabion-dev/chromium-headless-shell/releases/latest/download/chromium-headless-shell-libs-linux-arm64.tar.gz

tar -xzf chromium-headless-shell-linux-arm64.tar.gz -C /path/to/browser
tar -xzf chromium-headless-shell-libs-linux-arm64.tar.gz -C /path/to/libs
```

### amd64

```bash
wget https://github.com/gabion-dev/chromium-headless-shell/releases/latest/download/chromium-headless-shell-linux-amd64.tar.gz
wget https://github.com/gabion-dev/chromium-headless-shell/releases/latest/download/chromium-headless-shell-libs-linux-amd64.tar.gz

tar -xzf chromium-headless-shell-linux-amd64.tar.gz -C /path/to/browser
tar -xzf chromium-headless-shell-libs-linux-amd64.tar.gz -C /path/to/libs
```

### Launch

```bash
LD_LIBRARY_PATH=/path/to/libs /path/to/browser/headless-shell \
  --no-sandbox --headless --disable-gpu --version
```

Works with Playwright via `executablePath`:

```javascript
const browser = await chromium.launch({
  executablePath: '/path/to/browser/headless-shell',
  headless: true,
});
```

The `LD_LIBRARY_PATH` must be set on the process that spawns the browser
(it is inherited by the Chromium child processes).

## Build details

- **Browser source:** chromedp/headless-shell Docker image (both arches).
- **Library floor:** Ubuntu 20.04 / glibc 2.31. Libraries from chromedp's own
  Debian 13 base are not used — their glibc 2.41 would restrict the host to
  Ubuntu 25.04+.
- **Guard:** each binary is tested with `--version` under `timeout 30` on four
  clean distro images. A binary that fails to start is never shipped.
- **arm64:** built on a native `ubuntu-24.04-arm` runner. QEMU-emulated arm64
  Chromium segfaults, so the guard requires real silicon.

## License

The build scripts of this repository are MIT ([LICENSE](LICENSE)). The browser
binary is Chromium, BSD-3-Clause: its license text travels as `LICENSE` inside
each browser archive, taken from the Chromium source tree at the exact version
the binary reports. Bundled system libraries carry their upstream licenses;
their notices are included in `LICENSES/` inside each libs archive.
