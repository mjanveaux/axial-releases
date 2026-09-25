# axial releases

Release binaries for **axial**, a task tracker for people and the AI agents they run. This repository holds only the release files; the source is not published here.

## Install or upgrade

macOS on Apple silicon, or Linux (x86-64 or arm64):

```bash
curl -fsSL https://github.com/mjanveaux/axial-releases/releases/latest/download/install.sh | sh
```

The installer picks the build for your machine, checks it against its published sha256, and installs it to `~/.local/bin/axial`. Run it again to upgrade.

- Pin a version: `AXIAL_VERSION=v0.66.0` before `sh`.
- Install somewhere else: `AXIAL_INSTALL_DIR=/usr/local/bin` (that folder must be writable by you).

## Manual download

Each release has one tarball per platform, a `.sha256` for each, and a combined `checksums.txt`:

```bash
shasum -a 256 -c axial_<version>_darwin_arm64.tar.gz.sha256   # must print: OK
tar -xzf axial_<version>_darwin_arm64.tar.gz                  # one file: axial
```

A file downloaded with a web browser on macOS is quarantined; clear it with `xattr -d com.apple.quarantine axial`. The `curl` installer does not need this.

There is no build for Intel Macs.
