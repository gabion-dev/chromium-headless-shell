# chromium-headless-shell

Self-contained chrome-headless-shell for Linux: the browser binary plus the
system libraries it needs, so it runs on a clean host with no root and nothing
installed.

## What this solves

Chromium requires system libraries (libnss3, libnspr4, libexpat, etc.) that
live in `/usr/lib` and can only be installed with root. This repository builds
an archive that bundles those libraries alongside the browser binary so they
load from a user-writable directory via `LD_LIBRARY_PATH` — no sudo, no
package manager, no system modification.

Google's Chrome for Testing ships no `linux-arm64` chrome-headless-shell build.
This repository covers **both** `amd64` and `arm64` from a single source
([chromedp/headless-shell](https://github.com/chromedp/headless-shell)).

## What ships

Each release publishes **two archives per architecture**:

| Archive | Contents |
|---------|----------|
| `chromium-headless-shell-linux-<arch>.tar.gz` | Browser binary + GL/Vulkan sidecars + resource paks + locales |
| `chromium-headless-shell-libs-linux-<arch>.tar.gz` | System libraries (NSS/NSPR/expat/libgcc_s) + license notices |

Architectures: `amd64`, `arm64`.

## Platforms

- **Linux** — the only platform that needs this. Windows and macOS ship
  self-contained browser bundles natively.
- **glibc 2.31+** — libraries are extracted on Ubuntu 20.04 (glibc 2.31).
  Runs on any host with glibc 2.31 or newer: Ubuntu 20.04+, Debian 11+,
  Fedora 34+, RHEL 8+, etc.
- **musl (Alpine)** — not covered. glibc-linked libraries do not load on musl.

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

```bash
# Download and extract both archives
tar -xzf chromium-headless-shell-linux-amd64.tar.gz -C /path/to/browser
tar -xzf chromium-headless-shell-libs-linux-amd64.tar.gz -C /path/to/libs

# Launch with bundled libraries
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
  Debian 13 base are NOT used — their glibc 2.41 would restrict the host to
  Ubuntu 25.04+.
- **Guard:** each binary is tested with `--version` under `timeout 30` on four
  clean distro images. A binary that fails to start is never shipped.
- **arm64:** built on a native `ubuntu-24.04-arm` runner. QEMU-emulated arm64
  Chromium segfaults, so the guard is only trustworthy on real silicon.

## License

MIT. Bundled system libraries carry their upstream licenses; notices are
included in `LICENSES/` inside each libs archive.